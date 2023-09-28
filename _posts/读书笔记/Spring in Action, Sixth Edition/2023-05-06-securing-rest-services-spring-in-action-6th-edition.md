---
layout: post
title: "在Spring中保护REST服务 - OAuth2"
date: 2023-05-06 01:17 +0800
category: [ 读书笔记, "spring实战第六版" ]
tag: [ Spring,保护REST服务,REST ]
---

在完成REST Service的创建之后，一个很重要的问题就是如何保护REST Service,
在分布式应用程序中，软件系统之间的信任至关重要。 即使只是一个简单的客户端请求，服务端也需要验证客户端是否是可信的，以将任何其他未授权的客户端请求拒绝。
例如，虽然任何人都可以发送一个DELETE请求给REST服务，但是应该只有对应权限的人才能删除数据。

之前使用SpringSecurity保护web应用是通过实现UserDetailsServer让Spring能获取用户信息，然后就可以让所有的请求都需要登录的用户才能访问，以及更进一步地根据user的角色和权限进一步管理请求访问。
但保护 REST API 与保护基于浏览器的 web 应用程序不同,主要是因为 REST API 是无状态的,它们不会在服务器端保留任何客户端状态,也就是说REST
Service无法获取User信息。(这是一个优点)

本博文将介绍如何遵循Oauth2规范使用 Spring Security 保护 REST API。

## OAuth2 介绍

REST service中不存储任何客户端状态，但是其本身仍然需要验证用户身份，以及用户是否有权限访问某些资源。具体来说项目中的REST
Service仍然需要使用@PreAuthorize注解来限制用户访问某些资源.

但是用户(客户端)请求该如何让REST服务确幸自己有权限呢？
当然可以直接在请求中直接attach上用户凭证，但那显然是敏感信息，不应该直接暴露在请求中。如果黑客以某种方式截获了请求，可以很容易地获得未编码的原始凭证，然后使用凭证进行非法操作.

> 在思路上需要强调的一点是，应该要尽可能地减少敏感信息的传输，越铭感的信息越应该减少其传输的次数和范围。因为只要进行传输，就有可能被截获，而且传输的次数越多，被截获的概率就越大。
 {:.prompt-tip}

REST服务当然也不需要所有的用户凭证才能执行操作，举一个具体的例子，你要去看电影，服务员只要知道你买了票就行,而不需要知道你的全名与银行卡号等去查询你具体的银行支付订单。
问题的关键点在于<u>"信任"</u>,如何以敏感程度最低的方式来建立信任关系是关键点。

也许你很快就能从之前那个电影的例子中察觉到解决方案，那就是使用票据来替换敏感的凭证信息进行传输与认证。

票据从哪里来？自然是你在支付完成之后，支付平台给的票据。这个票据是支付平台与你之间的信任关系的体现，你可以凭借这个票据去电影院看电影，而不需要告诉电影院你的银行卡号等敏感信息。
也就是说，你当然还是要泄露一些你的用户凭证类的敏感信息给平台，但是你可以确定这个平台是值得信任的第三方专业平台，或者这个平台就是电影院自己的平台.以及，你只需要泄露一次，之后你用的就一直是票据了。其中没有你的敏感信息。若是你的票据被偷，最严重的后果也是别人可以去看电影(
使用你的服务),而不是让别人知道你的银行卡号等敏感信息。

那么现在我们将上述例子中的角色替换成Oauth2中的角色，就是这样的：

- 发电影票的支付平台 - 授权服务器
- 电影院 - 资源服务器
- 电影票 - access token

那么要使用REST服务，需要先找授权服务器获取access token，然后使用access token去访问资源服务器的REST服务。

上面的流程是一个客户端的视角，与那个电影院的例子不同，具体来说在网络服务中，用户和客户端是两个需要区分的概念，对于用户来说，使用OAuth2的体验是类似与这样的:

在一个第三方登录的场景中，你目前操作的网页作为客户端想要消费某个第三方资源服务器的服务，但是该资源服务器当然不认识这个客户端，所以自然地，
客户端跳转请求到了授权服务器中，接着你输入你的用户凭证(账号和密码)给授权服务器表明你的身份，然后授权服务器会给客户端token来让客户端去访问资源服务器的服务。

---
Oath2的基本概念与流程就是这些内容, 补充一点就是授权服务器颁发Token有几种不同的模式。

