---
layout: post
title: "通过使用Spring Boot Actuator监控应用程序，并结合Spring Boot Admin以获得更出色的UI界面"
date: 2023-05-23 18:40 +0800
category: [ 读书笔记, "spring实战第六版" ]
tag: [ Spring, Spring Boot, Spring Boot Actuator, 服务监控 ]
---

Actuator是Spring Boot提供的一个监控和管理生产环境的模块，它提供了很多的端点(也支持自定义断点)，可以通过HTTP或者JMX的方式进行访问。
通过访问这些断点，我们可以获得应用的一些信息，比如应用的健康状况，应用的配置信息，应用的性能指标等等。

## Actuator介绍

在机械工程领域中，"Actuator"通常指代驱动器或执行器，用于转换输入能量为机械运动或执行某种操作。  
然而，在计算机软件领域，"Actuator"被引申为一种控制组件，用于监控和管理程序的行为和状态.

在 Spring Boot 应用程序中，Spring Boot Actuator 扮演同样的角色，使我们能够看到正在运行的应用程序的内部,并在一定程度上，控制应用程序的行为。例如

- 获取应用程序的健康状况
- 获取应用程序的配置信息(并对配置进行修改)
- 获取应用程序的性能指标(消耗的内存，CPU使用率等)
- 应用程序接受了哪些HTTP请求
- 应用程序的日志信息

要在应用程序中使用Actuator，只需要在pom.xml中添加如下依赖即可

```xml

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

只要导入依赖，Actuator就会自动启用，其提供的断点可以通过HTTP或者JMX的方式进行访问。  
JMX的部分不在本文的讨论范围内，本文主要讨论如何通过HTTP的方式访问Actuator的端点。

## 使用Actuator端点

### 配置baseurl

默认情况下，访问断电的URL是`/actuator/{endpoint}`，其中`{endpoint}`是端点的名称。  
而`actuator`本身作为baseurl是可以修改的，可以通过`management.endpoints.web.base-path`属性进行修改。

例如下述配置中，将`actuator`修改为`management`，那么访问断点的URL就是`/management/{endpoint}`

```yaml
management:
  endpoints:
    web:
      base-path: /management
```

之后所有的示例中,都将省略`/actuator`这部分，直接使用`/{endpoint}`来表示断点的URL。

### 启用与禁用端点

有些断点默认是启用的，有些断点默认是禁用的，可以通过`management.endpoint.web.exposure`属性陪着include和exclude来启用或者禁用断点。

例如下述配置中，启用了所有的断点，但是禁用了`threaddump`和`heapdump`这两个断点。

```yaml
management:
  endpoints:
    web:
      exposure:
        include: '*'
        exclude: threaddump,heapdump
```

访问/actuator断点，可以查看所有的可访问的端点信息。

本博文不会对所有的端点进行介绍，只介绍一些常用的端点。

> Actuator的端点返回的数据格式是JSON，其往往会很长并(在该博文语境中)没有特别有价值的信息，所以在本博文中，只会给出端点的URL，而不会给出端点的返回值。
{: .prompt-info}

### 获取应用程序的重要信息

`/health`端点用于获取应用程序的健康状况，它的返回值是一个JSON对象，包含了应用程序的健康状况信息。
健康状况信息有以下几种

- `UP`：应用程序正常运行
- `DOWN`：应用程序不正常运行
- `OUT_OF_SERVICE`：应用程序不可用
- `UNKNOWN`：应用程序的健康状况未知

在默认配置下，端点只返回一个status表示整个应用程序的健康状况，如果需要查看更详细的信息，可以通过`management.endpoint.health.show-details`
属性进行配置。
一个程序中的所有子组件均为up是，该程序才会被认为是up的。反之若有一个为down或out of service则该程序就会被认为是down/out of
service的。 UNKNOWN则是当程序无法确定自己的状态时返回的其不会影响程序的整体状态的判定。

`/info`端点用于获取应用程序的信息，它的返回值是一个JSON对象，包含了应用程序的信息。

需要注意的是，缺省情况下，该节点不返回任何信息，如果需要返回信息，一种方式是在配置文件中手动添加一些信息，例如

```yaml
info:
  contact:
    email: support@tacocloud.com
    phone: 822-625-6831
