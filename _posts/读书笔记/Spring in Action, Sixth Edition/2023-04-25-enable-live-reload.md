---
layout: post
title: "使用devtools在Spring开发中实现及时预览"
date: 2023-04-25 19:36 +0800
category: [ 读书笔记, "spring实战第六版" ]
tag: [ Spring热部署, Spring-LiveReload ]
---

顾名思义，DevTools 为 Spring 开发人员提供了一些方便的开发同步工具。其中包含：

- 当代码更改时自动重启应用程序
- 当以浏览器为目标的资源（如模板、JavaScript、样式表等）发生变化时，浏览器会自动刷新
- 自动禁用模板缓存
- 如果 H2 数据库正在使用，则在 H2 控制台中构建

一般来说，使用SpringInitializer生成的DevTools的pom依赖是这样的

```xml

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-devtools</artifactId>
  <scope>runtime</scope>
  <optional>true</optional>
</dependency>
```
Scope 设置为运行时，optional 设置为 true，这意味着它不会被传递到依赖项中且不会被打包到最终的 jar 中。这是因为 DevTools
仅用于开发目的，因此在部署生产环境时禁用它本身是非常明智的。
此外DevTools不是IDE 插件，即DevTools不要求特定的 IDE

>参考自 [StackOverflow](https://stackoverflow.com/questions/69449905/how-to-enable-spring-boot-live-dev-tools-on-intellij-2021-2-to-rebuild-classes-a)
{: .prompt-tip }
## 在idea中使用DevTools所需要的额外配置

1. Settings -> Build, Execution, Deployment -> Compiler: Check "Build project automatically"
2. Settings -> Advanced Settings: Check "Allow auto-make to start even if developed application is currently running"

## 使用liveReload功能所需要的额外工作
(Chrome浏览器)下载[LiveReload拓展程序](https://chrome.google.com/webstore/detail/livereload/jnihajbhpnppcggbcgedagnkighmdlei)
当启动项目后，鼠标悬浮在LiveReload图标上，应该显示"LiveReload is connected, click to disable Has access to this site".

>最终的LiveReload效果不像前端开发流程中的那么快，一般要等1-2秒左右
{: .prompt-tip }