- 简单模式：直接发送token给客户端
- 授权码模式：发送授权码给客户端，客户端再用授权码去获取token(更加安全，最常用也是最佳实践)
- 密码模式：不涉及授权服务器，而是客户端直接获取你的用户凭证并直接将其转为token使用,这是一个非常不安全的模式，因为客户端可以直接获取你的用户凭证，现代浏览器已经不推崇使用这种模式了。
- 客户端模式：不使用客户的凭证，而是使用客户端自己的凭证获取token，一般用于客户端自己访问自己的资源服务器，执行一些非用户相关的操作。

为什么授权码模式比直接发送token更安全？

其实也就是多了一步，若是授权服务器直接发送token，那么发送给谁这一问题上可能出错，有可能发错人了，或者是发给了一个假冒的客户端，这样就会造成安全问题。
多了一步是发送授权码给客户端，然后需要客户端再用授权码去获取token，这多出来的一步可以让授权服务器确认客户端的身份，然后再发送token给客户端，这样就可以避免上述的安全问题。

![Oauth2授权码模式](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/读书笔记/Spring实战第六版/Oauth2授权码模式.594epk3kark0.webp)

---
> 接下来就是实践的内容，注意每一部分都是完全独立的项目
 {:.prompt-info}

## 创建授权服务器

授权服务器的作用就是在用户授权后代表用户颁发token给客户端.
所以其主要需要完成的工作是：

1. 获取用户身份
2. 创建token
3. 颁发token

1的实现直接用SpringSecurity完成即可

2和3的通过添加`spring-security-oauth2-authorization-server`依赖来帮助实现，即2和3的具体细节 不需要我们手动实现，我们要做的是一些其他的事情。

> 这个依赖不属于SpringSecurity，需要单独添加。
> 因为Spring官方鼓励使用第三方的授权服务器，例如Google，Facebook等。这个依赖是由社区驱动的，实验性质的，所以不属于SpringSecurity的核心功能。
 {:.prompt-info}

添加完依赖后需要额外在SecurityFilterChain中配置 `OAuth2AuthorizationServerConfiguration`以开启授权服务器的功能。

接着需要实现一个重要的接口，RegisteredClientRepository，其与SpringSecurity中的UserDetailsServer类似，只是不用于获取用户的信息，这个接口定义的方法是用户客户端的信息。
也就是说，(当然)不是所有的客户端都可以获取token，而是只有预先在授权服务器中注册过的客户端才可以获取token，这个接口就是用于哪些预先注册过的可以信任的客户端的信息的。

RegisteredClient实体本身需要包含的信息有：客户端名称与密码，授权范围，授权类型，重定向地址，token有效期等信息。其中重定向地址指的是授权服务器授权发送授权码后重定向的地址。
其应该是客户端指定的客户端自己的可信地址,然后该地址的客户端将url中的授权码信息解析然后再发送给授权服务器来获取token。这就是授权码模式的中多出来的一个步骤需要配置的地方.
补充一点是RegisteredClient可以配置一个clientSettings的Lambda，其可以配置一些额外的信息，在案例中其额外配置的一个用户consent的配置，即用户登录之后，还需要确定是否授权给客户端。

接着就是JWT的部分，就像之前说的那样，JWT具体的创建与颁发不需要手动实现，但是需要配置一些信息，例如签名算法，签名密钥，token的有效期等信息。
其中主要是签名算法与签名密钥，书中的做法是

```java
@Bean
public JwtDecoder jwtDecoder(JWKSource<SecurityContext> jwkSource) {
  return OAuth2AuthorizationServerConfiguration.jwtDecoder(jwkSource);
}

@Bean
public JWKSource<SecurityContext> jwkSource() {
  RSAKey rsaKey = generateRsa();
  JWKSet jwkSet = new JWKSet(rsaKey);
  return (jwkSelector, securityContext) -> jwkSelector.select(jwkSet);
}
private static RSAKey generateRsa() {
  KeyPair keyPair = generateRsaKey();
  RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
  RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();
  return new RSAKey.Builder(publicKey)
    .privateKey(privateKey)
    .keyID(UUID.randomUUID().toString())
    .build();
}
private static KeyPair generateRsaKey() {
  try {
    KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
    keyPairGenerator.initialize(2048);
    return keyPairGenerator.generateKeyPair();
  } catch (Exception e) {
    return null;
  }
}
```

