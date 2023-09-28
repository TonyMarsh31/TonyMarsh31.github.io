---
layout: post
title: "当坏事发生时，SpringCloud的弹性模式与Resilience4j的使用"
date: 2023-06-17 02:06 +0800
category: [ 读书笔记, "Spring微服务实战 第二版" ]
tag: [ Spring, SpringCloud, MicroServices, 微服务, 服务发现与注册 ]
---

本博文是原书第七章的读书笔记，主要内容包括

- circuit breakers, fallbacks, and bulkheads 的实现
- 使用circuit breaker 模式来节约客户端资源
- 当远程服务不可用时，使用Resilience4j
- 实现Resilience4j的bulkhead模式来隔离远程资源调用
- 微调Resilience4j的circuit breaker与bulkhead
- 客制化Resilience4j的并发策略

所有系统，尤其是分布式的系统，都会经历故障。如何建立一个可以对故障做出反应的系统，是一个很重要的问题。
然而当考虑到如何构建一个有弹性的系统时，大多数的软件工程师只对 基础设施与关键的系统组件 进行考量。
他们关注于在 应用程序的每一层上 用集群化、负载均衡与将基础设施隔离到多个地方来构建冗余。

尽管该方法考量到了系统组件的完全(通常是大量的)失败所造成的潜在损失，但这些只是构建一个弹性系统的一小部分。
当一个服务奔溃时，很容易侦测到并且绕过它。但是，当一个服务变得缓慢时，侦测该服务并试图绕过是极为困难的。以下是一部分原因

- 服务退化的开始阶段可以是间歇性的，然后逐渐积攒动量直到瞬间崩溃  
  一个服务的退化可能一开始只是一个小范围的延迟，直到该服务达到一个临界点(线程池耗尽)，然后突然崩溃。
- 服务的调用方通常是同步的，且不设定超时  
  软件开发者通常调用一个远程服务执行一些操作然后等待响应。这一调用通常没有设定超时。
- 应用程序通常仅被设计为处理完全故障，而不是部分退化  
  通常，只要服务没有down，应用程序就会继续调用它(即使服务已经是表现不佳的)。这种情况下，不会有快速失败的反馈。
  应用程序可以进行优雅降级，但更可能的是，对导致由Resource exhaustion(资源耗尽)引起的完全故障。
  资源耗尽指的是，当一个有限的资源，例如线程池或数据库连接池，被消耗殆尽时，新的调用将阻塞直到资源可用。

由性能不佳的远程服务引起的问题的阴险之处在于它们不仅难以被发现，而且会引发一连串的影响，可能会波及到整个应用生态系统。
如果没有保障措施，一个单一的、糟糕的服务就会迅速导致多个应用程序瘫痪。
云原生的微服务应用程序特别容易受到这些类型的故障影响,因为这些应用是由大量细粒度的分布式服务组成的，
而这些服务有不同的基础设施参与完成用户的操作。

所以我们接下来要讨论的 Resiliency patterns弹性模式是微服务架构中最关键的方面之一。
本章将解释四种弹性模式以及如何使用Spring Cloud和Resilience4j实现这些模式，从而使程序能够在有需要的时候快速失败

## What are client-side resiliency patterns?

客户端一侧的弹性模式聚焦于保护(消费其他远程资源的)客户端，避免因为 远程服务的失败或退化 而导致的故障。  
即当一个客户端消费另一个微服务或者一次数据库查询的时候，若这个服务失败或性能极差，弹性模式将允许客户端进行快速失败
来避免继续消耗有限的资源(例如数据库连接或线程池)。 同时其也防止了差表现的微服务 影响到其他的服务，拉慢整个应用的速度。

下图中展示了我们将要讨论的四种弹性模式与其在 微服务架构中的位置。

![image](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/image.377imjly49a0.webp)

该弹性模式(包含了客户端负载均衡，断路器，fallback与bulkhead)在(微服务组件)客户端实现。
其在逻辑上位于 客户端消费其他微服务 与 微服务或远程资源之间。

让我们具体看看每一个弹性模式的作用。

### Client-side load balancing

我们在上一章(上一博文)谈论服务发现时介绍了客户端的负载均衡，其包括了客户端查询所有可用的服务实例，然后将其物理地址信息缓存并自己
通过负载均衡算法来选择一个服务实例。当一个服务的事例不可用或者性能不佳(超时),客户端的负载均衡组件将其从可用的服务列表中移除。
然后单独的，客户端负载均衡组件会时不时地与服务发现组件对数据进行同步。

这些内容都在上一博文中有了详尽的描述，所以这里不再进行赘述。

### Circuit breaker

