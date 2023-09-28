---
layout: post
date: 2023-03-18 02:26 +0800
title: "SSH连接局域网内的windows电脑的踩坑记录"
category: 随笔
tag: [SSH, windows,踩坑]
---
windows SSH踩坑记录

心血来潮想用ssh连接windows，结果就是连不上，一直报错，最终还是找到了解决办法，记录一下。   
## 无法连接，没有任何提示，只有timeout  

饶了一点弯路 ，最后从防火墙方向入手，其实就是server不理这个请求，被防火墙ban了，加了个入站22端口的规则就行。

## 无法连接，请求直接被reset

这一步是个大坑，我也没搞明白是什么原因  

最终的解决方式是我把我scoop install的OpenSSH删了，然后换成了windows system setting -> optional feature ->  sshserver sshclient解决了

## 小坑

当用Microsoft账号登录windows后，你ssh登录的username和password就也得是Microsoft账号的名称(一般为注册邮箱)和密码。这是我一开始不知道的，以为自己密码忘了不断尝试了很久

## 告一段落，但是自己作出了新坑

我想把ssh登录后的default shell给换了，但是误操作结果又搞得ssh连不上了  

最后发现是参考[windows官方文档](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_server_configuration)加入这个语句的时候，Value路径填错了

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -PropertyType String -Force
```

我自己从Microsoft Store下载的Powershell不在这个路径，而是user/local里很偏的地方，命名也很奇怪，
anyway,就是这个参数的问题，但是ssh报错没有这么具体，所以又浪费了一点时间

## 反思

最终回顾一下，其实都不是什么大问题，如果直接按照网络上流行的规范操作的话就不会遇到这些奇怪的问题
不知道为什么一开始用的是scoop 安装的ssh，好像是很久以前上linux课程时用到的  
我需要改正的地方是dont fuck around in the first place, 以及遇到问题不断重复相同类型的操作是很傻的(我在上面的那个小坑里浪费了漫长时间做无效的试错)，应该去思考问题出现在了什么环节然后做排除法，

![ssh-success](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Miscellaneous/ssh-success.5aoj6tyujx80.webp)
