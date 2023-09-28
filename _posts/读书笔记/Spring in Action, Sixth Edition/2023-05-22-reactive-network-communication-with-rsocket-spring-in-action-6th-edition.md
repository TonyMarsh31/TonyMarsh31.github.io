---
layout: post
title: "响应式的异步网络消息传递协议 - RSocket的简单介绍与在Spring中的使用示例"
date: 2023-05-22 19:52 +0800
category: [ 读书笔记, "spring实战第六版" ]
tag: [ 响应式编程,响应式的网络请求方法,Spring,RSocket,WebSocket ]
---

> 一些网络概念的补充：
>
> RSocket、WebSocket、HTTP均为应用层的协议，三者都基于传输层的TCP协议。
>
> HTTP是无状态的请求/响应协议。Client发送请求，Server进行响应，然后链接关闭。  
> Server端无法主动向Client端发送消息。
>
> WebSocket允许Server端主动向Client端发送消息的协议。是全双工的。  
> 因为WebSocket协议的建立往往需要先用HTTP通信发送一次协议升级请求，然后再建立WebSocket连接。所以WebSocket中有使用到HTTP协议,但是
> "WebSocket基于HTTP"
>
这句话是不准确的。Again,WebSocket和HTTP都是应用层的协议，两者都基于传输层的TCP协议的。只是WebSocket的建立需要先用HTTP通信发送一次协议升级请求，然后再建立WebSocket连接.
> 同时HTTP使用文本或二进制格式的数据进行通信，WebSocket使用Frame(帧)格式的数据进行通信,其更适用于实时通信场景。
>
> RSocket同样也是基于TCP协议的一个单独的应用层协议。本博文就是关于RSocket的。  
> 但是RSocket是可以基于WebSocket协议的。也就说RSocket可以引入WebSocket依赖库，在WebSocket之上使用。这是RSocket的一个特性。
>
因为在一些场景中，例如客户端是用js写的，运行在浏览器中，不能方便地或者无法使用RSocket的客户端库。有或者有些时候客户端需要跨网关或者防火墙才能访问RSocket服务。这种情况下就可以让RSocket运行在WebSocket之上，最终让客户端可以使用WebSocket链接来访问RSocket服务。
{: .prompt-info}

## What

RSocket是一种 应用层上的 异步消息传递协议(R就是响应式单词的首字母缩写，Socket就是套接字)
，允许客户端与服务器之间以异步消息的方式传递消息。其一般作为HTTP通信协议的替代，但是RSocket本身也可以依赖WebSocket作为通道来兼容HTTP请求。

(下段内容由ChatGPT生成)

WebSocket通过在客户端和服务器之间建立持久的双向连接，允许双方在连接打开后随时发送消息。类似于电话线路，一旦建立连接，双方可以通过该连接进行实时的双向通信。WebSocket适用于需要实时交互和即时通信的应用场景，例如在线聊天应用或实时协作工具。

RSocket则是一种应用层协议，它提供了更灵活的通信模型。RSocket使用了Reactive
Streams的概念，可以支持请求-响应、请求-流、流-请求和流-流等多种通信模式。RSocket还具有强大的流控制能力和异步处理机制，适用于高吞吐量、低延迟和可靠性要求较高的应用场景，如分布式系统的服务间通信或边缘计算。

下面是一个形象的例子来说明两者的区别：

想象你是一名游戏玩家，正在玩一个在线多人游戏。使用WebSocket作为通信协议时，你可以与其他玩家建立一个实时的语音聊天通道。你们可以同时听到彼此的声音，就像你们都在同一个房间里一样，可以即时交流。这是WebSocket提供的持久双向连接的通信方式。

现在假设游戏需要更高的性能和更复杂的通信模式。这时候RSocket就派上用场了。RSocket提供了更多的灵活性，就像你和其他玩家之间有了更多的沟通方式。你可以向其他玩家发送请求，例如请求获得游戏中的某个物品的信息，其他玩家可以及时响应你的请求。你也可以订阅其他玩家的游戏动态，实时地接收他们的状态更新。RSocket的异步处理和流控制能力可以确保通信的高吞吐量和低延迟，使你的游戏体验更加流畅。