断路器模式是一种保护机制，该术语继承自电气工程领域。
断路器是一种开关，当电路中的电流超过其额定值时，它会自动断开电路，以防止电路进一步过载烧坏下游的部件设备。

在软件领域中的断路器会监视对远程服务的调用，如果调用失败则进行快速失败并防止之后继续调用该服务。
如果调用时间过长，断路器会干预然后直接 kill the call。

### Fallback processing

使用Fallback模式，当一个对远程资源的调用失败是，其不会返回一个Exception而是尝试通过另外一种范式来执行操作。
Fallback就是作为一个备用方案(Alternative)，当一个服务不可用时，其会提供一个备用的响应。

具体设计一个Fallback是根据业务需求的，例如简单地尝试另一个服务，或是返回缓存数据，或是直接告知用户服务不可用等等。
或是一整段复杂的业务逻辑，例如在一个电子商务网站中，当由于缺少数据样本导致无法计算出个性化的推荐数据时，选择流行的商品推荐给用户。

### Bulkheads

Bulkheads(舱壁)是一个基于船舶的概念，一艘船的载货区中有多个隔离的舱室，其是完全隔离与水密的，这样当一个舱室被破坏时，其它舱室仍然可以保持浮力。

![image](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/image.42soex67kpq0.webp)

同样的理念也被应用到了软件领域。
当一个服务需要与多个远程资源进行交互时，利用Bulkheads模式可以将对每个远程资源的调用隔离开来，
从而防止一个远程资源的故障影响到其他的远程资源调用或是整个应用程序。

具体来说，Bulkheads通常表现为一个线程池，每个线程池都有自己的资源，例如数据库连接或是远程服务的连接。
而每个线程池都是独立的,每一个远程调用也使用单独的线程池，这样当一个线程池中的资源耗尽时，其不会影响到其他线程池的资源。

## Why client resiliency matters

我们已经对客户端的弹性模式有了一个大致的了解，接下来我们结合一些传统的情形与例子来解释为什么客户端的弹性模式是必要的。

![image](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/image.2oc0mf9sogm0.webp)

如果不使用任何我们之前所提到的弹性模式，那么当图上库存服务的NAS出现故障或表现不佳时，由于写organization服务的代码将
读读取库存的代码与自己写入组织数据库的操作组合成了一个事务，所以当库存服务一直没有完成操作的时候，自己对于组织数据库的写入操作也会一直被阻塞。
同时该数据库连接将会一直被占用，同时许可证服务由于等待组织服务的响应而被阻塞，并慢慢地耗尽了资源。

其实上述所有的问题，只要在每一个分布式的系统中实现一个断路器模型就可以解决。下面是一个简单的图例

![image](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/image.6rm7ea61x7o0.webp)

在上图中，如果对于库存服务的调用是用一个断路器来实现的，那么当库存服务表现不佳时，该断路器会跳闸进行快速失败，不让调用继续进行来
持续地占用资源。同时保证了错误不在传递，其他不依赖于库存服务的微服务不会受到影响。

断路器是应用程序和用户之间的一个中间人，其保护用户在服务出现故障时不会完全崩溃。
同时在形式上，断路器也是一个proxy，当进行服务调用时，消费方不是直接call服务，而是通过断路器来进行调用。
这样断路器可以对请求进行监控并在需要时进行干预。

图中具体有三种不同的状态。第一个为最优状态，即服务正常，没有错误发生。
断路器做的事情仅仅只是记录请求的时间，由于服务正常，响应在阈值时间内返回，所以断路器不会干预请求。

在第二种情况下，断路器记录请求时间超过了阈值，判断服务出现了部分退化，此时其会跳闸并返回一个错误作为快速失败，来防止请求继续占用资源。

第三种情况就是第二种情况的拓展，其组合了Fallback.  
现在当服务部分退化后不直接进行快速失败，而是使用Fallback模式来返回一个备用的响应。
这允许 原先的组织服务有一点喘息的空间，来防止其被过多的请求占用资源最终导致完全崩溃。  
然后在断路器定期地尝试对服务进行健康检查 或直接进行访问后，若服务恢复正常，那么断路器会自动恢复，这样就可以继续使用服务了。(
断路器具体的恢复机制在后面会进行详细阐述)

总之断路器带来了以下好处：

- 快速失败：当服务出现了部分退化时，断路器会快速失败，不会让请求继续占用资源。 在多数情况下，快速失败总是好于 等待超时后的完全失败
- 优雅降级：除了快速失败，当有被提供的备用响应(Fallback)时，断路器可以选择使用备用响应来代替完全失败
- 无缝恢复：断路器充当着中间人的角色，其会时不时地再次尝试对服务进行访问，如果服务恢复正常，那么断路器会自动恢复，这样就可以继续使用服务了

