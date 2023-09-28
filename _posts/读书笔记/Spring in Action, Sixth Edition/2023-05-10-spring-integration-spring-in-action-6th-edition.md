---
layout: post
title: "Spring集成的大致使用"
date: 2023-05-10 01:31 +0800
category: [ 读书笔记, "spring实战第六版" ]
tag: [ Spring,Spring Integration,Spring集成 ]
---

集成的意思可以直接参考英文Integration，其还可以翻译为 一体化

具体来说,有时有将系统与外部的其他系统进行集成的需求。比如，你的系统需要从外部系统中获取数据，或者将数据发送给外部系统，或者是将数据转换为其他格式的数据发送给外部系统。
Spring集成简单来说就是 就是让Spring能够与外部的系统发送与接收消息， 并针对消息构建一条流水线进行处理， 从而实现系统间的集成。

其实现的奠基石就是上一章中讲解的异步消息 的消息队列中间件。但是Spring集成是一个更高程度的抽象，你的代码中可以完全没有底层的对于消息处理的显式代码。
本章的核心在于介绍Spring集成中组成集成流水线的一些主要组件，以及如何使用这些组件来构建集成流水线。

书中案例是： 从邮件服务器中(外部系统)接受邮件 → 将邮件信息进行读取、处理、转换为订单信息 →
通过REST请求将订单信息发送给另一个外部系统(Taco后厨)，创建一个订单
作者的这个案例中，该集成实现还是比较复杂的，但很多其实并非是Spring集成本身的复杂度。

> Spring集成是一个很好理解，但是具体到实践的话根据不同的使用场景，实现难度将会非常高。书中以及本博文仅仅是对于Spring集成的概念，组件介绍与使用做一个简单的
> know how 的介绍。
{: .prompt-info}

## 一个简单流的创建 - 初步使用Spring集成

### 依赖导入 与 网关入口的创建

导入依赖，其中`spring-boot-starter-integration`是Spring集成的核心依赖，而具体根据不同的使用需求，可以进一步导入不同的集成组件。
例如`spring-integration-file`，此模块是用于与外部系统集成的二十多个端点模块之一。我们将在下一节中更多地讨论端点模块。
但是，就目前而言，要知道文件端点模块提供了将文件从文件系统提取到集成流或将数据从流写入文件系统的能力。 file - spring - file

然后是创建网关GateWay，其作为集成流的入口(或者出口)。 形式上是一个接口，我们只需要定义其接口方法即可，而具体的实现则是由Spring集成来完成。
在接口中，我们需要使用@MessageGateway注解来标注该接口，以便Spring集成能够识别该接口并将其作为网关使用。注解的属性设定为当调用该接口方法时，将消息发送哪一个消息通道中。

消息通道是一个集成流水线的组件，其用于将消息从一个组件传递到另一个组件。对于集成中常用组件的介绍将在下一节中进行介绍。目前来说，起就是一个消息传递的通道。

```java
@MessagingGateway(defaultRequestChannel="textInChannel")
public interface FileWriterGateway {

  void writeToFile(
      @Header(FileHeaders.FILENAME) String filename,
      String data);

}
```

例如上述代码中，当调用`writeToFile`方法时，将消息发送到`textInChannel`
通道中。而@Header是用于设置消息头的，这里设置了文件名的消息头。这使得filename属性将会存储在消息的header中而不是负载中。

### 集成流的创建 - xml

用xml进行配置是非常古早的做法，现在已经不推荐使用了，但是作者还是进行了类似于介绍荣誉老将般地简单说明了一下。

```xml

<beans>
  <int:channel id="textInChannel"/>

  <int:transformer id="upperCase"
                   input-channel="textInChannel"
                   output-channel="fileWriterChannel"
                   expression="payload.toUpperCase()"/>

  <int:channel id="fileWriterChannel"/>

  <int-file:outbound-channel-adapter id="writer"
                                     channel="fileWriterChannel"
                                     directory="/tmp/sia6/files"
                                     mode="APPEND"
                                     append-new-line="true"/>
</beans>
```

目前要简单理解这和流，其实看一下这种图就能大致明白了。

![集成流-简单文件流](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/集成流-简单文件流.595vvpakxk00.webp)
第一个网关是我们之前使用 接口的方式进行定义的。在运行时Spring会自动生成动态代理，当使用Gateway接口实现类方法的时候，就是调用了该集成流

