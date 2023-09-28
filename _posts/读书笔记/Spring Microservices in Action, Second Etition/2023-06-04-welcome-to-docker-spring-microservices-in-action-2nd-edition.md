---
layout: post
title: "Docker在微服务架构中的使用"
date: 2023-06-04 14:47 +0800
category: [ 读书笔记, "Spring微服务实战 第二版" ]
tag: [ MicroServices, 微服务, Docker ]
---

在继续接下来的微服务开发历程之间，我们需要先考虑微服务应用的移植性(Portability) ,即如何将微服务应用在不同的环境下运行.

在近些年(2019年),容器的概念流行了起来，从一个锦上添花的技术到了必不可少的技术。
容器是一种 轻量便捷与有效的手段来将 任何软件 迁移并运行到另一个环境中(例如从开发者的机器上到另一个物理或是虚拟机上).
通过容器化技术提供的更小且适应性更好的容器替代传统VM模型可以为我们的微服务提供更好的速度、可移植性与可拓展性等优势

本博文为原书第四章节的精要概括，其简要介绍了Docker的基本概念与使用方法，以及如何将Docker集成到微服务中。

## 在VM中的容器

在许多公司，VM仍然是软件部署时的事实标准。本节主要讲述VM(virtual machines 虚拟机)与容器(containers)的区别。

VM是一种虚拟化技术，它可以在一台物理机上运行多个虚拟机，每个虚拟机都有自己的操作系统，可以运行不同的应用程序。虚拟机是一种完全隔离的环境，可以在同一台物理机上运行不同的操作系统。虚拟机的缺点是它们需要大量的资源来运行，包括内存、CPU和磁盘空间。
另一方面，容器仅是在操作系统级别之上的部分进行虚拟化的，即容器之间共享操作系统内核，因此比虚拟机更轻量级。容器的优点是它们可以在更少的资源上运行，因此可以在更少的资源上运行更多的容器。

![虚拟机与容器](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/虚拟机与容器.6tdf3gxqyjw0.webp)

依据上述的图， 乍一看没有什么区别，只是Hypervisor层换成了容器引擎，同时每个容器中不包含独立的操作系统。
但实际上，这些区别所带来的影响是巨大的.

在虚拟机中，我们必须事前设定分配给虚拟机多少物理资源，例如，每个虚拟机需要多少个虚拟处理器与多少GB的内容与磁盘空间。对于这些值的定义
往往是一个棘手的问题，需要注意以下几点：

- 处理器是可以在不同虚拟机之间共享的
- 为虚拟机设定的磁盘空间为最大值，而不是实际使用的值
- 为虚拟机设定的内存是确定与独占的，不同虚拟机之间不能共享

而使用容器化技术，我们可以使用Kubernetes来设置我们需要为容器分配的内存与CPU。
但不同于VM,这一步骤不是必须的，缺省情况下，容器引擎会自动为容器分配资源，使其正常执行。
最重要的是，因为容器可以重用底层的操作系统，所以容器对于物理机器的资源消耗要比虚拟机少得多，
从而带来轻量级的特性：更快的启动时间、更少的资源消耗、更好的可移植性与更好的可拓展性。

不过最终而言，具体使用什么方案还是看具体的需求。如要在操作系统上完全隔离的需求，那么虚拟机是最好的选择。

由于我们需要使用云框架，所以选择了容器化技术,这带来的好处有：

- 容器可以在任意环境下运行，有利于开发与实施，增加了可移植性
- 容器的启动与关闭速度更快，有利于快速部署与拓展
- 容器可以主动安排资源，有利于提高资源利用率与可拓展性
- 通过容器，可以在最少的服务器上运行最大数量的微服务，有利于降低成本

## 什么是Docker

Docker是一个基于Linux的开源容器引擎。负责启动与管理容器。
这项技术是我们能够让不同的容器共享单一的物理资源，而不是VM那样重新暴露不同的物理资源。

Docker的核心引擎是一个遵循客户端-服务器架构的程序。该程序包含了三个核心组件：
服务器、客户端与镜像仓库。

![Docker架构](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/Docker架构.234iziiml8kg.webp)

具体来说，Docker的组件如下

- 守护进程  
  一个长期运行的进程，负责管理Docker对象，例如镜像、容器、网络与卷等。守护进程可以通过socket或者REST API与客户端进行通信。
- 客户端  
  一个命令行工具，可以通过socket或者REST API与守护进程进行通信。
