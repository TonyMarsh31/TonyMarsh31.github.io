---
layout: post
title: "Spring中的配置管理"
date: 2023-05-02 23:56 +0800
category: [ 读书笔记, "spring实战第六版" ]
tag: [ Spring,Spring中的配置文件 ]
---

> 书中，以及本博文不涉及XML的配置方式，仅涉及基于Java的配置方式。
{: .prompt-info}

一般在概念上存在两种形式的Configuration:  **BeanWriting** 和 **PropertyInjection** 。  
前者配置依赖关系,后者用于注入一些具体的配置属性。

BeanWriting自然是Spring配置的核心，毕竟一切都是Bean，但用现代的基于Java配置方式来完成BeanWriting是如此自然与简单,以至于我没有什么好说的。
你想要什么注入到Spring中，就用对应的注解即可(在config中用bean,简单对象用Component，还有MVC的注解等)
。至于使用，甚至不需要显式注解，直接在构造器中添加参数即可。(
当然还有autowired等方式，不过构造器已经是最佳实践所以这里不展开了)

所以本文没有BeanWriting的内容.而是关注其他宏观的配置。

## 调整Spring的自动配置

Spring帮我们自动完成了很多事情，例如H2数据库的使用，我们引入依赖后没多写任何一行关于H2的代码，就可以直接使用H2数据库了。但有时，我们需要对Spring自动完成的工作做一些调整，当然这不会涵盖所有Spring完成的自动配置，只是一些最最常用的配置。

### 理解Spring的环境抽象

首先需要明白的是，Spring是如何处理配置属性的。

简单来说，Spring对所有不同来源的配置属性做了一个统一的抽象,所有的配置属性:
包括项目路径中的applicant配置文件，jvm配置参数，操作系统环境变量，程序启动时的命令行参数等最终都会被集成抽象为The Spring
Environment Bean。然后在SpringBoot自动创建Bean的过程中,若有需要的属性便从其中获取。

由于这个特性，我们可以通过上述提到的多种方法修改Spring的配置属性,不过就目前的场景而言，最简单实用的方式就是在application.properties中添加配置信息。(
当然也支持yaml,后面均用yaml格式)

### 数据源

一般是

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost/tacocloud
    username: tacodb
    password: tacopassword
```

Spring使用的连接池是固定的几种，如果有必要，可以手动添加自定义的连接池配置。

同时，如果存在的话，Spring默认会运行项目路径中的schema.sql和data.sql文件，用于初始化数据库。如果要自定义修改

```yaml
spring:
  datasource:
    schema:
      - order-schema.sql
      - ingredient-schema.sql
      - taco-schema.sql
      - user-schema.sql
    data:
      - ingredients.sql
``` 

### 服务器

```yaml
server:
  port: 8443
  ssl:
    key-store: file:///path/to/mykeys.jks
    key-store-password: letmein
    key-password: letmein
```

### 日志

Spring默认用Logback作为日志框架，
如果有非常特殊的需求，例如自定义日志的信息格式，这需要在log.xml文件中进行配置。

但一些简单的配置，例如日志的级别，可以直接在application文件中进行配置。

```yaml
logging:
  level:
    root: WARN
    org.springframework.security: DEBUG
```

其中root是默认的日志级别，org.springframework.security是SpringSecurity的日志级别。

### 在配置文件中使用插值表达式来对属性进行引用

常见操作，一个属性的值引用另一个属性的值，来达到重用的效果。

```yaml
greeting:
  welcome: ${spring.application.name}
```

插值表达式是可以直接嵌入到其他字符串中的，并不会影响其他字符串的解析。

```yaml
greeting:
  welcome: You are using ${spring.application.name}
```

## 创建自定义的配置

前一节中我们只需要添加配置信息就可以让程序发生变化，但这是Spring自动配置的情况，如果我们需要自定义配置自己的类，那么就还需要做一些额外工作。

通常来说，如果需要注入多个属性值或者需要支持属性的嵌套，则可以使用`@ConfigurationProperties`
注解；如果只需要注入一个属性值，则可以使用`@Value`注解。

使用`@Value`注解很简单，直接在属性上添加注解即可,value设定为配置的属性名即可

```java
@Value("${MyConfig.property}")
String someProperty;
```

而如果有大量的属性需要注入，那么使用`@ConfigurationProperties`注解会更加方便。
直接标记在类上，然后指定配置文件的前缀，然后所有的名称可以被匹配的字段属性都会被注入。
如果有代码重用的需求，一个推荐的做法是将配置属性封装到一个pojo中作为一个property对象，然后在需要的地方注入property使用其get方法获取属性值即可，这样只要维护一个pojo即可。

值得一提的是，你可以直接为pojo在代码中使用硬编码进行赋值，最终配置文件中的值会覆盖掉硬编码的值，而若没有配置则会使用硬编码的值。

```java
@Component
@ConfigurationProperties(prefix="taco.orders")
@Data
public class OrderProps {
  private int properity1 = 20;
  private int properity2 = 30;
  private int properity3 = 40;
  private int properity4 = 50;
}
```

## 使用profile来管理配置多个配置文件

### 创建多个配置文件profile

#### 经典做法:
创建新的 application-{profile 名称}.yml 或 application-{profile 名称}.properties  
例如 application-prod.yml , application-test.yml , application-dev.yml

#### 仅支持yaml的做法:
在同一文件中用三个连字符分隔:
```yaml
logging:
  level:
    tacos: DEBUG

---
spring:
  profiles: prod
  
  datasource:
    url: jdbc:mysql://localhost/tacocloud
    username: tacouser
    password: tacopassword

logging:
  level:
    tacos: WARN
```
最开始的是默认配置，后面的是prod配置，当profile为prod时，后面的配置会覆盖前面的配置。

### 激活使用

一种流行的做法是直接在application.yml中添加spring.profiles.active属性
```yaml
spring:
  profiles:
    active: prod
```
但这种做法非常不推荐。还记得之前[Spring配置环境抽象](#理解spring的环境抽象)吗，其中提到的Spring将多个来源的环境集中抽象在一起，所以我们可以有多种方式来修改配置属性.

我们要配置的还是spring.profiles.active属性，但不是在applicant.yml中，而是在jvm启动参数中,或者直接在计算机的环境变量中！
```bash
% java -jar taco-cloud.jar --spring.profiles.active=prod
```
```bash
% export SPRING_PROFILES_ACTIVE=prod
```


### 对profile有条件的Bean的配置

需求:只有当特定profile激活时，该Bean才会被创建。

实现：使用@Profile注解

```java
@Profile("dev") // 只有当dev profile激活时，该Bean才会被创建
@Profile({"dev", "qa"}) // dev 或 qa
@Profile("!prod") // 非prod
```