而具体组成流的每一个组件就是在xml中定义的bean，具体来说
这段配置中，可以比较清楚地看到了注册了4个bean，其中2个是消息队列(消息通道), 一个转换器，一个出站通道适配器。

- TextInChannel是一个消息通道，其作为网关发送消息的载体。
- UpperCase是一个转换器，其作用是将消息转换为大写。 具体行为可以在xml文件中看到payload.tuUpperCase()的定义
- FileWriterChannel是一个消息通道，其作为转换器的输出通道，也是出站通道适配器的输入通道.作用就是一个中间的消息载体
- 最后的writer是一个出站通道适配器，其作用是将消息写入文件系统。 具体的配置中，可以看到其配置了文件的路径，写入模式配置为追加模式，以及是否追加新行。

> 虽然对于集成流的配置结束了。但注意Spring默认不会对项目路径中的xml配置文件进行读取。所以你还需要额外做的事情是通过 包扫描
> 或者 注解的方式让Spring读取xml配置文件
{: .prompt-warning}

例如

```java
@Configuration
@ImportResource("classpath:/filewriter-config.xml") // 上述xml配置文件的项目相对路径
public class FileWriterIntegrationConfig { ... }
```

### 集成流的创建 - Java注解

同样还是这个流，这里再贴下图
![集成流-简单文件流](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/集成流-简单文件流.595vvpakxk00.webp)

```java
@Configuration
public class FileWriterIntegrationConfig {

  @Bean
  @Transformer(inputChannel="textInChannel", outputChannel="fileWriterChannel")
  public GenericTransformer<String, String> upperCaseTransformer() {
    return text -> text.toUpperCase();
  }

  @Bean
  @ServiceActivator(inputChannel="fileWriterChannel")
  public FileWritingMessageHandler fileWriter() {
    FileWritingMessageHandler handler = new FileWritingMessageHandler(new File("/tmp/sia6/files"));
    handler.setExpectReply(false);
    handler.setFileExistsMode(FileExistsMode.APPEND);
    handler.setAppendNewLine(true);
    return handler;
  }

}
```

你可能注意到了我们只配置了 转换器 以及出站通道适配器(ServiceActivator) 这两个Bean，而没有对消息通道进行配置。
具体来说，如在转换器@Transformer注解中，直接指定了InputChannel为textInChannel,而没有textInChannel的定义。
但不用担心，Spring会自动为我们创建消息通道。不过如果你相对消息通道做一些额外的自定义配置，你也当然自己手动注册消息通道来覆盖Spring自动创建的消息通道。

GenericTransformer和FileWritingMessageHandler是Spring集成流中提供的两个组件，其定义了一些规范与基本实现。
具体在直接注解配置的方法中，其对于返回的Bean进行了配置，例如Transformer的的入参与出参通过泛型进行了定义，操作就是将textUpperCase；
而另一个出站通道适配器的配置，其通过构造器注入了文件路径，以及其他的一些配置。(
出站通道适配器是对于Spring集成流的概念，而对于SpringBean来单独来说，其作为一个ServiceActivator)

值得一提的是在FileWritingMessageHandler中，其设置了setExpectReply(false)
。表示其不预期一个消息返回值，即这是一个单向的消息通道，消息消费后(file被写入后)将不会返回任何消息。

### 集成流的创建 - Java DSL (domain-specific language)

对于集成流的创建来说，Java注解的方式相较于Xml的格式已经好了很多。但是一个更加简洁的方式就是使用Java DSL。

对于之前的xml方法与java注解方式来说，我们操作的本质就是创建一个又一个的组件Bean，但是对于流的创建来说，我们更关心的是组件之间的关系，而不是组件本身。
即我们想要直接整体地把握一个流的定义，这个流水线的作用，而不是其中具体的一个个组件在做什么。
所以Java DSL就是为了解决这个问题而生的。使用DSL, 我们直接在一个方法中定义整个流。

如果你有使用过Java StreamAPI的经验，那么你会发现Java DSL的使用方式与StreamAPI的使用方式非常相似。

