---
layout: post
title: "在Spring中使用消息队列中间件发送异步消息"
date: 2023-05-07 02:37 +0800
category: [ 读书笔记, "spring实战第六版" ]
tag: [ Spring,发送异步消息 ]
---

同步 通信有它的适用场景，这是我们在 REST 中所看到的。但这并不是开发人员可以使用的惟一的应用程序间通信方式。异步
消息传递是一种间接地将消息从一个应用程序发送到另一个应用程序而无需等待响应的方式。这种间接方式解耦了相互通信的应用程序，并带来了更好的伸缩性。

在书中，作者介绍了三种发送异步消息的做法，分别是

- 实现JMS(Java Message Service)规范的 Artemis/ActiveMQ、
- 实现AMQP(Advanced Message Queuing Protocol)规范的RabbitMQ
- Apache Kafka。

## JMS: Artemis/ActiveMQ

JMS定义了在不同java程序中发送异步消息的规范。类似于JDBC规范，在Spring程序中我们可以使用JmsTemplate来完成消息的发送与接收.
而Artemis和ActiveMQ是两个实现了JMS规范的消息中间件,(Artemis是ActiveMQ的升级版，ActiveMQ将会在未来的版本中被废弃)
,提供了JMS规范中定义的所有功能。

在引入相应的依赖后，需要在配置文件中指定消息中间件的地址，端口，用户名，密码等信息，然后在程序中注入JmsTemplate即可使用。

宏观来说，存在两种模型对消息进行处理，其分别是 pull 和 push 模型。

- pull模型中，消息的消费者主动地(一般轮询地)对消息中间件进行查询，看是否有新的消息，如果有则进行消费。形象地说，customer pull
  message from message broker.
- push模型中，消息的消费者是被动地对消息进行接收，需要额外进行消息监听器，当有新的消息到达时，消息中间件会主动地通知消费者，消费者
  进行消费。形象地说，producer push message to message broker.

一般来说，push模型被推崇为最佳实践，因为在pull模型中，消息消费者会阻塞进程(等待直到获得消息),而push模型不会。
但作者补充的一个视角是，对于一些很快就能处理的消息，使用push模型自然是完美的选择。但如何一个消息需要很长时间才能处理，那么(
处于瓶颈位置的角色)可以使用pull模型来获取消息。

具体来说，使用pull模型可以解放consumer的压力。在书中的例子是：
一个订单消息如何可以很快地被处理：例如直接将该消息展示在后厨的大屏幕上,那么对于这个操作来说，使用push模型是和合适的，因为把消息直接进行展示是一个很快速的处理流程。
但是对后厨具体的厨师来说，其不想在认真做菜的时候被消息提示音不断地打扰，对这些后厨来说，当一个菜做好了之后，它主动会去大屏幕上看一下新的订单来决定接下来做什么。
简单来说，后者可以把主要精力放在干活上，而不是监听消息，这在一定程度上解放了厨师的压力。

因此，在处理速度较快的消息时，使用push模型是比较好的选择。但对于处理速度较慢的消息，使用pull模型可以缓解消费者的压力，并提高系统的性能和可靠性。

### JmsTemplate提供的方法重载的不同参数 (使用套路)

- 目标地址: 一般用String，可以使用中间件定义的Destination类型来提供很多有关目的地的信息，但绝大部分情况下直接使用String即可。
- 消息构造:
  可以使用消息中间件提供的构造器进行构造，但一种更简单的方式是直接传递Object对象，让Spring和中间件对其进行序列化。  
  具体来说，可以使用spring默认的消息转换器，或者自己注入一个自定义的消息转换器，例如最流行的Jackson。
- 消息后处理： 有时候在自动将Object转换成消息之后，还要对消息添加一些额外的信息，使用中间件提供的消息后处理器可以很方便地实现这一点。

### 发送消息

- send 发送一个由消息转换器生成的对象
- convertAndSend 发送一个Object，Spring中配置的消息转换器会自动将其转换成消息,可以添加一个后处理器对消息进行额外处理

方法的参数在上一节已经提到过了，对于目的地参数来说，如果不指定String或者Destination对象，那么会发送到默认的目的地中。默认目的地的地址在配置文件中进行设置.

convertAndSend之所以能够完成消息转换的原理是其使用了Spring中的消息转换器，而消息转换器的配置在配置文件中进行设置。
具体可以查看[Spring中的消息转换器]({% post_url /读书笔记/Spring in Action, Sixth Edition/2023-04-25-converter-in-spring %})

对于消息的后处理来说，如果使用的是send方法，可以直接使用消息构建器为消息set property，例如设置header等。
如果使用的是convertAndSend则需要添加一个参数，其一般就是简单的消息后处理器的lambda实现，或者当一个后处理太复杂时，可以抽离为方法然后使用方法引用。