---

- 总之RSocket最主要的特性就是支持多种响应式的异步消息传递模型，  
  包括请求/响应、请求/流、即发即忘和流/流。

## RSocket 的四种消息传递模型

再回顾一下，RSocket是一种二进制的异步消息传递协议，其是基于响应式流的。换句话说，RSocket提供了应用之间的异步通信机制，以支持我们之前博文中讲解的响应式编程模型(
如Mono、Flux)

作为HTTP通信协议的替代，RSocket更加灵活，提供了四种不同的通行模型

- 请求/响应
- 请求/流
- 阅后即焚/即发即忘(请求/无响应)
- 通道(流/流)

![响应](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/Rsocket模型-请求/响应.3nkbyfd9tva0.webp)
请求/响应模型 模仿了典型的HTTP的请求响应模型，客户端发送一个请求，服务器返回一个响应。

不过虽然其在形式上与HTTP的请求/响应模型相似，但是要注意RSocket本质上是非阻塞与异步的，其传递的消息是基于响应式流模型的。即尽管客户端需要等待服务器的响应，但这一过程不会阻塞客户端的线程，而是通过回调的方式来处理响应。

![Rsocket模型-Stream](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/Rsocket模型-Stream.34fa476jx5m0.webp)
请求/流模型 与请求/响应模型类似，但是服务器返回的是一个响应流，而不是单个的响应。

这种模型适用于需要返回多个响应的场景，例如客户端需要获取一个数据集合。(这里的数据集合可以是无限的，也可以是有限的)
一些具体的代码示例将在后续的博文中给出。(查询实时的股票价格)

![Rsocket模型-阅后即焚](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/Rsocket模型-阅后即焚.ox0uo8pxzy8.webp)
阅后即焚/即发即忘(请求/无响应)模型 与请求/响应模型类似，但是服务器不会返回任何响应。

![Rsocket模型-通道](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/Rsocket模型-通道.4q4oa959sgm0.webp)
最后，RSocket最强大与灵活的模型是通道模型，其允许客户端和服务器之间建立一个持久的双向连接，客户端与服务器之间都可以向对方发送响应式消息流。

## 一些简单的使用示例

Spring为RSocket提供了非常好的支持，要使用RSocket，我们需要添加`spring-boot-starter-rsocket`依赖，其会自动配置RSocket服务器和客户端。

具体来说，若在配置信息中，指定了RSocket监听的端口，则Spring会将程序自动配置RSocket服务器。若没有指定RSocket监听的端口，则会自动配置为RSocket客户端。

```yaml
spring:
  rsocket:
    server:
      port: 7777
```

---

创建一个RSocket服务 非常类似于创建一个 REST服务，只是用到的注解不一样

- 我们仍可以使用`@Controller`注解来标注一个服务接口
- 但是我们需要使用`@MessageMapping`注解,而不是`@RequestMapping`来标注一个服务方法
- 以及，使用`@DestinationVariable`,而不是`@PathVariable`来标注方法参数

---

创建一个RSocket客户端 的流程如下

- 使用依赖组件中的`RSocketRequester.Builder`来创建一个`RSocketRequester`实例作为客户端
- 一般使用Builder的tcp方法来创建一个TCP连接，参数指定url和port，这一步可以在配置Bean中完成，然后在需要的地方注入即可
- Builder还可以会使用websocket方法创建一个基于Websocket的链接，关于这一点在最后一小节单独讲解
- 之后的一些具体用法会在后面的代码示例中给出

接下来是一些具体的代码示例

### 请求/响应 模型

```java
// Server
@Controller
public class GreetingController{
  @MessageMapping("greeting/{name}")
  public Mono<String> handleGreeting( @DestinationVariable("name") String name, Mono<String> greetingMono) {
    return greetingMono 
      .doOnNext(greeting -> log.info("Received a greeting from " + name + " : " + greeting))
      .map(greeting -> "Hello to you, too, " + name);
  }
}

----
// Client
RSocketRequester tcp = requesterBuilder.tcp("localhost", 7000);
...
String who = "Craig";
tcp
  .route("greeting/{name}", who)
  .data("Hello RSocket!")
  .retrieveMono(String.class)
  .subscribe(response -> log.info("Got a response: " + response));
```