```java
@Configuration
public class FileWriterIntegrationConfig {

  @Bean
  public IntegrationFlow fileWriterFlow() {
    return IntegrationFlows
      .from(MessageChannels.direct("textInChannel"))
      .<String, String>transform(t -> t.toUpperCase())
      .handle(Files
        .outboundAdapter(new File("/tmp/sia6/files"))
        .fileExistsMode(FileExistsMode.APPEND)
        .appendNewLine(true))
      .get();
  }
}
```

你应该还记得之前那张集成流的图像，上述代码就是这个流以DSL的方式的定义。所有的组件都在一起通过 函数型编程的风格，使用`.`
操作符连接不同的组件。而组件的定义就是在函数中进行的。最终返回一个Spring集成定义的IntegrationFlow对象。
(不用考虑集成流对象的调用，因为流对象对于使用者来说应当是无法感知的，使用时通过我们之间定义的Gateway进行调用)

同样的，除了第一个消息通道textInChannel需要显式声明(这是因为需要将其与GateWay进行绑定),我们不需要对其他的消息通道进行显式的声明。
因为消息队列本身会被自动创建。以及，如何你对消息队列(通道)有自定义的需求，也可以进行添加。不过就这个简单案例来说，我们不需要对消息通道进行自定义。

all in one 的好处时我们可以只通过一个方法获取整个流的全貌，但是这也存在一些问题。因为这只是一个简单的流定义，所以代码不是很长。
但想象一个非常复杂的流，如果把所有的代码全集中在一个方法中显然是非常阅读不友好的。一个好的解决方法是将复杂的流拆分成多个方法，然后使用方法引用等方式来简化函数中的代码。

## Spring集成的主要组件介绍

### `Gateways`

Pass data into an integration flow via an interface.
![集成流-网关](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/集成流-网关.61x8g0fvh6c0.webp)

网关作为一个接口，用于将外部的数据传入到集成流中, 同时若流水线有消息返回的话，对消息进行接收

之前的简单案例中，我们定义的GateWay是没有返回消息的，但是实际上我们也可以定义返回消息的GateWay。以下是一个简单的案例。

```java
@Component
@MessagingGateway(defaultRequestChannel="inChannel", defaultReplyChannel="outChannel")
public interface UpperCaseGateway {
  String uppercase(String in);
}


// 在DSL中指明 入口和出口 ，从而使其与GateWay进行绑定
@Bean
public IntegrationFlow uppercaseFlow() {
  return IntegrationFlows
    .from("inChannel")  // 入口
    .<String, String> transform(s -> s.toUpperCase())
    .channel("outChannel") // 出口
    .get();
}
```

> 之前在简单案例中我可能将网关说成是一个流的入口或出口，其实这是不准确的。 
> 
> 因为GateWay某种程度上是脱离于流的概念，因为不同于其他组件，在DSL中无法定义一个GateWay，GateWay只能以接口定义，然后Spring动态实现的方式注入。  
> 
> 换句话说，GateWay是一个流的接入点，但其本身并不是流的一部分。流的入口或出口严格意义上来说是后面提到的 channel
> Adapter组件。 或者说，GateWay和channel Adapter组件共同组成了流的入口和出口。
{: .prompt-info}

### `Channels`

Pass messages from one element to another.
![集成流-消息通道](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/集成流-消息通道.php8mf47rao.webp)

字面意义上的消息通道，消息从中流过，从一个组件到达另一个组件。 从之前简单流的案例中，消息通道仿佛完全没有存在的必要。因为我们只是简单地将消息从一个组件传递到另一个组件。
但有一些场景需要进行复杂的消息路由，这时候就需要用到不用类型的消息通道了。就和Rabbitmq的Exchange一样，消息通道也有不同的类型。

- PublishSubscribeChannel  
  广播消息通道，消息会被广播到所有的订阅者。即该通道可以设置多个订阅者，每个订阅者都会收到消息。
- QueueChannel  
  FIFO,不支持多订阅，只有一个订阅者可以收到消息。
- PriorityChannel
  同Queue，只是消息会按照优先级进行排序。
- RendezvousChannel
  字面意思上翻译为约会通道，还是Queue的结构，但是发送者会阻塞直到有一个订阅者接收到消息。这么做的效果是可以让发送者和接收者同步。
- DirectChannel  
  类似于广播，但是不支持多订阅，同时最重要的一点该通道
  会以线程唤醒的方式激活同一个线程的消费者接受消息，即发送者和接受者在同一线程中。由此，该消息通道的特点是可以传递事务(
  Transaction)
