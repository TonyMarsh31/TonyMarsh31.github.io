---
layout: post
title: "构建与部署SpringBoot应用程序"
date: 2023-05-24 17:41 +0800
category: [ 读书笔记, "spring实战第六版" ]
tag: [ Spring, Spring Boot, Deploying, 部署 ]
---

> 书中简单地谈了谈Kubernetes的操作，但本博文没有记录。 本博文将会非常简单记录构建与项目部署的简单步骤，不涉及运维内容。所以会很简短。
{: .prompt-info}

直接在IDE中，或者在终端中使用Maven/Gradle 工具启动项目是很方便的，但是这种方式并不适合生产环境。本章将介绍如何将Spring
Boot应用程序部署到生产环境中。

## 构建

- 构建可执行的jar文件
- 构建可执行的war文件
- 构建为Docker镜像

如果创建项目的时候在IDE中选择了项目为 jar ，那么应该很方便地可以通过命令行 mvn package 或者直接在ide中点点按钮进行build

---

如果要将jar项目，重新构建为war项目，那么需要修改pom.xml文件，将packaging标签的值改为war，然后在IDE中build即可。（这也许可以，但这是我自己预测了，书中有一些额外步骤，但考虑到时间问题，现代的IDE或新版本的Maven应该可以很简单地完成这种工作）

---

build为docker镜像可以走 docker的模式 即写一个Dockerfile文件，然后在终端中执行 `docker build -t <镜像名>` 即可。

但通过SpringBoot可以简化这一流程，直接运行`mvnw spring-boot:build-image` 或者 `gradlew bootBuildImage`
之类的命令，其会根据pom或gradle文件中的配置，自动构建为docker镜像。
但其生产的image名称可能是带library前缀了，作者补充了一些命令行配置，来自定义image名称。

总之知道能这么做就行了，这些具体的做法没有必要记住，其信息也没有时效性，随时可以查阅。

## 部署

- 将jar文件部署到云平台(PaaS)
- 将war文件部署到 不同的应用服务器
- 将Docker镜像部署到Kubernetes集群

其实作者也没有很详细地介绍，只是简单地提了一下。

部署到Paas，使用服务商提供的命令行工具 push 自己的jar包即可，一般只要一条命令或点点鼠标。简单化就是选择PaaS的意义，

部署war到服务器，作者直接省略了，可以更具不同的服务器例如tomcat或netty查询官方文档，同样因为这类信息没有时效性，可以随时查阅。

部署到Kubernetes集群，作者有做介绍，但是我懒得学了…………

## 总结

这算是某种意义上的虎头蛇尾了，总之这一章的内容很简单，但是我也不想花时间去学习Kubernetes，所以就这样吧。