一个简单的请求/响应模型的示例，客户端发送一个请求，服务器返回一个响应。

客户端代码中，使用tcp链接对象后用route方法指定了请求的路由，data方法指定了请求的数据，retrieveMono方法指定了响应的类型（Mono<String>
），最后使用subscribe方法来订阅响应。

### 请求/流 模型

```java
@Controller
public class StockQuoteController {

  @MessageMapping("stock/{symbol}")
  public Flux<StockQuote> getStockPrice( @DestinationVariable("symbol") String symbol) {
    return Flux
      .interval(Duration.ofSeconds(1))
      .map(i -> { 
        BigDecimal price = BigDecimal.valueOf(Math.random() * 10);
        return new StockQuote(symbol, price, Instant.now());
    });
  }
}

----

String stockSymbol = "XYZ";

RSocketRequester tcp = requesterBuilder.tcp("localhost", 7000);
tcp
  .route("stock/{symbol}", stockSymbol)
  .retrieveFlux(StockQuote.class)
  .doOnNext(stockQuote -> {
    log.info(
        "Price of " + stockQuote.getSymbol() +
        " : " + stockQuote.getPrice() +
        " (at " + stockQuote.getTimestamp() + ")");
  })
  .subscribe();
```

一个简单的请求/流模型的示例，客户端发送一个请求，服务器返回一个响应流。

服务端中，Flux中每秒生产一个StockQuote对象，其中包含了随机生成的股票的价格，时间戳等信息。
Client中，使用tcp链接对象后用route方法指定了请求的路由，retrieveFlux方法指定了响应的类型（Flux<StockQuote>
），最后使用subscribe方法来订阅响应。

### 即发即忘 模型

想象一下，您在一艘刚刚受到敌舰攻击的星际飞船上。您发出全舰范围的“红色警报”，以便所有人员都处于战斗模式。您不需要等待飞船计算机的响应来确认警报发送状态。在这种情况下，您也没有时间等待阅读任何回应。您设置了警报，然后继续执行更多关键操作。这是一个
即发即忘 的例子。考虑到目前的情况，虽然您可能不会忘记您处于红色警戒状态。对您来说，应对这场战争危机比处理警报响应显然更重要。

```java
@Data
@AllArgsConstructor
public class Alert {

  private Level level;
  private String orderedBy;
  private Instant orderedAt;

  public static enum Level {
    YELLOW, ORANGE, RED, BLACK
  }
}

@Controller
@Slf4j
public class AlertController {

  @MessageMapping("alert")
  public Mono<Void> setAlert(Mono<Alert> alertMono) {
    return alertMono
        .doOnNext(alert -> {
          log.info(alert.getLevel() + " alert"
          + " ordered by " + alert.getOrderedBy()
          + " at " + alert.getOrderedAt());
        })
        .thenEmpty(Mono.empty());
  }
}

---

RSocketRequester tcp = requesterBuilder.tcp("localhost", 7000);
tcp
  .route("alert")
  .data(new Alert( Alert.Level.RED, "Craig", Instant.now()))
  .send()
  .subscribe();
log.info("Alert sent");
```

即Client向Server发送一个Alert对象，Server不需要返回任何响应，Client也不需要等待响应，这就是即发即忘模型。

### 通道 模型

为了演示如何在 Spring 中使用 RSocket 通道通信，让我们创建一个计算账单上的小费、接收一个 Flux 请求并以 Flux 响应的服务。