- Registry  
  一个集中的存储库，用于保存镜像。Docker Hub是一个公共的Registry，可以免费使用。同时，我们也可以在本地搭建一个私有的Registry。
- 镜像  
  一个只读的模板，用于创建容器。例如，一个镜像可以包含一个基本的操作系统，里面安装了一个Web服务器。我们可以使用这个镜像来创建一个或多个容器。
  镜像可以通过Dockerfile来创建，也可以通过Registry来获取。
- 容器  
  一个容器是一个运行中的镜像。我们可以使用Docker API或者CLI来创建、启动、停止、移动或删除一个容器。我们可以把容器看作是一个简易版的虚拟机。
  容器之间是相互隔离的，每个容器都有自己的文件系统、网络、进程空间等。容器的好处是它们可以共享操作系统的内核，因此比虚拟机更轻量级。
- volumes  
  一个volume是一个可选的持久化存储区域，可以在容器之间共享。Docker可以在主机上创建一个volume，然后将其挂载到容器中。这样，容器就可以在volume中读写数据了。
- 网络  
  我们可以为容器创建一个网络，使其可以与其他容器进行通信。Docker提供了多种网络驱动，可以满足不同的需求。(
  桥接，host，overlay、macvlan等)

再一次附上Docker的架构图，以便理解

![Docker架构2](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/Docker架构2.4kgd0v7xoqw0.webp)

## Dockerfiles

Dockerfile是一个文本文件，包含了一条条的指令用于描述创建docker镜像的过程。
通过Dockerfile描述的自动化流程，我们可以创建自定义的镜像。
Dockerfile中的指令就类似于Linux中的shell脚本，因此不是很难理解。

![Dockerfile使用](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring微服务实战第二版/Dockerfile使用.6gj15yrcgok0.webp)

以下是一个简单的Dockerfile示例：

```dockerfile
FROM openjdk:11-slim
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

- FROM  
  指定了基础镜像，这里使用了openjdk:11-slim，这是一个基于Debian的镜像，包含了OpenJDK 11。
- ARG  
  指定了一个参数，这里是JAR_FILE。这个参数可以在构建镜像的时候自动传入，
  手动执行的效果例如：docker build --build-arg JAR_FILE=build/libs/*.jar
- COPY  
  指定了将本地文件复制到镜像中的路径。这里是将本地的jar文件复制到镜像中的/app.jar路径。
  除了本地文件，也可以指定一个URL，Docker会自动下载并复制到镜像中。
- ENTRYPOINT  
  指定了容器启动时执行的命令。这里是执行java -jar /app.jar命令。

还有一些常用命令补充如下：

- LABEL  
  为镜像添加元数据，例如作者、版本、描述等。
- ENV  
  设置环境变量。这些变量可以在Dockerfile中使用，也可以在容器中使用。
- VOLUME  
  创建一个挂载点，用于存储数据或者与其他容器共享数据。不过大部分情况下为了隔离特性，我们不会使用这个指令，而是使用默认的匿名卷。
- RUN  
  在镜像中执行命令。例如，我们可以在镜像中安装软件包，或者执行一些编译操作。
- ADD  
  类似于COPY，不过它可以自动解压缩本地文件或者下载并解压缩远程文件。
- CMD  
  指定容器启动时执行的命令。与ENTRYPOINT不同的是，CMD指定的命令可以被docker run命令行参数中指定的命令覆盖。
  一般来说，CMD用于定义默认的启动命令，可以被覆写，而ENTRYPOINT用于定义哪些固定的不会变的命令。

## Docker Compose

DockerFile 用于创建镜像，对应的，Docker Compose用于创建与管理具体的容器。
Docker Compose使用YAML文件来定义应用程序的服务，然后使用docker-compose命令来启动、停止、删除应用程序。

使用DockerCompose的步骤如下：

- 安装Docker Compose(新版本的Docker可能已经默认安装了Docker Compose)
- 创建一个docker-compose.yml文件定义应用程序的服务
- (可选) 在yaml文件的路径下，使用 `docker-compose config` 命令检查配置文件是否正确
- 在yaml文件的路径下，使用 `docker-compose up` 命令启动应用程序

Docker Compose的yaml文件示例就不贴了，可以很简单，也可以很复杂，具体可以参考书中的示例。

常用配置项如下：

- version  
  Docker Compose的版本
- service  
  服务，可以有多个服务，每个服务都可以指定镜像、端口、环境变量、挂载点等。
  一次启动多个服务是DockerCompose的重要特性，这样可以很方便的启动多个服务，而不需要手动启动多个容器。
- image  
  指定容器的镜像，可以指定镜像名称，也可以指定Dockerfile的路径，DockerCompose会自动构建镜像。
- port  
  指定端口映射，可以指定主机端口和容器端口，也可以只指定容器端口，这样DockerCompose会自动分配主机端口。
- environment  
  指定环境变量，可以指定多个环境变量。
- network  
  指定网络，可以指定已有的网络，也可以指定新的网络。

此外再补充一些 DockerCompose的命令行参数：

- up -d  
  启动容器，-d参数表示后台运行。
- logs  
  查看容器日志。
- ps  
  查看容器状态。
- stop  
  停止容器。
- down  
  停止并删除容器。

其中，命令之后可以具体指定某个服务，例如 `docker-compose logs service1`。否则会对所有服务执行命令。

## 集成Docker到微服务中

这一部分的内容可能有时效性，推荐直接查询当下的Spring提供的文档。
在写这篇博文的时候，Spring提供了[Spring Boot Docker指南](https://spring.io/guides/topicals/spring-boot-docker/)

--- 

接下来作者将要将之前创建的一个微服务节点示例 进行Docker化。

作者在pom中添加了一个`spotify的dockerfile-maven-plugin`插件用于构建与管理镜像，
接着继续在该插件的pom中配置一些属性例如镜像名称、镜像标签、Dockerfile路径等。
最终在命令行中直接通过一个`mvn clean package dockerfile:build`即可一键完成项目的构建和镜像的构建。

不过在Dockerfile的编写上，我们有一些选择

### Basic Dockerfile

在该Dockerfile中，我们将定义直接将SpringBoot的jar包复制到镜像中，然后执行java -jar命令启动服务。

```dockerfile
#Start with a base image containing Java runtime
FROM openjdk:11-slim