- ExecutorChannel  
  类似于Direct，但是消息由线程池发送，所以不支持传递事务
- FluxMessageChannel  
  响应式编程概念的队列，书中暂时没有解释

如果要使用不同的消息通道，需要手动注入，消息通道的名称默认就是方法名

```java
@Bean
public MessageChannel orderChannel() {
  return new PublishSubscribeChannel();
}


// Java注解配置中使用
@ServiceActovator(inputChannel="orderChannel")


// DSL中使用
@Bean
public IntegrationFlow orderFlow() {
  return IntegrationFlows
    ...
    .channel("orderChannel")
    ...
    .get();
}
```

注意，对于Queue结构的消息通道来说，需要消息消费者主动从通道Queue中pull消息。所以还需要设定pull的频率

```java
// 如果使用的是Queue结构的Channel,需要设置手动pull消息的频率
@ServiceActivator(inputChannel="orderChannel", poller=@Poller(fixedRate="1000"))
```

### `Filters`

Conditionally allow messages to pass through the flow based on some criteria.
![集成流-过滤器](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/集成流-过滤器.6gxls4e3qt40.webp)

没有需要额外说明的，直接看使用示例

```java
//从number中筛选中偶数

// Java注解配置
@Filter(inputChannel="numberChannel", outputChannel="evenNumberChannel")
public boolean evenNumberFilter(Integer number) {
  return number % 2 == 0;
}


// DSL
@Bean
public IntegrationFlow evenNumberFlow(AtomicInteger integerSource) {
  return IntegrationFlows
    ...
    .<Integer>filter((p) -> p % 2 == 0)
    ...
    .get();
}

```

### `Transformers`

Change message values and/or convert message payloads from one type to another.
![集成流-转换器](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/集成流-转换器.3g0kklwjh4y0.webp)

转换器对消息执行一些操作，通常会产生不同的消息，并且可能会产生不同的负载类型.
具体来说，转换既可以是做一些简单的数值计算，或者是一些复杂的mapping工作将信息完全转换，例如将一个标识符转换为一个具体的对象。

书中给出的例子是将 一个阿拉伯数字 转为 罗马数字

```java
// 假设已经有一个RomanNubmers工具类完成了具体的转换逻辑

// java 注解配置
@Bean
@Transformer(inputChannel="numberChannel", outputChannel="romanNumberChannel")
public GenericTransformer<Integer, String> romanNumTransformer() {
  return RomanNumbers::toRoman;
}


// DSL
@Bean
public IntegrationFlow transformerFlow() {
  return IntegrationFlows
    ...
    .transform(RomanNumbers::toRoman)
    ...
    .get();
}

// 除了方法引用，还直接传递一个 实现Transformer 接口的类作为入参
    ...
    .transform(romanNumberTransformer)
    ...
```

### `Routers`

Direct messages to one of several channels, typically based on message headers.
![集成流-路由器](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/集成流-路由器.586fsc4t6ok0.webp)

```java
@Bean
@Router(inputChannel="numberChannel")
public AbstractMessageRouter evenOddRouter() {
  return new AbstractMessageRouter() {
  @Override
    protected Collection<MessageChannel> determineTargetChannels(Message<?> message) {
      Integer number = (Integer) message.getPayload();
      if (number % 2 == 0) return Collections.singleton(evenChannel());
      else return Collections.singleton(oddChannel());
    }
  };
}
```

DSL中使用route方法完成，泛型参数分别定义入参与出参的类型.
由于描述的是一整个流的定义，所以添加了一些额外代码表示路由之后的操作.

```java
@Bean
public IntegrationFlow numberRoutingFlow(AtomicInteger source) {
  return IntegrationFlows
    ...
      .<Integer, String>route(n -> n%2==0 ? "EVEN":"ODD", mapping -> mapping
        .subFlowMapping("EVEN", sf -> sf
          .<Integer, Integer>transform(n -> n * 10)
          .handle((i,h) -> { ... })
          )
        .subFlowMapping("ODD", sf -> sf
          .transform(RomanNumbers::toRoman)
          .handle((i,h) -> { ... })
          )
        )
      .get();
}
```

### `Splitters`

Split incoming messages into two or more messages, each sent to different channels.
![集成流-分离器](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/集成流-分离器.z4vv6su38vk.webp)
将一个消息分裂为子消息，使用场景有：