### 接受消息(pull模型)

类似的

- receive 直接指定目的地即可，返回Message对象,Message对象可以直接获取payload或者获取更多有关消息本身的属性，例如header
- receiveAndConvert 同样指定目的地即可，直接返回消息的 经过转换后的payload对象

### 接受消息(push模型)

不要显式地创建与注册一个消息监听器组件，Spring与引入的依赖可以自动完成。
使用一些注解即可。

```java
@Profile("jms-listener")
@Component
public class OrderListener {

  private KitchenUI ui;

  public OrderListener(KitchenUI ui) {
    this.ui = ui;
  }

  @JmsListener(destination = "tacocloud.order.queue")
  public void receiveOrder(TacoOrder order) {
    ui.displayOrder(order);
  }
}
```

这个Component不会被主动地调用，而是每当有消息到来时，OrderListener.receiveOrder方法会被自动调用。
只要添加 `@JmsListener(destination = "tacocloud.order.queue")`注解,就是这么简单

## AMQP: RabbitMQ

JMS很好，但是其限定了只有Java程序才能使用，而AMQP是一个开放的标准，可以被任何语言使用。RabbitMQ是一个AMQP的实现。
同时，Rabbitmq与Artemis最大的不同点在于，Rabbitmq多出来了一个Exchange的概念，而Artemis没有。(
更新：可能artemis已经支持了消息路由了，但先暂时忽略这些不重要的细节)

之前的JMS 示例中 对于消息的发送，消息发送发要直接指明接收方的地址。同时接收方也要直接指明自己监听的地址。这种方式是不灵活的。

在Rabbitmq中，其使用交换机与 routing key的概念来制定复杂的消息路由规则。这种方式可以让消息的发送方不需要知道消息的接收方的地址，

### RabbitMQ101

我们的重点主要集中在how to use的主题中，具体如何使用Rabbitmq玩出花来，与最佳实践等等内容不在本博客的范畴中。
但是理解一些最基础的概念对于了解Rabbitmq的消息路由转发流程来说是必要的。

在Rabbitmq中，消息发送方要指定消息的routing key
与exchange交换机。而接收方只要监听一个queue即可。而将视角转到最重要的exchange上，exchange会与queue进行绑定，同时会指定一个binding
key用于描述其与queue的关系。

最终消息该如何路由到queue上，取决于三个因素：消息的routing key，exchange的类型，binding key。
在流程上来说，先会根据exchange匹配。然后再根据具体的exchange类型定义的对 routing key和binding key的关系进行queue的匹配

不同的exchange定义了不同的匹配规则。

- Default 默认的exchange，由Rabbitmq broker自动创建，匹配规则时 routing key == queue's name(注意直接是queue的名字而不是binding
  key)
- Direct 严格匹配，routing key == binding key
- Topic 模糊匹配，routing key 与 binding key 之间可以使用通配符
- Fanout 广播，不需要routing key，直接将消息发送到所有绑定到该exchange的queue上
- Headers 类似于Topic，但是匹配规则是通过header来进行匹配的
- Dead Letter Exchange 用于处理死信，所有无法匹配、超时的消息都会被发送到这里

其中最简单的Default 与 Fanout 就类似于之前的JMS中的使用案例。 其他的exchange类型则是Rabbitmq的特色。
但是就如之前所说的那样，如何使用这些exchange类型，以及如何使用这些exchange类型来实现复杂的消息路由，不在本博客的范畴中,如何设计一个复杂的Rabbitmq也与Spring本身无关。本博客仅是一个how
to use 的教程。

---

### 使用Rabbitmq

要使用Rabbitmq，首先要在Spring中引入Rabbitmq的依赖,并在配置文件中对Rabbitmq的服务器地址进行配置,这部分就不赘述了

> 你将会发现使用Rabbitmq的时候，其与使用JMS几乎没有任何区别，只是在配置文件中将JMS相关的配置改为Rabbitmq相关的配置即可,以及注入的不是JmsTemplate，而是RabbitTemplate
 {: .prompt-tip}

- send 发送一个Message对象
- convertAndSend 直接发送一个对象，其会被自定义的消息转换器进行转换后发出

不同于JMS，你要指定的不是一个简单的Destination，而是一个routing key
与exchange名称。但剩下的几乎完全一致。可以使用send对消息进行构建，或者直接用conversend发送对象，或者搭配一个后处理器对message加点东西。

可以不指定routing key和exchange,此时使用的便是默认的routing key 或者交换机，其在没有配置的情况为空字符串，你可在项目配置文件中对其进行配置。
同样需要配置的是一个消息转换器，一般使用最流行的jackson。