# Add Maintainer Info
LABEL maintainer="Illary Huaylupo <illaryhs@gmail.com>"

# The application's jar file
ARG JAR_FILE

# Add the application's jar to the container
COPY ${JAR_FILE} app.jar

#execute the application
ENTRYPOINT ["java","-jar","/app.jar"]
```

基本上很直白，唯一需要说明的是 `JAR_FILE` 是在pom中定义的一个属性。这中做法是可选的，也可以直接在Dockerfile中硬编码

这是一个很直接的，基础的Dockerfile，但是我们还可以使用多阶段构造的做法来优化镜像的构建。

### Multistage build Dockerfile 多阶段构建

宏观来说，多阶段构建就是将build的过程进行拆分，对其进行更细程度的控制。
多阶段构建是一个抽象的概念，意思就是构建完一个镜像之后还没完，而是继续在这个镜像的基础上进行更多的操作来构建下一层镜像
具体在做法上可以很丰富,是一个不断迭代的过程，一般最终的目的都是为了尽量减少镜像的体积。

作者在这里的做法就是不将 jar包直接 一股脑地全部复制到镜像中，
而是(在基础jar包镜像的基础上进一步)将jar包解压，然后将必要的文件复制到镜像中，最终镜像中只包含了必要的文件。

```dockerfile
#stage 1
#Start with a base image containing Java runtime
FROM openjdk:11-slim as build

# Add Maintainer Info
LABEL maintainer="Illary Huaylupo <illaryhs@gmail.com>"

# The application's jar file
ARG JAR_FILE

# Add the application's jar to the container
COPY ${JAR_FILE} app.jar

#unpackage jar file
RUN mkdir -p target/dependency && (cd target/dependency; jar -xf /app.jar)

#stage 2
#Same Java runtime
FROM openjdk:11-slim

#Add volume pointing to /tmp
VOLUME /tmp

#Copy unpackaged application to new container
ARG DEPENDENCY=/target/dependency
COPY --from=build ${DEPENDENCY}/BOOT-INF/lib /app/lib
COPY --from=build ${DEPENDENCY}/META-INF /app/META-INF
COPY --from=build ${DEPENDENCY}/BOOT-INF/classes /app

