---
layout: post
title: "使用SpringCloud配置中心控制配置"
date: 2023-06-04 19:25 +0800
category: [ 读书笔记, "Spring微服务实战 第二版" ]
tag: [ Spring, SpringCloud, MicroServices, 微服务, 配置中心 ]
---

本博文为原书第五章的读书笔记。

> 书中对于 如何 让ConfigServer结合Vault使用的部分讲的非常不清楚，Spring是有一个专门用于结合Vault使用的项目 Spring Cloud
> Vault 的.但作者完全没有提及。
> 总之关于如何secure config的部分作者只给出了对称加密的方案，使用Vault的方案是不完整的
 {:.prompt-info}

软件开发者需要时刻铭记于心的一个重要准则就是将程序代码与配置信息进行分离。
在绝大多数情况下，这意味着不要将配置信息硬编码到程序代码中.
如果不这么做，将会导致在每次进行配置更改时都需要重新编译和部署应用程序。

但将配置与代码完全分离后，这一行为也增加了新的复杂度，即现在我们多了一个工件需(纯配置信息)要处理了.
如何管理与部署这些配置信息呢？大多开发者倾向于使用配置文件(YAML,JSON,XML等)来管理配置信息。
这并不是一件难事，但当面对基于云的微服务架构中的数不胜数的服务节点时，这种方式就不再适用了。
因为即使完全分配了配置文件，可配置文件本身还是和代码绑定在一起的，配置修改的需求还是难以满足。

最好的解决方案是将配置信息存储在一个外部的中心化位置，同时依据一下原则进行管理：

- 完全分离代码与配置信息，即代码中不存在任何硬编码的配置信息。
- 构建好的程序镜像 需要是不可变的
- 当程序启动时，通过环境变量或外部微服务配置的中央存储库 将配置信息注入到程序中。

## 关于管理配置信息(和复杂度)

微服务实例需要以最少的人为干预行为进行快速启动。
而若需要人为对配置信息进行管理，则会直接导致微服务实例的启动时间变长，且带来潜在的配置漂移问题。

在关于如何管理配置信息这一点上，有一下几个原则可以参考：

- Segregate 隔离性  
  我们需要将配置信息与具体的服务的部署组件进行隔离。事实上，配置文件从根本上就不应该包含在项目的部署组件中，而是应该从环境变量与外部配置中心进行注入。
- Abstract 抽象性  
  需要对配置信息的访问进行抽象。不直接读取具体的配置信息，而是通过一个抽象层进行访问(例如一个REST API服务)
  。这样可以在不改变代码的情况下，更换配置信息的来源。
- Centralize 集中化  
  配置信息应该集中存储在一个(或尽量少的)中心化的位置，而不是分散在各个服务实例中。这样可以更好的管理配置信息，同时也可以更好的保护敏感的配置信息。
- Harden 加固  
  其实也就是要保证 配置信息服务接口需要是高可用与冗余的。

最终需要强调的一点:
配置信息需要被追踪与版本控制,因为管理不善的应用程序配置将会是滋生难以检测的错误和计划外中断的肥沃土壤

### 配置管理的大致架构

这张图微服务生命周期的图片在之前的博文有过提及，这里再次贴出来，用于说明配置管理的大致架构。

![微服务的生命周期](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/微服务的生命周期.3rj7blscha60.webp)

下面是一张更加具体的 bootstrapping 流程图，可以关注之前提到的四个原则是如何在这一过程中体现的.

![配置管理的概念架构](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/配置管理的概念架构.5huibeyq22s0.webp)

上图中包含了4个环节，以下是其具体阐述：

1. 当一个微服务实例出现时，其首先对调用配置信息服务获取 特定于其环境的配置信息。
   然后，具体的配置信息服务会将配置信息从中心化的配置信息存储库中获取到，并将其返回给微服务实例。
2. 实际的配置信息被存储在一个中心化的配置信息存储库中。这个存储库可以是一个简单的文件系统，也可以是一个更加复杂的数据库。
3. 对于配置信息的管理是独立于 微服务实例本身的部署环节的。 配置信息通过 自己的部署管道进行部署到不同的环境中，并且其需要有自己的版本控制
4. 当配置信息发送变更时，使用旧配置的服务需要被通知到，然后重新加载配置信息。

以上就是微服务配置管理的大致架构了，接着来看我们有那些解决方案供我们选择。

### 实现方案的一些选择

- etcd
  - 描述: 用Go编写。用于服务发现和键值管理。使用Raft协议进行分布式计算模型。
  - 特点: 非常快速和可扩展。可分布部署。基于命令行驱动。易于使用和设置。

- Eureka
  - 描述: 由Netflix开发。久经考验。即可以用于服务发现，也可以用于键值对的管理。
  - 特点: 分布式键值存储。灵活但需要一些功夫来进行设置。提供了开箱即用的动态客户端刷新。

- Consul
  - 描述: 由HashiCorp开发。类似于etcd和Eureka，但使用了不同的分布式计算模型算法。
  - 特点: 快速。提供了原生的服务发现，并可选择直接与DNS集成。但默认情况下不提供客户端动态刷新功能。