```

还有一种做法是自定义该端点的实现，这一部分的内容在后一节中进行介绍。

### 获取配置详细信息

除了获取有关应用程序的一般信息之外，了解应用程序的配置信息也很有用。应用程序上下文中有哪些
bean？哪些自动配置条件通过或失败？应用程序可以使用哪些环境属性？HTTP 请求如何映射到控制器？一个或多个包或类设置为怎样的日志级别？

Actuator提供了丰富的开箱即用的端点为我们提供了上述这些信息，以及更多。

- 通过 /beans 可以查看应用程序上下文中的所有bean的包括依赖关系的具体信息(这将会是一个非常长的列表)
- 通过 /conditions 可以查看所有的自动配置条件的信息 (
  SpringBoot自动配置为我们的项目做了什么，其行为的condition和最终的结果是什么)
- 通过 /env 可以查看应用程序的环境属性信息。 在之前的[Spring中的配置管理]({% post_url /读书笔记/Spring in Action, Sixth Edition/2023-05-02-working-with-configuration-properties-spring-in-action-6th-edition %})
  博文中，提到Spring的配置环境是多个配置信息源的集成抽象，其中包含了系统环境变量，系统属性，JNDI，命令行参数，属性文件，YAML文件，以及其他的一些配置信息源。Actuator的
  /env 端点可以查看到这些配置信息源中的所有配置信息。

> 你可以在端点名之后继续添加属性名，以获取特定的属性信息。例如，/env/spring.datasource.url
> 将会返回spring.datasource.url属性的具体信息。
{: .prompt-tip}

- 通过 /mappings 可以查看应用程序中的所有HTTP请求映射到控制器的具体信息,包括请求的方法，请求的路径，以及对应的控制器方法。
- 通过 /loggers 可以查看应用程序中的所有日志记录器的具体信息，包括日志记录器的名称，日志记录器的级别，以及日志记录器的有效级别。

> 类似的，可以在端点名之后添加包名、日志级别等信息，以获取更加精确的信息。例如，/loggers/org.springframework.web
> 将会返回org.springframework.web包下所有日志记录器的具体信息。
{: .prompt-tip}

### 查看应用程序活动

关注应用程序中的活动是非常有用的。这些活动包括应用程序正在处理的各种 HTTP 请求，以及应用程序中的所有线程等等。为此，Actuator
提供了 /httptrace、/threaddump 和 /heapdump 端点。

- /heapdump 是一个非常高级的端点，它会导出应用程序的堆转储文件。这个文件可以用于分析应用程序的内存使用情况，以及检查内存泄漏等问题。但这种分析是非常复杂的，这里不做进一步介绍。
- /httptrace 会返回应用处理的最近 100 个请求。详细信息包括请求的方法和路径、时间戳、请求和响应的头信息，以及处理请求的耗时。
- /threaddump 端点生成一个当前线程活动的快照。

### 利用运行时的指标信息

/metrics端点能够报告正在运行应用程序的多项指标，包括有关内存、处理器、垃圾收集以及 HTTP 请求的相关指标。
由于涉及的指标太多，不可能在本文中全部讨论。作为如何使用 /metrics 端点的示例，这里仅讨论 http.server.requests 指标。

通过 /metrics/http.server.requests 端点可以查看应用程序处理的所有 HTTP
请求的指标信息。这些信息包括请求的方法、路径、状态码、请求的总数、请求的总时间、请求的最大时间、请求的最小时间、请求的平均时间等等。

> 使用Actuator端口获取的信息是非常原始的，本博文最后会讲到一个更加友好的UI界面，可以更加方便地查看这些信息。所以这里不对如何获取精细的指标信息进行讨论。(
在操作上还是类似上面的方法，在url中添加更加精细的分类或标记信息即可)
{: .prompt-tip}

## 自定义Actuator端点

Actuator开箱自带了非常多 并且有用的端点。但是，如果您需要更多的端点，或者需要更多的信息，那么您可以自定义 Actuator 端点。
Actuator 的最大特点之一是可定制，以满足特定应用程序的需求。一些端点本身允许定制，同时，Actuator 本身允许您创建自定义端点。
让我们看一些定制 Actuator 的方法，首先是添加信息到 /info 端点。

### 定制/info端点 添加自定义信息

在配置文件中手动输入静态的数据不是为info端点添加信息的唯一方法。通过实现Actuator提供的InfoContributor接口，可以将动态的信息添加到/info端点中。

```java
@Component
public class TacoCountInfoContributor implements InfoContributor {
  @Autowired
  private TacoRepository tacoRepo;
  