#execute the application
ENTRYPOINT ["java","-cp","app:app/lib/*","com.optimagrowth.license.LicenseServiceApplication"]
```

在上面的dockerfile中我们实际上创建了两个镜像(其中一个作为多阶段构造的临时镜像)，
这可以从两个 `FROM` 指令看出来。多个`FROM`的情况下只有最终的`FROM`指令创建的镜像才是最终的镜像，其余的是临时镜像。
第一个镜像中我们使用了 `as build` 来指定了这个镜像的别名，这样我们就可以在后面的指令中使用这个别名来指定这个镜像了。
然后注意在第二阶段中，我们在`COPY`指令中使用了 `--from=build` 来指定了使用上一个阶段的镜像中的文件来进行复制操作。

具体只COPY了哪些文件，可以通过 `jar -tf app.jar` 来查看jar包中的文件结构。
或者参考之前的列出的[Spring Boot Docker指南](https://spring.io/guides/topicals/spring-boot-docker/)中的说明。

### 使用SpringBoot简化流程镜像构建流程

SpringBoot新版特性中提供了快捷构建应用镜像的功能。 这需要SpringBoot的版本够高，同时安装了DockerCompose等，这里不赘述了

#### Buildpacks

Buildpacks是一种将应用程序打包成容器镜像的工具，它可以将应用程序的源代码转换为可运行的容器镜像。
SpringBoot2.3.0之后引入了对Buildpacks的支持，可以通过 `mvn spring-boot:build-image` 来构建镜像。

运行指令后，SpringBoot会自动检测项目中的依赖，然后根据依赖自动选择合适的Buildpacks来构建镜像。
如果想要对镜像做一些自定义配置，可以直接在`spring-boot-maven-plugin`
插件下添加config标签，然后进一步配置image的名称、标签、镜像仓库等。

#### Layered Jars

Spring Boot引入了一种新的JAR布局，称为分层JARs。在这种格式中，/lib
和/classes文件夹被分割开来，并归入不同的层。
分层的依据是其有多大的可能性在应用程序的生命周期内发生变化。

Layered Jars是除了Buildpacks之外的另一个非常好的选项，
使用步骤如下：

- 添加依赖
- 打包应用
- 用layertools插件解压jar包
- 用Dockerfile构建镜像

layertools不是一个单独的依赖，而是SpringBoot的一个插件，在pom中的插件配置中打开即可。
然后先执行 `mvn clean package`打包，再在项目根目录下执行`java -Djarmode=layertools -jar target/{the project}.jar list`
来解压jar包。
执行后的命令行的输出会像这样：

```shell
dependencies
spring-boot-loader
snapshot-dependencies
application
```

现在有了layer信息，就可以编写Dockerfile来构建镜像了

```dockerfile
FROM openjdk:11-slim as build
WORKDIR application
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} application.jar
RUN java -Djarmode=layertools -jar application.jar extract

FROM openjdk:11-slim
WORKDIR application
COPY --from=build application/dependencies/ ./
COPY --from=build application/spring-boot-loader/ ./
COPY --from=build application/snapshot-dependencies/ ./
COPY --from=build application/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

`COPY`指令中使用之间命令行输出的layer信息来指定复制的文件，这样就可以将jar包中的文件分层复制到镜像中了。
然后最终使用 `org.springframework.boot.loader.JarLauncher` 来启动应用。

注意layered jar的方式仅提供layer信息，我们仍然需要自己编写Dockerfile，然后手动执行docker build来构建镜像。
layered jar仅帮我们将jar包中的文件分层，然后我们可以根据layer信息来指定复制哪些文件到镜像中。

最终，再次说明，这些信息可能会随着SpringBoot版本的更新而变化，具体可以参考官方文档。
在写这篇博文的当下为： [Spring Boot Docker指南](https://spring.io/guides/topicals/spring-boot-docker/)

### 使用DockerCompose启动服务

这一部分(在原书中的内容)太简单了，没啥好讲的

在Yaml中定义容器的配置，然后执行 `docker-compose up` 启动即可

## 总结

- 容器化可以让程序运行在任何环境下(从开发者的机器到其他物理或虚拟机上)
- VM能让我们在一台计算机上模拟另一台计算机，这是基于hypervisor来完全模型新的计算机硬件，需要手动预先配置虚拟机的各项
  虚拟化硬件的参数，例如CPU，内存，磁盘等
- 容器是另一种虚拟化的方案，运行在host机上创建一个个完全隔离与独立的环境来运行应用程序。
- 相较于VM，容器的优势在于更轻量级，更快速地启动与执行，所以可以减少项目的开销
- Docker是一个基于Linux的流行容器引擎，现已成为容器化的标准解决方案。
- Docker的主要组件有：DockerEngines，clients，registries，images，containers，networks，volumes
- DockerFile是一个简单的包含了一系列指令的文本文件，用来构建Docker镜像
- DockerCompose是一个用来定义和运行多容器Docker应用的工具
- Dockerfile Maven插件集成了Maven和Docker，同时使用SpringBoot也可以提供快速构建镜像的功能