```java
@Data
@AllArgsConstructor
public class GratuityIn {

  private BigDecimal billTotal;
  private int percent;

}

@Data
@AllArgsConstructor
public class GratuityOut {

  private BigDecimal billTotal;
  private int percent;
  private BigDecimal gratuity;

}

---

@Controller
@Slf4j
public class GratuityController {

  // 就是做小费的计算工作
  @MessageMapping("gratuity")
  public Flux<GratuityOut> calculate(Flux<GratuityIn> gratuityInFlux) {
    return gratuityInFlux
        .doOnNext(in -> log.info("Calculating gratuity: " + in))
        .map(in -> {
          double percentAsDecimal = in.getPercent() / 100.0;
          BigDecimal gratuity = in.getBillTotal() .multiply(BigDecimal.valueOf(percentAsDecimal));
          return new GratuityOut(in.getBillTotal(), in.getPercent(), gratuity);
    });
  }

}

---

RSocketRequester tcp = requesterBuilder.tcp("localhost", 7000);

Flux<GratuityIn> gratuityInFlux =
    Flux.fromArray(new GratuityIn[] {
        new GratuityIn(new BigDecimal(35.50), 18),
        new GratuityIn(new BigDecimal(10.00), 15),
        new GratuityIn(new BigDecimal(23.25), 20),
        new GratuityIn(new BigDecimal(52.75), 18),
        new GratuityIn(new BigDecimal(80.00), 15)
    })
    .delayElements(Duration.ofSeconds(1));

    tcp
      .route("gratuity")
      .data(gratuityInFlux)
      .retrieveFlux(GratuityOut.class)
      .subscribe(out ->
        log.info(out.getPercent() + "% gratuity on "
            + out.getBillTotal() + " is "
            + out.getGratuity()));

```

至此，以上就是所有的RSocket的消息模型了，下面是一个总结表格：

| 通信模式 | Handler参数类型 | Handler返回类型 |
| ------------ | -------------- | -------------- |
| 请求/响应 | Mono | Mono |
| 请求/流 | Mono | Flux |
| 即发即忘 | Mono | Mono |
| 通道 | Flux | Flux |

> 你可能会好奇，是否存在一种模型是Client发送Flux，然后Server返回Mono。虽然顺着上面的这一表格可以推导出这种形式的存在，但是其实并不存在这种模型。
> 因为这种模型的存在并没有意义, 没有实际的应用场景符合这样的模型。
{: .prompt-tip}

## 补充，RSocket的WebSocket支持

如本博文最开始所说，有时候我们需要在浏览器中使用RSocket，或者由于跨网关或防火墙的限制，我们需要使用WebSocket来进行RSocket通信。这时候就需要使用RSocket的WebSocket支持了。

RSocket的数据流即可以通过TCP传输，也可以通过WebSocket传输。要让RSocket支持WebSocket，需要在服务端和客户端做一些修改。

---

在服务端，由于WebSocket通道的建立需要一次HTTP请求，所以Server也需要支持HTTP请求，所以需要引入`spring-boot-starter-webflux`
的依赖。
同时需要修改RSocket的配置，显式声明其使用WebSocket通道作为传输通道，同时指定WebSocket的映射路径。

```yaml
spring:
  rsocket:
  server:
  transport: websocket
  mapping-path: /rsocket
```

不像TCP，WebSocket不基于端口，而是基于HTTP路径。所以在客户端这里就不用配置端口了，只需要配置路径即可。

这就是服务端所需要做的全部工作了。

---
在客户端，其实只要修改一下RSocketRequester的创建方式即可。

```java
RSocketRequester requester = requesterBuilder.websocket(URI.create("ws://localhost:8080/rsocket"));

 requester
 .route("greeting")
 .data("Hello RSocket!")
 .retrieveMono(String.class)
 .subscribe(response -> log.info("Got a response: " + response));
```

使用websocket() 而不是tcp() 来创建RSocketRequester。同时注意url的前缀为ws即可。

## 总结

* RSocket是一种异步二进制协议，提供四种通信模型：请求/响应、请求/流、即发即忘 以及 通道。
* Spring 通过带 @MessageHandler 注解的控制器处理方法，在服务端支持 RSocket。
* 通过 RSocketRequester 来创建客户端的 RSocket 通信。
* 在服务端与客户端中，Spring 的 RSocket都支持通过响应式的 Flux 和 Mono 支持完全响应式通信。
* 默认情况下，RSocket 通信通过 TCP 进行，但也可以通过 WebSocket，以避开防火墙限制和支持浏览器客户端。