- Zookeeper
  - 描述: Apache项目。提供分布式锁定功能。通常用作访问键值数据的配置管理解决方案。
  - 特点: 最古老、经过最多测试的解决方案。使用最复杂。可以用于配置管理，但是仅在其他部分中已经使用Zookeeper的情况下才推荐使用。

- Spring Cloud配置服务器
  - 描述: 开源项目。提供通用的配置管理解决方案，支持不同的后端存储。
  - 特点: 非分布式的键值存储。与Spring和非Spring服务紧密集成。可以使用多个后端存储来存储配置数据，包括共享文件系统、Eureka、Consul或Git。

上述所有方案都可以方便地完成配置管理的工作。作者使用了Spring Cloud Config Server，原因如下

- 简单
- 与Spring Boot紧密集成，用简单的注解就可以读取配置
- 可以使用多种后端存储来存储配置数据
- 可以直接与git 与 HashiCorp Vault 集成, 这一部分内容在后面的章节中会有详细的介绍

在之后的内容中，将会演示3种提供配置信息的方式 - 本地文件系统、Git仓库和HashiCorp Vault。
然后具体将其与书本中的一个Licensing微服务节点进行集成。

## 构建Spring Cloud Config Server的简单示例

SpringCloudConfigServer(之后简称配置中心)是基于SpringBoot的REST API服务。
配置中心是可以直接嵌入一个已存在的SpringBoot应用程序中的，虽然独立部署是非必要的，但在最佳实践中一般还是会单独创建一个配置中心的应用程序。

创建一个SpringBoot项目的流程就不赘述了，注意加入ConfigServer的依赖即可。
当然，最终部署会以容器形式部署，所以可以添加Docker以及Actuator等依赖，因为不是重点,这里就不具体展开了

接着是对配置中心的配置文件的定义。
配置文件一般会有两个，一个是application.yml，另一个是bootstrap.yml。(properties文件也可以)
两者的区别在名称中就可以看出来，bootstrap.yml是在应用程序启动时就会加载的配置文件其先于application.yml加载。

配置文件中目前没有什么需要配置的
只需要配置应用名称与监听的端口号即可，前者将用于服务发现，后者用于提供服务。

### 设置配置中心的启动(引导)类

在SpringBoot的启动类上添加`@EnableConfigServer`注解即可

### 通过本地文件系统提供配置信息

配置如下

```yaml
spring:
  application:
    name: config-server
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations: classpath:/config
          # 也可以是绝对路径 file:///D:/config , 并提供多个路径

server:
  port: 8071
```

使用本地文件系统提供配置信息的关键属性是 `search-locations`
其值可以是一个绝对路径，也可以是一个相对路径. 也可以提供多个路径，用逗号分隔(或者用yaml语法格式的数组)

使用本地文件系统提供配置信息的做法是最简单的，但不推荐在中大型项目中使用，
更推荐的做法是Git与HashiCorp Vault提供配置信息,这将在后面的章节中进行介绍，
接下来先看怎么具体定义要提供给不同微服务的配置信息

### 设置具体提供给微服务的配置信息

微服务有很多个，用于区分的就是其名称，所以配置文件的名称就是微服务的名称。
同时，每一个微服务都有自己的不同的环境，所以还需要定义不同的环境，比如开发环境、测试环境、生产环境等等。

![image](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/image.fo0nv9099vk.webp)

> 更新：上面的图有误导性，当访问/dev的时候，不会有pord的配置文件，即三个箭头是分别由上至下的，不是从中间的dev同时指向三个配置文件
> {: .prompt-warning}

读取顺序是：服务名加profile→服务名不加profile→application
然后将多个配置文件中的属性合并，如果有重复配置属性，则保留先读取到的配置属性

即读取 `/licensing-service/dev`时 ,或者不用/dev 而是直接指定最终要的文件名最为url也行,例如`/licensing-service-dev.yml`

其查找顺序是 `/licensing-service-dev.yml` → `/licensing-service.yml` → `/application.yml`
然后将多个配置文件中的属性合并，如果有重复配置属性，则保留先读取到的配置属性

(至少在作者当下这个版本，属性合并的工作应该是由客户端完成的，当你直接call config server的REST
API的时候，其返回的JSON中属性是未合并的)

## 在SpringBoot客户端中集成Spring Cloud Config Server

服务端中已经可以提供配置信息了，接下来就是在客户端中集成配置中心。

首先需要config的客户端依赖：`spring-cloud-starter-config`

接着所有要做的工作仅仅是在客户端的配置文件中添加一些属性即可

```yaml
spring:
  application:
    name: licensing-service
  profiles:
    active: dev
  cloud:
    config:
      uri: http://configserver:8071
```

配置传递的流程如下：

![image](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/image.5yrli79hpu8.webp)

### 在命令行中指定参数来覆写配置信息

还有一个常用的做法是在项目启动时，通过命令行参数传递配置信息，例如传递profile.active参数为prod等来方便地覆写配置信息
如果使用Docker，那么在docker-compose.yaml中可以执行类似的操作，即添加`environment`属性

