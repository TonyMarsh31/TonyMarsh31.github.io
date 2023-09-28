---
layout: post
title: "从SpringBoot实战这本书中学到的内容"
date: 2023-05-25 17:19 +0800
category: [ 读书笔记, "SpringBoot实战" ]
tag: [ SpringBoot ]
---

> 作为一本InAction的书，作者除了SpringBoot的部分，还完成了介绍了整个开发流程的步骤与其他一些工具，例如SpringBootCli、Groovy、TestInSpringBoot、Grails、deploy等内容，
> 但是因为这是一本将近7年前的书了，我个人觉得这些内容已经不再具有时效性，所以这里只记录一些关于SpringBoot的核心内容。
{: .prompt-info}

## What - SpringBoot精要

简单来说，SpringBoot完成了项目的(接近所有的)配置工作,boot后让开发者可以直接进行业务逻辑的开发。

SpringBoot最大的特性就是下述两个

- 起步依赖(spring-boot-starter)
- 自动配置

starter依赖就是一系列的依赖集合，本质上就是将更细粒度的依赖组件进行了打包。Spring官方对其预先进行了测试确保了内部无冲突可开箱即用。
最经典的例子就是web starter依赖，其包含了web开发所需要的所有依赖，引入后就可以直接进行web开发了。
相较于一个个的引入每一个具体的依赖，如SpringMVC，Tomcat等,直接引入starter的行为本质上就是把 软件工程中对于依赖处理的 复杂度
转移给了Spring.

---

自动配置，其本质上是 对所有配置属性集成的抽象环境进行一系列的条件判断以筛选满足条件的配置bean注入到Spring容器中。

缺省情况下，起步依赖中所有预先配置好的Bean注入Spring容器，形成了开箱即用的效果。
但是每个项目都有自定义需求，例如精调starter的配置属性，或者直接用自定义的配置Bean替换掉starter中的配置Bean。
自动配置使得这一过程变得非常简单，开发者只要做声明的工作，然后SpringBoot会做剩下的所有工作。

---

总的来说，SpringBoot主要完成了以下两件事，转移了很大一部分的 不属于程序开发核心流程的复杂度

- starter dependency , 提供值得信赖的，预打包的依赖
- configuration as code, 对于配置只需要做属性声明，具体的配置工作交给SpringBoot

## boot starter

如果你用的是 现代IDE 与 Maven构建工具，那么可以执行以下操作来很快把握starter的概念

- 进入一个SpringBoot项目的pom.xml文件中
- 找到任意starter依赖，在IDE中一般 通过window/commend + 鼠标左键点击 可以直接进入stater依赖的内部
- 接着你就会发现，starter本身就是另一个pom.xml，其中定义了一系列的依赖，这就是starter的本质，一个预先打包好的依赖集合

---

starter在绝大多数情况下能很好的完成任务，但如有需要，开发者也可以对starter动刀微调。
一种常用的做法是(在Maven中)使用exclude标签来排除starter中的一些依赖，来达到"瘦身"的目的。或者在排除后使用自定义的依赖来替换starter中的依赖。

对于后者，一般可以直接在项目顶层pom中直接添加新依赖来达到同样的效果，但是这部分内容没有时效性，好的做法是直接参考对应版本的官方文档，这里仅做一个提醒：你可以对starter进行微调。

## how boot starter works exactly

所以，只是写了这些抽象的pom标签，最终是如何让我的SpringBoot程序一个配置也没有就能直接运行起来的呢？

具体来说，首先Maven会做他的工作，将所有的依赖下载到本地仓库。
然后SpringBoot会在程序启动时，扫描ClassPath，然后根据一系列的条件判断，将满足条件的Bean注入到Spring容器中。(这就是auto
configuration)

> ClassPath是是Java虚拟机（JVM）用于查找类和资源文件的路径。它是一个包含目录和JAR文件路径的集合，JVM会在这些路径中搜索需要加载的类和资源。
> 具体来说，你写的所有代码都会被编译成class文件，然后被打包成jar文件，这些文件都会被放在ClassPath中，然后JVM会在ClassPath中搜索需要加载的类和资源。
>
> 补充一些其他常见的Path概念: 最常接触的就是 相对和绝对路径， 然后一个项目路径project path一般是在IDE语境下的项目根模具。
> SourcePath为源代码目录，一般在Maven中是以 src/main/java 为顶层目录所有的(项目自身的)源代码都在这个目录下。
> 还有测试目录与build目录等，IDEA都用不用颜色进行了标识。
{: .prompt-tip}


如果你使用的现代IDE 例如IntelliJIDEA， 在maven下载依赖后你应该可以直接找到所有具体的依赖代码。一般是在project栏中最下方的External
Libraries中。
很重要的一点是,虽然具体细节可能有差异，这些Libraries所在的地方在概念上也是ClasPath,所以SpringBoot可以扫描到这些所有的代码，然后根据一系列的条件判断，将满足条件的Bean注入到Spring容器中。

那么要注入的资源在这里，但具体Spring注入的步骤是什么？

继续在这些Library中查看，使用查找功能，可以具体找到SpringBootAutoConfiguration依赖，浏览其中的代码，你就会发现这些就是
将配置自动注入的逻辑实现。
这里不深入其具体实现，但是可以看到的就是说过一些注解的使用，例如@ConditionalOnClass，@ConditionalOnMissingBean等等，这些注解就是用来做条件判断的。

简单来说，在SpringBoot之前，这些代码都是要开发者自己手写的，但其中很多都是重复的，所以SpringBoot作为解决方案被开发出来了。

## auto configuration

这里不讨论 how auto configuration work, 只是补充一些怎么使用

并且在之前的博文 [Spring中的配置管理]({% post_url /读书笔记/Spring in Action, Sixth Edition/2023-05-02-working-with-configuration-properties-spring-in-action-6th-edition %}) 
中有了详细的介绍，所以这里仅作为一些补充.

- 使用`Condition`
  为自定义的配置类添加条件判断，来达到只有在满足条件的情况下才会注入到Spring容器中。具体的条件判断可以参考`Condition`
  的子类，例如`OnClassCondition`，`OnBeanCondition`等等。
- 如果Condition很复杂，可以在配置类中实现`Condition`接口，然后在`matches`方法中实现自定义的条件判断逻辑。

基本上就是这些，还有很多内容，就像之前说的，已经在SpringInAction对应的博文中有了详细的介绍，这里就不再赘述了。