- 消息是一个同类别的集合，需要将集合拆分为单个消息处理.例如一个用户集群，需要将其拆分为每一个具体的用户单独处理。
- 消息是一个聚合对象，需要根据不同类型进一步拆分。例如一个订单消息，包含订单本身的支付信息与其包含的商品信息。

对于前者来说，实现很简单，Spring会智能地帮我们完成拆分。

```java
// java 注解配置
@Splitter(inputChannel="lineItemsChannel", outputChannel="lineItemChannel")
public List<LineItem> lineItemSplitter(List<LineItem> lineItems) {
  return lineItems; // 不要额外处理，直接返回即可，Spring会自动拆分
}

// DSL
return IntegrationFlows
  ...
    .split() // Split无参方法，会自动拆分
    .route( // Split后往往紧接着路由
    ...

```

而如果需要进行自定义的规则拆分，则需要手动实现拆分逻辑然后注入。

```java
// 在某处已经完成了自定义的split逻辑
public class OrderSplitter {
  public Collection<Object> splitOrderIntoParts(PurchaseOrder po) {
    ...
    return parts;
  }
}

// java注解配置
@Bean
@Splitter(inputChannel="poChannel", outputChannel="splitOrderChannel")
public OrderSplitter orderSplitter() {
  return new OrderSplitter();
}

// DSL
return IntegrationFlows
  ...
    .split(orderSplitter())
```

### `Aggregators`

The opposite of splitters, combining multiple messages coming in from separate channels into a single message.

聚合器，也就是Split的反向操作，在编码上与Splitter类似，所以不赘述了.

### `Service activators`

Hand a message off to some Java method for processing, and then publish the return value on an output channel.
![集成流---服务激活器](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/集成流---服务激活器.51hsj34rcj00.webp)

服务激活器是是整个集成流的核心，负责将消息转换为业务逻辑的处理，然后将处理结果发送到下游的通道中。(
或者没有结果，直接作为整个流的终点。就像简单案例中的文件写入)

即，之前那么多的组件都是对消息进行整理、重组、分类、转发等等工作，而到了这一步，是真正开始了"加工" 流程。 我们要call
一些系统中的Function了。

```java
// java注解配置
@Bean
@ServiceActivator(inputChannel="orderChannel", outputChannel="completeChannel")
public GenericHandler<EmailOrder> orderHandler( OrderRepository orderRepo) {
  return (payload, headers) -> {
    return orderRepo.save(payload);
  };
}

// DSL
public IntegrationFlow orderFlow(OrderRepository orderRepo) {
  return IntegrationFlows
    ...
      .<EmailOrder>handle((payload, headers) -> {
      return orderRepo.save(payload);
      })
    ...
      .get();
}

```

### `Channel adapters`

Connect a channel to some external system or transport. Can either accept input or write to the external system.
![集成流---通道适配器](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/集成流---通道适配器.3d3lx1icnsk0.webp)

通道适配器是整个集成流的入口与出口,对于一些使用场景来说，其是非必要的，因为其直接接收网关传递的消息作为开始，就像最开始介绍的简单文件流那样，网管会直接传递要保存的信息。

但在一些场景中，流需要维护自己的状态，使用者不知道这些状态(或者没有必要知道)
，只能通过GateWay发送启动命令，那么此时就需要ChannelAdapter作为起点来完成整个流的初始化。
例如下述案例中，流需要从一个原子类中获取初始值

```java
@Bean
@InboundChannelAdapter( poller=@Poller(fixedRate="1000"), channel="numberChannel")
public MessageSource<Integer> numberSource(AtomicInteger source) {
  return () -> {
    return new GenericMessage<>(source.getAndIncrement());
  };
}

//DSL
@Bean
public IntegrationFlow someFlow(AtomicInteger integerSource) { // 入参中声明
  return IntegrationFlows
      .from(integerSource, "getAndIncrement",
        c -> c.poller(Pollers.fixedRate(1000)))
    ...
}
```

或者在一些场景中，最终流结束之后，当下最为终点的组件的返回值不能很好地表述整个流的运行结果，此时也需要多加一个ChannelAdapter作为终点来创建一个最终结果返回网关的消息。
这里仅做一个说明，没有代码示例。

### `Endpoints Module`