  @Override
  public void contribute(Builder builder) {
    long tacoCount = tacoRepo.count();
    Map<String, Object> tacoMap = new HashMap<String, Object>();
    tacoMap.put("count", tacoCount);
    builder.withDetail("taco-stats", tacoMap);
  }
}
```

代码很直接，就不解释了

查询info端口的效果(前后部分略)：

```json
{
  "taco-stats": {
    "count": 44
  }
}
```

作者还提到了一点，若想要添加build信息(artifact Version group等信息),以及若是一个公开项目的话，想要添加git
commit的具体信息到info中，添加一些依赖即可， 这里就不赘述了，具体的依赖信息没有时效性，这里仅做补充。

### 定制/health端点 添加自定义健康指标

应用场景 : 一些组件(或没有遵循规范或太过老旧)没有提供健康指标，或者提供的健康指标不够精细，需要自定义健康指标。

实现HealthIndicator接口，重写health()方法即可，其需要返回Health对象，可以通过Health对象的静态方法创建，也可以通过Health.Builder创建，这里使用后者。

```java
@Component
public class WackoHealthIndicator implements HealthIndicator {
@Override
  public Health health() {
    int hour = Calendar.getInstance().get(Calendar.HOUR_OF_DAY);
    if (hour > 12) {
        return Health
            .outOfService()
            .withDetail("reason",
                "I'm out of service after lunchtime")
            .withDetail("hour", hour)
            .build();
    }
    if (Math.random() < 0.1) {
        return Health
            .down()
            .withDetail("reason", "I break 10% of the time")
            .build();
    }
    return Health
        .up()
        .withDetail("reason", "All is good!")
        .build();
  }
}
``` 

上述代码显然只是为了演示，其在中午之后，返回 OUT_OF_SERVICE 的健康状况，并提供一些详细信息来解释故障原因。
而在午前，健康指标也有 10% 的可能性会报告 DOWN 的状态，因为它使用一个随机数来决定它是否正常运行。如果随机数字小于
0.1，状态将报告为 DOWN 。否则，状态为 UP。

这个WackoHealthIndicator在真实环境中任何情况下都不会有用。但是想象一下，如果代码中不是用随机数，而是远程调用某个外部系统并确定其状态。在这种情况下，这将是一个非常有用的健康状况指示器。

### 定制/metrics端点 添加自定义指标
书中的案例为，添加一个`/metrics/tacocloud` 返回有多少玉米卷用不同的配料制作出来了

```java
@Component
public class TacoMetrics extends AbstractRepositoryEventListener<Taco> {
  private MeterRegistry meterRegistry;
  public TacoMetrics(MeterRegistry meterRegistry) {
    this.meterRegistry = meterRegistry;
  }

  @Override
  protected void onAfterCreate(Taco taco) {
    List<Ingredient> ingredients = taco.getIngredients();
    for (Ingredient ingredient : ingredients) {
      meterRegistry.counter("tacocloud", "ingredient", ingredient.getId()).increment();
    }
  }
}
```
做法是 通过Actuator提供的meterRegistry组件，注册与维护一个计数器，计数器的名字为tacocloud，标签为ingredient，标签的值为配料的id。

对其维护的行为是创建一个TacoRepository的时间监听器，当有新的Taco被创建时，遍历其配料，为每个配料在meterRegister中记录的数据做自增操作。

访问结果：
$ curl localhost:8087/actuator/metrics/tacocloud
```json
{
  "name": "tacocloud",
    "measurements": [ { "statistic": "COUNT", "value": 84 }
  ],
  "availableTags": [
    {
      "tag": "ingredient",
      "values": [ "FLTO", "CHED", "LETC", "GRBF",
                  "COTO", "JACK", "TMTO", "SLSA"]
    }
  ]
}
```
$ curl localhost:8087/actuator/metrics/tacocloud?tag=ingredient:FLTO
```json
{
  "name": "tacocloud",
  "measurements": [
    { "statistic": "COUNT", "value": 39 }
  ],
  "availableTags": []
}
```

### 创建完全自定义的端点
Actuator的端点可以像是RESTApi一样被消费，这可能会让你认为创建与注册一个 Actuator自定义端点就像创建一个RestController一样。  
但事实上，我们先前也提到过，Actuator的端点也会注册为JMX bean，这意味着它们可以通过JMX客户端来访问。

所以不同于用 @Controller 或 @RestController 注解的类，Actuator 端点使用带有 @Endpoint 注解的类定义。
更重要的是，不再使用 HTTP 命名注解，诸如 @GetMapping、@PostMapping 或 @DeleteMapping。
Actuator 端点用 @ReadOperation、@WriteOperation 和 @DeleteOperation 注解。   
但这些注解并不意味着任何特定的通信机制，事实上，允许 Actuator 通过各种通信机制进行通信，HTTP 和 JMX 开箱即用。

以下是一个简单的例子，它创建了一个名为 Notes 的端点，用于记录信息
```java
@Component
@Endpoint(id="notes", enableByDefault=true)
public class NotesEndpoint {