---
接受消息也是与JMS相类似的，使用receive或者receiveAndConvert即可。

与JMS不同的地方在于其指定的是Queuename，而不是Destination。

> 注意，不同于JMS中的receive方法会造成阻塞，Rabbitmq中的receive方法默认是不会造成阻塞的，如果没有消息的话,其会立即返回一个null对象，
 {:.prompt-warning}

任意receive方法都可以指定一个timeout入参，默认为0表示不会进行阻塞等待。同时值得一提的是，receiveAndConvert方法还可以指定返回一个类型安全的对象，而不是一个Object，这是JMS所不具备的。

---
然后对于push模型的消息监听。你所需要做的事情就是把jms案例中的@JmsListener注解改为@RabbitListener即可。其余的一切都是一样的。（同样注解中的属性设置为queue
name）

## Apache Kafka

JMS和AMQP都是基于broker的消息中间件，而Kafka则是一个分布式的消息系统。其broker是一整个集群。Kafka没有复杂的消息路由功能，其只是简单的将消息发送到一个topic上，然后由订阅该topic的消费者进行消费。
但是Kafka的优势在于其高吞吐量，高并发，以及高可用性。具体对于Kafka的工作流程细节书中没有介绍，在本博文中也不会详细介绍。本博文仅仅是一个how
to use的教程。

同样进行依赖导入后，配置kafka地址(通常是多个地址代表集群),然后注入KafkaTemplate即可。

> 不同于JmsTemplate于RabbitmqTemplate，KafkaTemplate的方法定义是截然不同的。
 {: .prompt-info}

### 发送消息

首先，kafka的消息是以键值对的形式存在的，所以你需要指定一个key和一个value。
这意味着没有sendAndConvert这样的函数，因为我们只要指定泛型的KV即可，换句话说，所有的方法都会进行消息转换。

kafka只有send和sendDefault，后者将消息送到默认的topic上，而前者则需要指定topic。
send() 和 sendDefault() 的参数，它们与 JMS 和 Rabbit 中使用的参数完全不同。当使用 Kafka 发送消息时，可以指定以下参数来指导如何发送消息：

- 发送消息的 topic（send() 方法必要的参数）
- 写入 topic 的分区（可选）
- 发送记录的键（可选）
- 时间戳（可选；默认为 System.currentTimeMillis()）
- payload（必须）

topic 和 payload 是两个最重要的参数。分区和键对如何使用 KafkaTemplate 几乎没有影响，除了作为 send() 和 sendDefault()
的参数用于提供额外信息。

一般在使用时，在注入的使用指定Kafka泛型的类型后，send方法的形式是很简单的。
例如`kafkaTemplate.send("tacocloud.orders.topic", order);`
或者`kafkaTemplate.sendDefault(order);`

### 接受消息

同样完全不同于先前的JMS 和RabbitMq， KafkaTemplate 没有 receive() 方法。即接受消息只有push模型一种方式。

对于任意方法，使用`@KafkaListener(topics="tacocloud.orders.topic")`形式的注解，就会在接收到消息后调用该方法。
在方法中直接指定接受消息载荷的class即可，还可以添加一个入参 ConsumerRecord 或 Message 对象,用于获取消息的额外信息.

```java
@KafkaListener(topics="tacocloud.orders.topic")
public void handle(
    TacoOrder order, ConsumerRecord<String, TacoOrder> record) {
  log.info("Received from partition {} with timestamp {}",
        record.partition(), record.timestamp());

  ui.displayOrder(order);
}
```

```java
@KafkaListener(topics="tacocloud.orders.topic")
public void handle(Order order, Message<Order> message) {
  MessageHeaders headers = message.getHeaders();
  log.info("Received from partition {} with timestamp {}",
    headers.get(KafkaHeaders.RECEIVED_PARTITION_ID),
    headers.get(KafkaHeaders.RECEIVED_TIMESTAMP));
  ui.displayOrder(order);
}
```

或者可以只指定一个Message或者ConsumerRecord对象，不再入参中声明载荷而是用Message的方法获取载荷

## 总结
* 异步消息传递在通信的应用程序之间提供了一个间接层，允许更松散的耦合和更大的可伸缩性。
* Spring 支持 JMS、RabbitMQ 或 Apache Kafka 的异步消息传递。
* 应用程序可以使用基于模板的客户端（JmsTemplate、RabbitTemplate 或 KafkaTemplate）通过消息 broker 发送消息。
* 接收应用程序可以使用相同的基于模板的客户端，在基于 pull 的模型中使用消息。(Kafka不支持pull模型)
* 也可以通过向 bean 方法应用消息监听器注解（@JmsListener、@RabbitListener 或 @KafkaListener）将消息推送给消费者。