Endpoint Module 端点模块 可以理解为一个 第三方完成的流 将其封装为模块给Spring进行复用。我们直接将其作为我们自定义流的一个端点，融入到自己的流中。

还记得吗，我们一开始使用SpringIntegration的时候，我们不仅引入了核心依赖，还引入了一个`spring-integration-file`
。这就是一个端点模块，其已经实现了一些文件操作，
通过使用这些端点模块，我们可以简化流的代码。

就像之前所说的那样，一个端点模块可以理解为 一个第三方的实现完备的流，然后封装为了一个模块。这意味着根据具体模块提供的功能不同，其提供的一些操作可以充当一个流中的任意一个角色。
一般来说，之前的提到的通道适配器、服务激活器等往往都会用到端点模块。

Spring提供的端点模块如下表所示：
这张表很长，我是故意贴了一个很长的表格，用其当做整个章节的结束，翻过这个表格就是最后一章。作者的一个集成流实践 -
将Email转换Taco订单的实现。

| Module| Dependency artifact ID (Group ID: org.springframework.integration) | |
| --- | --- |
| AMQP | spring-integration-amqp |
| Spring application events | spring-integration-event |
| RSS and Atom | spring-integration-feed |
| Filesystem | spring-integration-file |
| FTP/FTPS | spring-integration-ftp |
| GemFire | spring-integration-gemfire |
| HTTP | spring-integration-http |
| JDBC | spring-integration-jdbc |
| JPA | spring-integration-jpa |
| JMS | spring-integration-jms |
| JMX | spring-integration-jmx |
| Kafka | spring-integration-kafka |
| Email | spring-integration-mail |
| MongoDB | spring-integration-mongodb |
| MQTT | spring-integration-mqtt |
| R2DBC | spring-integration-r2dbc |
| Redis | spring-integration-redis |
| RMI | spring-integration-rmi |
| RSocket | spring-integration-rsocket |
| SFTP | spring-integration-sftp |
| STOMP | spring-integration-stomp |
| Stream | spring-integration-stream |
| Syslog | spring-integration-syslog |
| TCP/UDP | spring-integration-ip |
| WebFlux | spring-integration-webflux |
| Web Services | spring-integration-ws |
| WebSocket | spring-integration-websocket |
| XMPP | spring-integration-xmpp |
| ZeroMQ | spring-integration-zeromq |
| ZooKeeper | spring-integration-zookeeper |

## 一个复杂流的实现 - Email集成

Taco Cloud 可以让用户通过电子邮件提交他们的 Taco 设计并下订单。
然而，如果这一功能被广泛使用，每天接收到大量电子邮件订单，就需要雇用临时工来处理这些订单。
但临时工的工作无非是打开电子邮件、读取订单信息，并将订单信息输入 Taco Cloud 的订单系统中。

在本节中，我们将实现一个集成信息流，用于轮询 Taco Cloud 收件箱中的 Taco 订单电子邮件，并解析邮件订单的详细信息，然后提交订单到
Taco Cloud 进行处理。

具体来说，我们将使用入站通道适配器从邮箱端点模块中提取 Taco Cloud 收件箱中的邮件，
然后将电子邮件解析为订单对象，并将其发送到另一个处理器中，该处理器将订单提交到 Taco Cloud 的 REST API 进行处理。

其大致上的流程如下图所示：
![Spring集成流-Email](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/Spring集成流-Email.5yxezrw0ies0.webp)

DSL的大致定义如下：

```java
@Configuration
public class TacoOrderEmailIntegrationConfig {

  @Bean
  public IntegrationFlow tacoOrderEmailFlow(
    EmailProperties emailProps,
    EmailToOrderTransformer emailToOrderTransformer,
    OrderSubmitMessageHandler orderSubmitHandler) {

  return IntegrationFlows
    .from(Mail.imapInboundAdapter(emailProps.getImapUrl()),
      e -> e.poller( Pollers.fixedDelay(emailProps.getPollRate())))
    .transform(emailToOrderTransformer)
    .handle(orderSubmitHandler)
    .get();
  }
}
```

其实这就差不多是集成流所有的SpringIntegration部分了，剩下的就是这些transform、handle的具体实现.
所以正如本博文一开始提到的那样，构建一个集成流的工作是较为简单的,一个DSL定义的IntegrationFlow可以简洁,集成流的复杂性往往在具体的组件实现上。