由上到下不断嵌套，总之就是创建了一个RSAKeyPair(公钥和私钥对)，将其存入JWKSet中，然后将JWKSet存入JWKSource中，最后将JWKSource存入JwtDecoder中。
微调这些配置我们可以创建不同算法的秘钥，创建多个秘钥等操作。

这些就是全部内容了,运行项目之后到指定url即可看到授权服务器的登录页面，登录之后其会跳转到客户端指定的重定向地址，并且url中会带有授权码信息，然后客户端再用授权码去获取token即可。

## 利用资源保护器保护API

不变的是，仍然需要使用@PreAuthorize注解来标注哪些方法需要被保护，以及需要哪些权限才能访问。或者搭配authorizeRequests()
方法来配置哪些url需要被保护，以及需要哪些权限才能访问。

但验证token的有效性，然后再根据token中的信息来判断用户是否有权限访问该资源等一系列的操作，
我们不需要手动实现这些，只需要添加`spring-boot-starter-oauth2-resource-server`依赖即可，其会自动帮我们实现这些功能。
[]
在配置类中添加以下代码进行启用

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
  http
    ...
      .and()
        .oauth2ResourceServer(oauth2 -> oauth2.jwt())
  ...
}
```

这是资源服务器的一个最简单的配置，其检查令牌的内容，以查找它包含哪些安全声明。  
但这还不够，为了验证token的有效性，我们需要授权服务器的地址来获取公钥，以验证token的签名是否正确，以及token是否过期等信息。

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: http://localhost:9000/oauth2/jwks
```

这些就是全部内容了，但目前要验证上述的一些，我们在命令行或者postman这类工具中比较麻烦，毕竟还没有一个client，所以接下来就是实现一个client.

## 开发客户端

> 作者实现的使用REST服务的客户端很简略与粗糙，这一部分有更成熟的第三方工具可以使用(作者可能在后续的章节中作者可能会优化)
> 所以，以下内容不是最佳实践，仅作为参考.
 {:.prompt-info}

加一个`spring-boot-starter-oauth2-client`依赖，让其完成与授权服务器的交互，获取token等操作。 注意这个依赖本身已经包含了SpringSecurity。

然后进行基本配置

```java
@Bean
SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
  http
    .authorizeRequests(
      authorizeRequests -> authorizeRequests.anyRequest().authenticated()
    )
    .oauth2Login(
      oauth2Login ->
      oauth2Login.loginPage("/oauth2/authorization/taco-admin-client"))
    .oauth2Client(withDefaults());
  return http.build();
}
```

其让所有请求都需要身份验证,并添加了一个OAuth2Login页面。

注意！ 这不是一个需要用户名与密码的登录页面，同时，这也不是授权服务器的url地址。这个页面是授权服务器完成验证发送授权码的地址，客户端会在这个地址解析url中的授权码，然后再发送给授权服务器来获取token。

然后就是配置客户端自己的信息了

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          taco-admin-client:
            provider: tacocloud
            client-id: taco-admin-client
            client-secret: secret
            authorization-grant-type: authorization_code
            redirect-uri: "http://127.0.0.1:9090/login/oauth2/code/{registrationId}"
            scope: writeIngredients,deleteIngredients,openid
```

注意这里的redirect-uri是授权服务器的地址，即第一次redirect的地址。
具体再说明一下就是 用户请求使用第三方登录 - 第一次redirect到授权服务器，填写用户密码登录 - 第二次redirect到客户端地址就是再上面一个`oauth2Login.loginPage`

还有授权服务器的信息
```yaml
spring:
  security:
    oauth2:
      client:
...
        provider:
          tacocloud:
            issuer-uri: http://authserver:9000
            
            
            authorization-uri: http://authserver:9000/oauth2/authorize
            token-uri: http://authserver:9000/oauth2/token
            jwk-set-uri: http://authserver:9000/oauth2/jwks
            user-info-uri: http://authserver:9000/userinfo
            user-name-attribute: sub
```
一般只要issuer-uri一个就行了，其他的spring会自动猜测，但如何授权服务器有自己的特殊配置，那么就需要手动配置了。

至此，最终客户端会得到token，并将其存在SecurityContextHolder中。

但如何使用token，即将其加入到请求头中，这个就需要自己实现了。就行一开始所说的。作者的实现很粗糙和简略，这里就不阐述了，