  private List<Note> notes = new ArrayList<>();

  @ReadOperation
  public List<Note> notes() {
    return notes;
  }

  @WriteOperation
  public List<Note> addNote(String text) {
    notes.add(new Note(text));
    return notes;
  }

  @DeleteOperation
  public List<Note> deleteNote(int index) {
    if (index < notes.size()) {
      notes.remove(index);
    }
    return notes;
  }

  @RequiredArgsConstructor
  private class Note {
    @Getter
    private Date time = new Date();
    @Getter
    private final String text;
  }
}
```

该端点是一个简单的便笺端点。在该端点中，用户可以使用写入操作增加一个便笺，使用读取操作读取便笺列表，然后使用删除操作删除便笺。诚然，这个端点对于 Actuator 来说不是很有用，但是当您考虑开箱即用的 Actuator 端点会涉及太多领域时，很难想出一个实用的自定义示例。

需要注意的是，尽管这里只演示了使用 HTTP 如何与端点交互，但它还作为 MBean 公开了，可以使用任何您熟悉的 JMX 客户端进行访问。如果您想将其限制为仅公开 HTTP 端点，您可以使用 @WebEndpoint 而不是 @endpoint 注解端点类：

```java
@Component
@WebEndpoint(id="notes", enableByDefault=true)
public class NotesEndpoint {
  ...
}
```
类似的，如果您喜欢只使用 MBean 的端点，请使用 @JmxEndpoint 注解。

## 保护Actuator端点(通过SpringSecurity)
对于 Actuator 提供的信息，可能您并不想被间谍窥探到。 特别由于 Actuator 提供了可以更改环境属性，以及调整日志记录级别的操作(发送对应的post请求即可)，那么在设计上也应该使只允许具有适当访问权限的客户，才能使用 Actuator 

Actuator 端点的安全性是通过 Spring Security 来实现的。 
为了保护 Actuator 端点，您需要在应用程序中添加 Spring Security 依赖项，然后配置 Spring Security，以允许对 Actuator 端点的访问。 

由于Actuator端点就像是一个RESTApi一样，所以我们可以使用SpringSecurity保护其他API一样的配置方法来对其进行保护。

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
  http
    .authorizeRequests()
    .antMatchers("/actuator/**").hasRole("ADMIN")
    .and()
    .httpBasic();
}
```
这里我们要求所有请求都要来自具有 ROLE_ADMIN 权限的用户。另外还配置了 HTTP 基本身份验证，以便客户端应用程序，可以在请求头 Authorization 属性中，传输编码过的身份验证信息。

