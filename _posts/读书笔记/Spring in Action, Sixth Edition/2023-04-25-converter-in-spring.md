---
layout: post
title: "Spring中的转换器 To be continued"
date: 2023-04-25 22:29 +0800
category: [ 读书笔记, "spring实战第六版" ]
tag: [ Converter, 转换器 ]
---

顾名思义，转换器就是将A类型的对象转换为B类型的对象。
在Spring中，很多地方都会用到转换器，比如在SpringMVC中，将请求参数转换为Controller中的方法参数，将方法返回值转换为响应体,还有很多小地方。

(2023 05 08
更新，在使用消息队列中间件的时候，其也会自动使用spring中已经注册的消息转换器，将传递的原生JavaObject对象进行转换)
，当然最流行的做法也是使用
jackson将数据转为json格式

发生最频繁的是Json与java对象的转换,前后端往往会统一使用JSON格式的数据进行交互，这时候就需要将Java对象转换为JSON对象，或者将JSON对象转换为Java对象。
不过先不谈论这个复杂的转换器，而是从简单的转换器开始。

## 简单的转换器使用

在Spring中，转换器是一个接口，它有两个泛型参数，分别是源类型和目标类型。

在书中第二节，遇到了这样的场景是 html发送的表单中，包含了Taco的name和一个Ingredient的ID列表。处理器中将表单数据绑定为Taco对象进行处理。
虽然name属性可以和Taco对象中的name属性进行绑定，但是Ingredient的ID列表无法和Taco对象中的Ingredient列表进行绑定。
因为传递的Id只是一个String，而Ingredient是一个复杂的对象，
所以在书本的案例中，使用了一个自定义的转换器，来将请求携带的ID转换为java对象。

```java
@Component
public class IngredientByIdConverter implements Converter<String, Ingredient> {

  private Map<String, Ingredient> ingredientMap = new HashMap<>();

  public IngredientByIdConverter() {
    ingredientMap.put("FLTO", 
        new Ingredient("FLTO", "Flour Tortilla", Type.WRAP));
    ingredientMap.put("COTO", 
        new Ingredient("COTO", "Corn Tortilla", Type.WRAP));
    ingredientMap.put("GRBF", 
        new Ingredient("GRBF", "Ground Beef", Type.PROTEIN));
    ingredientMap.put("CARN", 
        new Ingredient("CARN", "Carnitas", Type.PROTEIN));
    ingredientMap.put("TMTO", 
        new Ingredient("TMTO", "Diced Tomatoes", Type.VEGGIES));
    ingredientMap.put("LETC", 
        new Ingredient("LETC", "Lettuce", Type.VEGGIES));
    ingredientMap.put("CHED", 
        new Ingredient("CHED", "Cheddar", Type.CHEESE));
    ingredientMap.put("JACK", 
        new Ingredient("JACK", "Monterrey Jack", Type.CHEESE));
    ingredientMap.put("SLSA", 
        new Ingredient("SLSA", "Salsa", Type.SAUCE));
    ingredientMap.put("SRCR", 
        new Ingredient("SRCR", "Sour Cream", Type.SAUCE));
  }

  @Override
  public Ingredient convert(String id) {
    return ingredientMap.get(id);
  }

}
```

convert操作的本质将就是根据ID从数据库中查询出对应的对象，然后返回。（此时还有没到数据库部分，所以手动维护了一个map）

---

此时一个很重要的问题是，这个转换器是如何被Spring管理的呢？
我们在Spring中注册了一个转换器，但是没有在需要使用的地方显式的使用这个转换器，Spring是如何知道我们需要使用这个转换器的呢？

其实Spring在启动的时候，会扫描所有的转换器，然后将其注册到一个转换器注册表中，当需要使用转换器的时候，就会从转换器注册表中获取对应的转换器。
获取的根据就是判定的类型是否匹配，如果匹配就会使用这个转换器。

具体来说，所有实现了Converter接口的组件，会在启动时统一注册到ConversionService中，
所有的convert操作统一仅由ConversionService来完成，其先会使用默认的转换器，然后再使用自定义的转换器。如果仍然无法转换，就会抛出异常。
如果存在多个转换器，就会使用优先级高的那一个，优先级的设定可以在注册转换器组件的使用，添加注解@Order来指定。值越小，优先级越高。

## 2023-5-8更新 在消息队列中间件中的使用

在书中的第9章中讲到了使用消息队列中间件对消息进行异步传输，其中的很多方法的入参支持直接传递Java对象，而不是消息中间件定义的Message等对象。
这也是因为消息中间件会使用Spring中注册的消息转换器将其进行隐式转化。

常见的Spring 消息转换器（全部在 org.springframework.jms.support.converter 包中）

- MappingJackson2MessageConverter - 使用 Jackson 2 JSON 库对消息进行与 JSON 的转换
- MarshallingMessageConverter - 使用 JAXB 对消息进行与 XML 的转换
- MessagingMessageConverter - 使用底层 MessageConverter（用于有效负载）和JmsHeaderMapper（用于将 Jms 信息头映射到标准消息标头）将
  Message 从消息传递抽象转换为 Message，并从 Message 转换为 Message
- SimpleMessageConverter - 将 String 转换为 TextMessage，将字节数组转换为 BytesMessage，将 Map 转换为
  MapMessage，将Serializable 转换为 ObjectMessage

SimpleMessageConverter 是默认的消息转换器，但是它要求发送的对象实现 Serializable 接口。这样要求可能还不错，但是可能更喜欢使用其他的消息转换器，如
MappingJackson2MessageConverter，来避免上述限制。

为了应用不同的消息转换器，需要做的是将选择的转换器声明为一个 bean。例如，下面这个 bean 声明将会使用
MappingJackson2MessageConverter 而不是 SimpleMessageConverter：

```java
@Bean
public MappingJackson2MessageConverter messageConverter() {
  MappingJackson2MessageConverter messageConverter = new MappingJackson2MessageConverter();
  messageConverter.setTypeIdPropertyName("_typeId");
  return messageConverter;
}
```

注意以上代码中的setTypeIdPropertyName方法，其指明了最终生成的json中，有一个属性叫做_typeId，其值为消息中的类型信息，这样在接收端就可以根据这个属性来确定消息的类型。

默认情况下，这个值的属性是，最终转换Object的全限定类名，但是这样做对于接收端来说，就需要知道发送端的全限定类名，这样就会造成耦合。
对于一些常用的Jdk中提供的类，我们当然没有这份烦恼，但对于一些自定义的属性，最好也自定义其自己的_typeId，这样接收端就可以根据这个_typeId来确定消息的类型。

例如设置如下消息转换器：

```java
@Bean
public MappingJackson2MessageConverter messageConverter() {
  MappingJackson2MessageConverter messageConverter = new MappingJackson2MessageConverter();
  messageConverter.setTypeIdPropertyName("_typeId");

  Map<String, Class<?>> typeIdMappings = new HashMap<String, Class<?>>();
  typeIdMappings.put("order", TacoOrder.class);
  messageConverter.setTypeIdMappings(typeIdMappings);

  return messageConverter;
  }
```

那么将一个TacoOrder转换为json后，其_typeId的值就是order，接收端就可以根据这个值来确定消息的类型。

这样做的意义在于，如何不设置，那么最终typeId的属性会是 xx.xx.xx.TacoOrder

但是消息接收方的TacoOrder不在这个包下，或者对方的TacoOrder不叫这个名字就叫Order，那么Spring就不能完成自动转换了

而发送方和接收方先统一好_typeId的值，那么接收方就可以简单的配置完成转换了。(接收方的Converter也加上上面的这段代码就行了)