### 在项目中，使用@ConfigrationProperties注解来注入配置信息

在SpringBoot中，可以使用`@ConfigurationProperties`注解来注入配置信息，这样就可以在代码中使用了
一般执行注解的prefix属性来限定获取的配置信息范围。

不过更推荐的一个做法还是使用单独的一个Bean来包裹配置信息，然后在需要的地方注入这个Bean即可，使用其get方法获取配置。

## Spring Cloud Config Server的其他特性

### 刷新配置信息

当配置属性发生变化时，配置中心如何动态刷新客户端的配置状态呢？

SpringBoot应用程序只会在启动时加载一次配置信息，所以当配置信息发生变化时，其不会自动对配置信息进行更新。
重启应用程序是一种解决办法，不过SpringBootActuator提供了一个`/refresh`
端点来强制让应用重新刷新配置信息，只需要在配置文件中添加`management.endpoints.refresh.enabled=true`即可
,或者在启动类上添加`@RefreshScope`注解也可以。

但注意，SpringActuator的`/refresh`端点只能刷新应用程序中的自定义属性，而不能刷新SpringBoot的内置属性，比如server.port与数据库连接等

另外一些做法是例如 使用Spring Cloud
Bus来推送配置信息的变化，搭配消息中间件RabbitMQ将配置信息的变化推送给所有的客户端.

不过，其实最简单直接的重启容器的方法的花费其实不是很大，如果我们符合设计原则，那么一个容器应该是随机可抛弃重启的，其启动速度也应该是很快的，所以重启容器的代价其实不是很大，而且也不会影响到用户的使用，所以这种做法也是可以接受的。

### 使用Git来存储配置信息

配置如下

```yaml
spring:
  application:
    name: config-server
  profiles:
    active:
      - native, git
  cloud:
    config:
      server:
        native:
          search-locations: classpath:/config
        git:
          uri: https://github.com/ihuaylupo/config.git
          searchPaths: licensingservice
server:
  port: 8071
```

如果github需要验证，那么可以在配置文件中添加用户名与密码

> 后续的关于对config信息进行加密的内容，书中写的不是很好，仅能作为思路参考，具体的实现上讲的不是很清楚，推荐直接参考SpringCloudConfigServer对应版本的官方文档
 {: .prompt-info}

### 使用HashiCorp Vault来存储配置信息

使用Vault，我们可以安全地存储敏感的配置信息，比如数据库的用户名与密码等

Vault作为一个独立的外部组件，需要单独启动,作者直接用docker来快速创建一个Vault的实例,这里就不详细介绍了

我们可以使用使用CLI来访问与使用Vault，但Vault也提供了其UI界面，可以通过`服务地址:8200/ui/valut/auth`来访问
登录可以使用创建Vault时的root token,然后就是很直接地操作UI界面了。

具体不展开，主要步骤是先创建KV的engine，然后创建secret即可。

Vault中的配置信息准备好了，接下来就是让ConfigServer连上Vault了，配置如下

```yaml
spring:
  application:
    name: config-server
  profiles:
    active:
      - vault
  cloud:
    config:
      server:
        vault:
          port: 8200
          host: ...
          kvVersion: 2 # KV engine version
server:
  port: 8071
```

## 保护敏感的配置信息

默认情况下ConfigServer直接以明文存储与传输配置信息，这是不安全的，使用对称与非对称加密算法来保护配置信息是一个不错的选择
非对称加密的安全性更高，但对称性加密更加简单，因为其只需要一个秘钥即可.

### 设置加密秘钥

可以在ConfigServer的bootstrap.yaml中`encrypt.key:`定义加密秘钥，或者在启动时通过命令行参数来指定，例如`--encrypt.key=xxxx`
后者是更好的做法。

### 加密与解密配置信息

当配置`encrypt.key`后，ConfigServer会自动添加两个新的端点`/encrypt`与`/decrypt`，分别用于加密与解密配置信息

对`/encrypt`节点进行post请求，传入需要加密的配置信息，即可得到加密后的配置信息

如果需要解密配置信息，那么可以使用`/decrypt`节点，传入加密后的配置信息，即可得到解密后的配置信息

最终的做法就是，在本地文件系统中存储加密后的配置信息，然后添加一个{cipher}前缀，这样ConfigServer就会自动解密配置信息了

加密过程是手动的还是有点难搞，不过一般需要加密的配置信息也不多。

## 总结

整个内容有点虎头蛇尾，因为我略过了很多作者对于书中那个license项目的是实现。也没想到作者最终没具体展开
如何加密ConfigServer，感觉这一点可以展开很多，但是作者仅仅介绍了一个加密算法的概念，然后就没有然后了……

- 使用ConfigServer可以将配置信息集中管理，方便统一修改与管理
- 可以用 file-based, Git-based, or Vault-based的方法来存储配置信息
- Vault-based的方法作者没有说的很明白，推荐直接看官方文档，好像要用到SpringCloud的Vault组件来访问Value获取内容，这一部分是作者完全没有说明的地方
- 可以使用加密方法，来避免配置中直接出现敏感信息，比如数据库的用户名与密码等
