---
layout: post
title: "通过JMX监控Spring"
date: 2023-05-24 00:41 +0800
category: [ 读书笔记, "spring实战第六版" ]
tag: [ Spring, Spring Boot, Spring Boot Actuator, 服务监控, JMX ]
---

JMX - Java Management Extensions 是十多年来管理和监控 Java 应用程序的标准方式。

在之前的博文中，我们使用了Spring Boot Actuator来暴露了许多开箱即用的监控端点，这些端点既可以通过HTTP暴露，也可以作为JMXBean暴露。
然而，在之前的博文中，我们只介绍了通过HTTP方式访问这些端点，而没有涉及如何使用JMX方式访问这些端点。

简单来说，JMX是一种与HTTP不同的获取应用程序信息的方式，它是一种基于Java的管理和监控的标准方法。
您可以使用JDK自带的JConsole或VisualVM等工具来访问JMX端点，也可以使用其他第三方的JMX客户端来进行访问。

无论是使用HTTP还是JMX，在最终的使用目的上都是为了获取应用程序的信息，只是获取方式上存在差异而已。

## Actuator端点的JMX暴露

Actuator的端点默认暴露为HTTP端点与JMX端点，关于Actuator不需要做额外的配置工作。  
但是,Spring程序本身是默认不会暴露JMX端点的，所以需要对Spring程序进行配置。

```yaml
spring:
  jmx:
  enabled: true
 ```

默认情况下，所有的Actuator端点都会暴露为JMX端点，但是如果你想要关闭某个端点的JMX暴露，可以通过配置`management.endpoints.jmx.exposure.exclude`
来实现。

```yaml
management:
  endpoints:
    jmx:
      exposure:
        exclude: env,metrics
        # 或者搭配 include使用
```

--- 

可以使用JDK自带的`jconsole`对JMX端点进行访问(若配置了jdk环境，直接在终端中输入`jconsole`即可打开)，也可以使用第三方的JMX客户端进行访问。

具体的jconsole消费JMX的使用细节就不赘述了。 play around with it.

## 创建自定义的 JMX MBeans

之前提到过可以自定义Actuator的端点，其会自动同时暴露为HTTP端点与JMX端点。所以你可以通过自定义Actuator端点来实现自定义的JMX端点。

但也可以使用JMX的规范来创建原生的JMX端点。步骤如下：

- 使用`@ManagedResource`注解来标注一个类，这个类就是一个MBean
- 使用`@ManagedAttribute`注解来标注一个方法，这个方法就是一个MBean的属性
- 使用`@ManagedOperation`注解来标注一个方法，这个方法就是一个MBean的操作

```java
@Service
@ManagedResource
public class TacoCounter extends AbstractRepositoryEventListener<Taco> {

  private AtomicLong counter;
  public TacoCounter(TacoRepository tacoRepo) {
    tacoRepo
        .count()
        .subscribe(initialCount -> {
          this.counter = new AtomicLong(initialCount);
        });
  }

  @Override
  protected void onAfterCreate(Taco entity) {
    counter.incrementAndGet();
  }

  @ManagedAttribute
  public long getTacoCount() {
    return counter.get();
  }

  @ManagedOperation
  public long increment(long delta) {
    return counter.addAndGet(delta);
  }
}
```

上述代码中，我们创建了一个`TacoCounter`类，它是一个MBean，  
它有一个`getTacoCount()`方法，它是一个MBean的属性，（严格来说 `getTacoCount`
会是一个Operation，然后依据JMX规范自动解析，去掉方法名中的get后，`tacoCount`为Attribute)  
它有一个`increment()`方法，它是一个MBean的操作。

虽然和JMX无关，但是该类继承了`AbstractRepositoryEventListener`，它是一个Spring Data的类，它可以监听Spring
Data的事件，这里监听了`Taco`的创建事件，每当创建一个`Taco`时，就会调用`onAfterCreate()`
方法，该方法会调用`counter.incrementAndGet()`方法，从而实现了对`Taco`的计数。

在JConsole的页面会看到如下的MBean信息：

![JConsole](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/JConsole.mi6f87zgq68.webp)

## 通过MBeans发送通知

虽然经过上述操作后，已经可以通过JMX来获取应用程序的信息了，但是这种方式是被动的，需要我们手动去获取信息。
如果我们想要实现让程序主动推送消息到client，就需要使用到JMX的通知机制。

在这里就不赘述JMX具体的通知规范与机制了，Spring封装了底层的细节，我们只需要实现其提供的`NotificationPublisherAware`
接口,使用Spring提供的`NotificationPublisher`来发送通知即可。

通知的信息是非常自由的。例如书中的案例为，想要没生产100个Taco就发送一次通知，为了实现这个功能，还需要依赖(之前也提到过的)
SpringData提供的`AbstractRepositoryEventListener<T>`监听Repository的操作。

综上代码如下：

```java
@Service
@ManagedResource
public class TacoCounter
        extends AbstractRepositoryEventListener<Taco>
        implements NotificationPublisherAware {

  private AtomicLong counter;
  private NotificationPublisher np;

  @Override
  public void setNotificationPublisher(NotificationPublisher np) {
    this.np = np;
  }

  ...

  @ManagedOperation
  public long increment(long delta) {
    long before = counter.get();
    long after = counter.addAndGet(delta);
    if ((after / 100) > (before / 100)) {
      Notification notification = new Notification(
              "taco.count", this,
              before, after + "th taco created!");
      np.sendNotification(notification);
    }
    return after;
  }
}
```

作者省略了 初始化`counter`的代码，并推送的代码，这里也不赘述了。以上主要展示的是一个JMX操作，可以手动地增加`counter`
的值，每当`counter`的值增加100时，就会发送一个通知。

## 总结

* 大多数 Actuator 端点都可以直接暴露为 MBean，从而使用任何 JMX 客户端进行访问。
* 通过在 bean 上添加 @ManagedResource 注解，可以将其公开为 MBean。它们的方法和属性，可以通过使用 @ManagedOperation 和
  @ManagedAttribute 进行公开。
* 可以使用 NotificationPublisher 将通知发布到已订阅的 JMX 客户端。

（感觉不如SpringBootAdmin…）