在大型的云原生应用中，这种优雅地恢复是非常重要的，因为在一个大型的应用中，服务的故障是不可避免的，而这中做法可以
显著地减少恢复一个服务所需要的时间。 同时也大大减低了 人为失误 导致的故障造成更大的影响。。

在Resilience4j之前，Netflix的Hystrix是一个非常流行的断路器实现，但是由于其已经停止维护，所以我们不会在本书中使用它。

## Implementing Resilience4j

Resilience4j是一个受Hystrix启发的容错库。它具体提供了以下弹性模式的实现，以增加当网络问题或任意服务节点故障时的应用程序的容错性

- Circuit breaker - 当服务失败时，停止对其进行调用
- Retry - 当服务暂时地失败时，重试调用
- Bulkhead - 限制对远程服务的并发调用数量来避免资源耗尽
- Rate limiter - 限制对远程服务的调用速率
- Fallback - 当服务失败时，提供一个备用响应

具体构建对于这些不同弹性模式的实现需要对于线程与线程池有深入的了解，但有幸的是，通过使用
SpringBoot和Resilience4j库，我们可以很容易地实现这些弹性模式，而不需要关心具体的实现细节。

有了Resilience4j搭配SpringBoot，我们可以简单地通过在方法上添加注解的方式来为其添加多个弹性模式。
例如，我们想要限制对远程服务的并发调用数量，并添加一个断路器，那么我们可以使用`@Bulkhead`和`@CircuitBreaker`注解来实现。

但要注意的是，当同时标注多个弹性模式时，Resilience4j要求不同弹性模式的注解需要按照一定的顺序来进行添加,
具体顺序在代码上的表现为`Retry(CircuitBreaker(RateLimiter(TimeLimiter(Bulkhead(Function)))))`  
(可能有时效性，请具体参考使用版本对应的文档)

在之后的内容中，我们将会覆盖以下内容：

- 配置项目以使用Resilience4j
- 使用SpringBoot/Resilience4j注解在远程调用方法上包装不同弹性模式
- 客制化一个单独的断路器来实现对不同的调用配置不同的超时时间
- 在断路器不得不干预或者调用直接完全失败时，实现fallback策略
- 在服务中使用单独的线程池来实现bulkhead模式

## 配置项目以使用Resilience4j

Resilience4j库的导入相对来说复杂一点(在原书使用SpringBoot2的版本的时期)

简单来说需要引入

- `resilience4j-spring-boot2`
- 接着，每一个弹性模式都需要引入单独的依赖组件，例如`resilience4j-circuitbreaker`,`resilience4j-timelimiter`等
- `spring-boot-starter-aop`作为间接依赖需要引入

----

## Implementing a circuit breaker

我们想要断路器发挥的作用是监视远程服务的调用请求，来避免长时间的等待。
若这一情景出现，断路器负责切断这些请求，并检测是否有更多的失败或表现不佳的调用。

在Resilience4j中，断路器是通过一个有三种状态的状态机来实现的，图例如下：

![image](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/image.4ycur7dxf4c0.png)

最初，断路器是关闭的，这意味着所有的请求都会被转发到远程服务并等待响应。
然后当失败出现的速率超过了阈值时，断路器会打开，这意味着所有的请求都会被立即拒绝(断路器直接返回一个异常对象)，而不会被转发到远程服务。
接着，经过一段时间后(可自定义配置时间)，断路器会进入半开状态，允许一些请求会被转发到远程服务，以重新计算失败速率进而进一步决定断路器是否应该打开或关闭。

这是断路器的状态机模型与基本运行机制，不难发现其影响其机制的一个核心指标就是**失败速率**,那么其是如何计算的呢？

断路器使用ringBuffer(Ring bit buffer)来记录最近的请求的结果(成功与否)，以便在未来的某个时间点来决定是否打开断路器。
图例如下：

![image](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/image.53iup9l4b9s0.webp)

当请求的响应成功(按时)返回时，断路器会将0bit存入ringBuffer中，而当请求的响应失败(超时)时，断路器会将1bit存入ringBuffer中。
计算失败速率的基本条件是先将RingBuffer存满，然后就是简单地计算0/1比值然后与阈值比较。
一个需要注意的细节是，因为计算失败速率的必要条件是RingBuffer存满，所以这意味着即使目前为止所有的请求都是失败的，但若RingBuffer还没有存满，断路器也不会计算失败速率进而不会打开。

---

补充：在Resilience4j中，断路器有两个额外的状态：禁用和强制打开。