不过也许你不想对baseurl进行硬编码，并且对于不同的端口你想有不同的权限控制，那么可以SpringSecurity提供的EndpointRequest类来实现。

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
  http
    .requestMatcher(EndpointRequest.toAnyEndpoint().excluding("health", "info"))
    // .requestMatcher(EndpointRequest.to("beans", "threaddump", "loggers"))
    .authorizeRequests()
    .anyRequest().hasRole("ADMIN")
    .and()
    .httpBasic();
}
```

以上就是有关Actuator的内容，但是Actuator提供的均为原生的端点，其一般是面向其他程序提供信息的而非直接面向用户,如果想要更好的展示，可以使用Spring Boot Admin来进行展示。

---

## 使用Spring Boot Admin提供的UI界面更好地获取Actuator端点信息

俗话说，一图胜千言。对于众多的应用使用者来说，一个对用户友好的 Web 界面抵得上一千次 API 调用。
Spring Boot Admin 就是一个管理型前端 web 应用程序，它使 Actuator 端点更易于人类使用
它分为两个主要组件：Spring Boot Admin 服务端及其客户端。

服务端收集和管理数据，显示从一个或多个 Spring Boot 应用程序获得的 Actuator 数据。  
相对应的提供数据的应用程序，被标识为 Spring Boot Admin 客户端，  
(注意这里的面向SpringBootAdminServer的client是其他Spring程序, 其提供给SpringBootAdmin其Actuator信息，)

![SpringBootAdmin-架构](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/SpringBootAdmin-架构.2yd6lho4vba0.webp)

### 创建SpringBootAdminServer
SpringAdminServer一般是一个独立的SpringBoot应用程序.

一个简单的创建流程如下
- 引入`spring-boot-admin-starter-server`
- 在启动类上添加`@EnableAdminServer`注解
- (往往需要)设置端口号

启动项目，访问端口后就是一个简单的SpringBootAdminServer的UI界面,但是其没有任何内容，因为没有任何程序被注册到SpringBootAdminServer上

### 创建SpringBootAdminClient (注册到SpringBootAdminServer)
有两种方式可以将应用程序注册到 Spring Boot Admin 服务端：
- 客户端手动注册到 Spring Boot Admin 服务端
- 在微服务架构下间接注册 - (SpringBootAdmin通过Eureka拉取注册的服务)

第二种方法书中没有介绍，也不在本文的讨论范围内，这里只介绍第一种方法

- 引入`spring-boot-admin-starter-client`依赖
- 在配置文件中添加`spring.boot.admin.client.url`属性，指向SpringBootAdminServer的地址

That's it.

### 使用SpringBootAdminServer

因为SpringBootAdminServer的UI界面已经很友好了，这里就不再赘述了

### 保护SpringBootAdminServer
SpringBootAdminServer本身自然也需要进行保护，这是符合逻辑的。就像我们对待其他的Web应用一样，可以使用Spring Security来对其进行保护。在这里不展开详细讨论。

然而，重点并不在于保护SpringBootAdminServer本身，而是另一个问题：
正如之前在Actuator章节中提到的，Actuator的端点应当受到Spring Security的保护。这是完全正确的，但现在出现了一个问题，那就是如何让SpringBootAdminServer能够获取到受保护的Actuator信息呢？
尽管带有保护机制的Actuator端点能够阻止未经授权的请求，但同时也会拦截SpringBootAdminServer的请求，导致SpringBootAdminServer无法获取Actuator的信息。

解决方案其实很简单直接(不讨论Eureka注册发现的情况，而是Client直接向Server注册自己的情况),就是在Client向Server注册的时候直接带上访问自己Actuator端点的凭证信息。

```yaml
spring:
  boot:
    admin:
      client:
        url: SPRING_BOOT_ADMIN_URL
        username: SPRING_BOOT_ADMIN_USERNAME
        password: SPRING_BOOT_ADMIN_PASSWORD
```

## 总结

* Spring Boot Actuator 提供了几个端点 ,其即可做为 HTTP 也可作为 JMX MBean，让您可以窥探 Spring Boot 应用程序的内部工作情况。
* 默认情况下，大多数 Actuator 端点处于禁用状态，但可以有选择地通过设置 `management.endpoints.web.exposure.include` 和 `management.endpoints.web.exposure.exclude` 调整停启用状态。
* 某些端点（如 /logger 和 /env 端点）允许写操作，动态更改正在运行的应用程序的配置。
* 有关应用程序的构建和 Git 提交的详细信息，可以通过 /info 端点获取。
* 应用程序的Health可能会受到自定义 health indicator的影响，其可以追踪一些外部集成应用的健康状况。
* ，可以通过 Micrometer 注册自定义应用程序度量指标。Micrometer 支撑了 Spring Boot 应用程序与几个流行指标引擎集成，包括 Datadog、New Relic 和 Prometheus 等。
* Actuator web 端点可以使用 Spring Security 进行保护，就像保护任何 Spring Web 应用程序中的其他端点那样。

---

* Spring Boot Admin 服务端消费一个或多个 Spring Boot 应用程序 Actuator 端点的数据，并以用户友好的方式显示在 web 界面中。
* Spring Boot 应用程序可以主动将自己注册为 Admin 客户端，也可以由 Admin 服务端通过 Eureka 发现它们。
* 与捕获应用程序快照的 Actuator 端点不同，Admin 服务端能够显示应用程序的实时状态。
* Admin 服务端可以方便地过滤 Actuator 端点数据，并在图形中直观地显示。
* 因为 Admin 服务端本质就是一个 Spring Boot 应用程序， 所以可以应用任何 Spring Security 提供的安全措施。