### Email 端点模块部分

具体代码太长了就不贴了，主要阐述思路

导入`spring-integration-mail`端点模块，其需要一个url来获取邮件。作者维护了一个`EmailProperties`
类，其从配置文件中读取自定义的配置信息，然后类方法getImapUrl()将配置信息转为一个符合格式的url,getPollRate()则是获取轮询的频率。

### emailToOrderTransformer 模块

这一块是整个流中最重要与最复杂的模块，怎么把一个邮箱中的信息提取出来然后转为一个订单消息。
如果完全放开思路，这一部分甚至可以做成一个完全独立的项目，使用机器学习来实现,本项目只要给他call message即可 😂

言归正传，作者应该是和客户协商了一些规则，当邮件负载中包含了TACO ORDER 关键字后，就会开始进一步处理。
具体来说，用到了端点模块的各种get方法获取信息，然后用(应该也是端点模块提供的)contains方法完成关键字的判断。

接着就是Transformer的过程，作者用`split("\r?\n")` 分割content ，
> 根据换行符 "\n" 或者 "\r\n" 来分割字符串的，其中 "\r?" 表示 "\r" 可能出现 0 次或者 1
> 次，因为不同的操作系统对于文本文件中的换行符的表示方式不同，有些操作系统使用 "\r\n"，有些使用 "\n"
> ，所以使用这个正则表达式能够兼容不同操作系统的换行符。
{: .prompt-tip}

然后

```java
 if (line.trim().length() > 0 && line.contains(":")) {
        String[] lineSplit = line.split(":");
```

接着按照固定顺序依次获取 tacoName和IngredientList。对于IngredientList，其通过split(",")分割，然后trim()
去除空格，最后根据SpringContext中的一个List对IngredientCode找mapping的Ingredient对象。

至此，可以创建一个Taco对象了(或TacoList)，但还需要将其transfer 为一个TacoOrder对象。
作者直接将email地址作为用户下单的凭据创建Order

```java
public class EmailOrder {
  private final String email;
  private List<Taco> tacos = new ArrayList<>();

  public void addTaco(Taco taco) {
    this.tacos.add(taco);
  }
}
```

### orderSubmitHandler 模块

有了EmailOrder之后，作者很直接地用RestTemplate发送过去了……

```java
@Component
public class OrderSubmitMessageHandler implements GenericHandler<EmailOrder> {

  private RestTemplate rest;
  private ApiProperties apiProps;

  public OrderSubmitMessageHandler(ApiProperties apiProps, RestTemplate rest) {
    this.apiProps = apiProps;
    this.rest = rest;
  }
  
  @Override
  public Object handle(EmailOrder order, MessageHeaders headers) {
    rest.postForObject(apiProps.getUrl(), order, String.class);
    return null;
  }
}
```

之前的REST Service应该没有这个API,所以按照道理讲还需要在REST Service模块中实现该API，但作者没有完善整个项目
同时只有一个email地址的话，也不好获取JWT来鉴权. 如果api没有鉴权的话，那么这个api还是比较危险的。

anyway ，其实这些问题和Spring集成流本身没有太大关系，所以就不再深究了。

不过值得一提的是，作者补充道既然要使用RestTemplate那么就要引入`spring-boot-starter-web`
，但是我们的项目严格来说不是一个web项目(这是一个单独的集成流项目),所以自然不要tomcat之类的玩意，
通过添加如下配置来让项目不启动web服务

```yaml
spring:
  main:
  web-application-type: none
```

spring.main.web-application-type 属性可以被设置为 servlet、reactive 或是 none，当 Spring MVC 在 classpath 中时，自动配置将这个值设置为
servlet。但是这里需要将其重写为 none，这样 Spring MVC 和 Tomcat 就不会自动配置了

## 总结

* Spring Integration
  使得程序可以接受外部消息进行流水线处理，并再发送给其他外部系统，从而实现了不同系统之间的连接，这就是集成(一体化)
* Integration 流可以以 XML、Java 或 Java DSL 配置的风格进行定义。
* 消息网关和通道适配器充当集成信息流的入口和出口。
* 消息可以被转化，分割，聚集，路由, 然后通过服务激活器做"加工"
* 可以使用 第三方端口模块 来简化与增强流的实现