- DISABLED - 断路器不会打开，即使失败速率超过阈值
- FORCED_OPEN - 断路器会一直打开，即使失败速率低于阈值

>
这两个状态因为不在本书范围内作者没有做详尽的介绍，但补充了[官方文档](https://resilience4j.readme.io/v0.17.0/docs/circuitbreaker)
{:.prompt-info}

---

接着，具体在书中的案例中，作者将要分两类来实现断路器：

- 将所有对数据库的调用都用一个断路器来包装
- 将内部的许可证服务队组织微服务的调用用一个断路器来包装

尽管两者的种类不同(数据库连接请求，RESTAPI消费),但我们将会看到得益于SpringBoot/Resilience4j的注解，我们可以很容易地用完全相同的方式来实现这两种断路器。

![image](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/image.1un3bbs0nbls.webp)

让我们从使用同步断路器包装数据库调用开始。
默认情况下，服务调用数据库会是一个没有超时的同步调用，而通过使用断路器，我们可以将其转换为一个有超时的同步调用。
在操作上只需要使用`@CircuitBreaker`来标记有对应行为的方法即可。

```java
@CircuitBreaker(name = "licenseService")
public List<License> getLicensesByOrganization(String organizationID) {
    //……
}
```

代码不是很对，但该注解做了非常多的工作。
当Spring框架检测到一个方法被`@CircuitBreaker`注解标记时，它会自动创建一个断路器并将其绑定到这个方法上。
该断路器作为一个Proxy代理包装了该方法,并通过一个专门用于处理远程调用的程池接管对该方法的所有调用。

具体来说，`getLicensesByOrganization()`在任意时刻被调用时，该调用都会被Resilience4j断路器包装，其会中断任何失败或超时的调用并直接返回一个异常对象。

接着作者进行了一些具体的测试，其定义了一些Random方法来使得该数据库调用有一定几率超时(通过Thread.sleep)
然后，具体调用该方法时，有一定几率会抛出`Internal Server Error`的异常，然后再进行多次尝试与部分失败后。
断路器通过RingBuffer记录的状态数量满了，进而计算失败速率，最终打开断路器，拒绝所有的请求，此时的异常就是`CircuitBreaker`的异常。
其信息一般会阐述中该service is open And does not permit further calls

---

接着作者在两个微服务组件之中进行断路器的配置，其与上述的数据库调用的断路器配置是一致的，只是其使用了不同的断路器名称。这里便不再赘述。

---

### Customizing the circuit breaker

这一部分同样详细的配置项可以在对应版本的[官方文档](https://resilience4j.readme.io/docs/circuitbreaker.)中找到，所以仅说明一些宏观的思路。

首先，每一个断路器的区分就是在注解上的`name`属性，其对应的就是断路器的名称，缺省情况下会使用方法名作为断路器的名称。
然后通过一下格式就可以在配置文件中为每一个不同的断路器进行配置：

```yaml
resilience4j.circuitbreaker:
  instance:
    name1:
      property1: value1
      property2: value2
    name2:
      property1: value1
      property2: value2
```

常用的配置项有

- 对RingBufferSize的配置。 注意，在Closed和HashOpen两个状态下使用的是不同的RingBufferSize，所以需要分别配置。
- waitDurationInOpenState. 即断路器开始后 多久再进入HalfOpen的状态允许部分请求通过来测试服务是否恢复。
- failureRateThreshold. 即失败速率的阈值，当失败速率超过该阈值时，断路器会打开。数值为百分数，50表示50%。
- recordExceptions. 列出哪些异常会被视作失败，从而计算失败速率。默认情况下，所有的异常都会被视作失败。

## Fallback processing

要使用Fallback，我们需要在`@CircuitBreaker`或其他弹性模式注解的***内部属性***中，指定对应的fallback方法.  
注意该方法的签名必须与原方法在一个类中，且方法签名一致并多一个`Throwable`类型的参数用于接收原方法抛出的异常。

```java
@CircuitBreaker(name = "licenseService", fallbackMethod = "buildFallbackLicenseList")
public List<License> getLicensesByOrganization(String organizationId) throws TimeoutException {
  //……
}

// Fallback method
private List<License> buildFallbackLicenseList(String organizationId, Throwable t){
  //……
}
```

具体在代码操作上，实施Fallback就只需要这些操作。

但在实际使用中，有以下两点需要注意：

- Fallback提供了一套在主业务逻辑失败时的备用逻辑。 而若是你使用Fallback要做的仅仅只是记录日志，那么一个更好的替代是直接使用传统的Try/Catch块。
- 时刻注意Fallback本身也有可能出错，即在主业务逻辑上出现的错误在Fallback可能还是会出现。所以推荐也要用其他的弹性模型，例如断路器包装Fallback方法进行防御性编程。

## Implementing the bulkhead pattern

在一个基于微服务的应用中，通常会调用多个远程微服务来完成一项任务。  
如果不使用bulkhead模式，那么缺省情况下，执行远程服务调用的线程将会是那些在Java容器中处理所有请求的线程。  
换句话说，对于远程服务的调用这一操作没有一个边界(作为隔离)，如果有阻塞，那么其最终可能会蔓延到整个应用中最终导致崩溃。

(例如在远程服务已经down了的情况下，新的线程又不断去执行该远程调用导致所有线程均阻塞后整个应用奔溃。
而使用Bulkhead后，对于单一远程服务的调用这一个工作可调用的资源量被限制，其不会再有能力占用所有的资源，而是仅能占用一个bulkhead中的资源)

这也就是为什么需要实施bulkhead模式的原因。
Resilience4j提供了 信号量 与 线程池 两种实现bulkhead模式的方式。默认情况下，Resilience4j使用信号量来实现bulkhead模式。

![image](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/image.m7xrwag9cnk.webp)

![image](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/image.1lf5sjgu1s8.webp)

---
要实施bulkhead模式，同样的只要在对应方法上添加`@Bulkhead`注解即可。
其命名规则与断路器相同，即`name`属性指定了bulkhead的名称，缺省情况下使用方法名作为bulkhead的名称。

然后对于每一个具体的bulkhead，我们需要在配置文件中进行配置，一个配置示例如下:

```yaml
resilience4j.bulkhead:
  instances:
  bulkheadLicenseService:
  maxWaitDuration: 10ms
  maxConcurrentCalls: 20

resilience4j.thread-pool-bulkhead:
  instances:
  bulkheadLicenseService:
  maxThreadPoolSize: 1
  coreThreadPoolSize: 1
  queueCapacity: 1
  keepAliveDuration: 20ms
```

由于bulkhead支持信号量与线程池两种模式的实现，所以在同一名称的bulkhead中可以同时配置信号量与线程池的配置项。

而具体在使用时，缺省使用的是信号量模式，若要使用线程池模式，需要在`@Bulkhead`注解中添加`type`属性，其值为`ThreadPool`。  
例如： `@Bulkhead(name = "bulkheadLicenseService", type = Bulkhead.Type.THREADPOOL)`

书中对于如何配置线程池参数有一些101的说明，这里仅补充一个(书中提供的)线程池最大线程数的参考公式：
> maxThreadPoolSize = (requests per second at peak when the service is healthy * 99th percentile latency in
> seconds) + small amount of extra threads for overhead

99th percentile latency in seconds 的意思是，99%的请求的响应时间都在这个时间内。  
即将所有的请求按照响应时间排序从小到大排序，取出排在99%位置的请求的响应时间，这个时间就是99th percentile latency in
seconds。

## Implementing the retry pattern

顾名思义，retry就是重试的意思。当初次调用失败时，retry弹性模式使得我们可以直接进行重试。

类似的，在使用上，只需要在对应方法上添加`@Retry`注解即可。
其命名规则与断路器相同，即`name`属性指定了retry的名称，缺省情况下使用方法名作为retry的名称。

每一个retry都需要在配置文件中进行配置，一个配置示例如下:

```yaml
resilience4j.retry:
  instances:
    retryLicenseService:
      maxRetryAttempts: 5
      waitDuration: 10000
      retry-exceptions:
        - java.util.concurrent.TimeoutException
```
- maxRetryAttempts: 最大重试次数  , 默认值为3
- waitDuration: 重试间隔时间，单位为毫秒, 默认值为500ms
- retry-exceptions: 重试的异常类型(为下述异常类型的子类时，才会进行重试),默认为空
 
书中具体使用到的配置就是这些，但也有如下一些常用配置：

- intervalFunction: 重试间隔时间的计算函数，其参数为当前重试次数，返回值为重试间隔时间，单位为毫秒.用于实现自定义的重试间隔时间计算逻辑
- retryOnResultPredicate: 重试的结果判断函数，其参数为当前重试次数与上一次调用的结果，返回值为是否进行重试.用于实现自定义的重试结果判断逻辑
- retryOnExceptionPredicate: 重试的异常判断函数，其参数为当前重试次数与上一次调用的异常，返回值为是否进行重试.用于实现自定义的重试异常判断逻辑
- ignoreExceptions: 忽略的异常类型(为下述异常类型的子类时，不会进行重试),默认为空


## Implementing the rate limiter pattern

## ThreadLocal and Resilience4j

