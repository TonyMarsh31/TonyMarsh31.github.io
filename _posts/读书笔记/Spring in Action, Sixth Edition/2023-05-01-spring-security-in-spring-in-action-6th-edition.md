---
layout: post
title: "SpringSecurity - how to use | Spring实战第六版"
date: 2023-05-01 21:50 +0800
category: [ 读书笔记, "spring实战第六版" ]
tag: [ Spring,SpringSecurity ]
---

> 就像书名Spring in action写的那样，内容主要都是how to use的内容，学习的目的只是对Spring的操作有个大致的了解，很多细节原书没有深入。
> 个人仅对自己觉得不明白与重要的地方做了一点补充，其他的就只是简单的记录一下，以备后续查阅。
{:.prompt-tip}

## 启动SpringSecurity

只要引入了`spring-boot-starter-security`
依赖，就会启用SpringSecurity的默认配置,开启了基于表单的登录，弹出一个登录表单，要求输入用户名和密码，然后提交到`/login`
，SpringSecurity会处理这个请求，验证用户名和密码是否正确，如果正确，就会重定向到`/`，否则会重定向到`/login?error`。
默认的用户名是user，密码会在程序启动时打印在控制台上

默认情况下，SpringSecurity会启用以下特性：

- 使用了上述介绍的一个简单的登录页面来提供验证
- 为所有的请求URL添加安全验证

但同时，默认的配置当然也是非常有局限性的， 其只有一个用户, 且没有权限控制的概念 要全面地使用SpringSecurity，我们还有很多工作要做。

在书中，以其TacoCloud项目为例,其需要实现的功能是：

- 提供一个匹配(自定义的)该网站的登录页面。
- 为用户提供注册页面，让新的 Taco Cloud 用户可以注册。
- 为不同的请求路径应用不同的安全规则。例如，主页和注册页面根本不需要身份验证。
- 以及其他一些细枝末节的地方在后面的小节中具体说明...

## 登录校验与用户注册

### 思路

实际上我们不需要显式地进行 获取用户输入的用户名和密码，验证用户名和密码是否正确，以及登录成功后的处理等等事情,
因为这些工作SpringSecurity会自动执行. 我们对该模块真正要做的事情是完成以下两个功能：

- UserDetailsService - 用于(让SpringSecurity)获取用户信息,进而自动进行登录校验
- 一个注册页面 - 将注册的新用户信息持久化到数据库中

### 登录校验

UserDetailsService接口是SpringSecurity提供用于获取用户信息的接口，其只定义了一个方法,
只要实现该接口，SpringSecurity就可以通过该接口来获取用户信息，进而进行自动的登录校验。

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

我们只需要实现上面这个Spring定义的**根据用户名找到用户信息**的接口即可处理用户登录，而无需考虑校验细节。同时，Spring要求用户信息的格式需要符合UserDetails规范。
为了满足这个要求，Spring提供了一个已实现UserDetails接口的User类作为模版，我们可以从它继承来实现自己的User类。但是这种做法会与Spring框架产生强耦合。因此，更好的做法是自己手动实现UserDetails接口。

针对UserDetails规范，一个重要的方法是getAuthorities，用于获取用户的权限信息。书中的演示项目中其getAuthorities的实现是固定返回USER权限,这显然只是为了演示而不是最佳实践.
我自己认为一个好的做法是User对象的构造器中需要传入一个权限列表，getAuthorities方法直接返回该权限列表。对于该权限列表的构造，可以根据User在数据库中的role属性来查询authority表，然后构造权限列表。

总之，如果不使用SpringSecurity提供的User模版，而是使用自定义的User，那么我们需要为其实现UserDetails接口.  
然后一种流行的做法是在自定义的User中额外实现一个方法，将User转换为UserDetails,  
这样最终在UserDetailsService的实现中，在使用repo 查询到User之后，再调用该方法将其转换为UserDetails返回给SpringSecurity即可。

### 用户注册

思路很简单，就是一个简单的表单提交，将用户信息持久化到数据库中即可。

但还是同样的问题，这仅仅是一个十分简单的例子，书本中没有考虑权限相关的设计。

> 补充：SpringSecurity提供了一些默认的密码加密组件，选择想要的将其cast为PasswordEncoder注册到Spring容器中即可。这一般是在SpringSecurity的配置类中完成的。
Spring在登陆验证的时候会自动对用户输入的密码encode之后再和数据库中的密码进行比较。,所以我们持久化到数据库的密码也是encode之后的密码。(
数据库不保存明文密码这是基本常识了)
{: .prompt-tip}

## 保护web请求 - SecurityFilterChain中可以做的事情

### 请求所需权限配置

这应该是最重要的配置了，毕竟不是所有的请求都是所有人都可以访问的，但有些请求我们例如主页，注册页面等等是不需要权限的。

这一部分其实随着SpringSecurity版本的更新，具体配置的语法会有所不同，但是大致上都是一样的，都是通过配置一个SecurityFilterChain来实现的。

----
补充内容 ：

作者在之后的文本中提到过另一种做法是 直接定义一个继承自WebSecurityConfigurerAdapter的Config类(
用@Configuration注解注入，然后通过@EnableWebSecurity开启)
，然后重写其configure方法来完成配置，configure方法同样用到了HttpSecurity的入参，方法体内部的具体配置是一样的  

个人觉得该方法更简洁直接，但是如果项目中的配置比较复杂，那么还是使用SecurityFilterChain的方式更好一些，这样可以将复杂的配置进行拆分(
其实也没差太多，仅做补充)

> 补充2，与上述逻辑类似的，UserDetailsService的实现既可以像之前说的那样手动实现后直接注入Spring后就行，
> 也可以通过继承WebSecurityConfigurerAdapter，在重写的config方法中,添加AuthenticationManagerBuilder入参，然后使用其userDetailsService方法来配置，指定一个实现了UserDetailsService接口的类的实例作为参数。  
> 两种方法最终都是配置，但是前者只是完成配置的内容，没有具体的"配置代码",因为这部分工作Spring帮我完成了。 而后者直接接触到了SpringSecurity的内部，通过其提供的方法来完成配置。即用代码显式地完成了配置。  
> 前者是一种更加高层次的抽象，而后者是一种更加底层的抽象。我们往往只需要使用前者即可，但是如果我们需要更加细致的配置，那么后者就是我们的选择了。(但这几乎没有什么必要)  
> 总而言之，这些补充内容中的信息都不是必须的，但是了解一下也是好的。 Spring帮我们自动完成了很多工作，我们只需要按照它的规范来做就行了，这里介绍的是更加底层的显式地配置。
{: .prompt-tip}

----

```java
// 直接继承WebSecurityConfigurerAdapter，重写configure方法对HttpSecurity进行配置
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http
    .authorizeRequests() 
    .antMatchers("/api/tacos", "/orders")
    .hasAuthority("USER") 
    .antMatchers("/**")
    .permitAll();
  }
}
```

```java
// 通过(在配置类中)注入SecurityFilterChain进行配置
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
  return http
    .authorizeRequests()
      .antMatchers("/design", "/orders").hasRole("USER")
      .antMatchers("/", "/**").permitAll()

  .and()
  .build();
}
```

> 注意代码片段中的一个细节是，方法1中我们只要进行configure就行了，而在方法2中，在configure结束后我们要显式地call一次build方法，  
> 这是因为方法1中，我们是reconfigure一个已经存在的SecurityFilterChain，而方法2中，我们是新建了一个SecurityFilterChain，所以需要进行显式build，然后注入Spring。
{: .prompt-warning}
 
对于一些复杂的授权场景，SpringSecurity提供了SpEL表达式的支持，可以通过表达式来实现更加复杂的授权逻辑。例如只想允许具有
ROLE_USER 权限的用户在周二（例如，在周二）创建新的 Taco;

```java
      .antMatchers("/design", "/orders")
        .access("hasRole('USER') && " +
          "T(java.util.Calendar).getInstance().get("+
          "T(java.util.Calendar).DAY_OF_WEEK) == " +
          "T(java.util.Calendar).TUESDAY")
```

可以放心的是，虽然在格式上只是字符串，但是现代的IDE会提供语法高亮和错误提示，所以不用担心写错了。

### 自定义用户登录页面

在SecurityFilterChain中添加

```java
.and()
      .formLogin()
        .loginPage("/login")
        .defaultSuccessUrl("/design", true)
```

注意因为lambda的特性,一般来说要在不同配置之间用and()连接，defaultSuccessUrl表示登录后跳转的页面，true表示总是跳转，false表示只有在没有指定跳转页面的情况下才跳转。

### 第三方登录支持(OAuth2)

同样的，只是how to use，具体不展开太多

- 加入依赖
- 在application文件中配置
- 在SecurityFilterChain中添加

添加的时候一般还是要指定loginPage为自定义的login，只是在登陆页面中添加一个超链接来指向OAuth2的登陆页面。

### 登出

在SecurityFilterChain中添加一个logout()即可，SpringSecurity会自动处理登出的逻辑。
可以在页面中添加一个超链接来触发登出，超链接的href为/logout即可。

### CSRF保护

默认开启,具体不展开了

## 启用方法级别的保护 - 更多保护防止未授权的访问

启用了SecurityFilterChain后，我们已经可以保护web请求了，但是方法级别的保护还是有必要的。例如这个场景，虽然用户请求不能直接到达某些方法，但是可能通过间接调用，一个控制器方法调用另一个控制器方法可能会出现问题。

具体启用方法很简单，两个注解: @PreAuthorize和@PostAuthorize。使用注解表述方法，注解属性中填写SpEL表达式即可。

pre表示在方法执行前进行检验，post表示在方法执行后进行检验。一般最常用的就是Pre，因为我们一般是在方法执行前进行授权，如果授权失败，方法就不会执行。

## 获取登录用户的信息

当校验通过后，用户的请求自然就可以到达控制器中的方法了。 但是，我们如何获取用户的信息呢？例如我们想要获取用户的用户名，或者用户的权限列表等等。

同样，只是howto，具体Spring对其底层实现不展开。
1. 注入 `java.security.Principal` 对象到控制器方法中
2. 注入 `org.springframework.security.core.Authentication` 对象到控制器方法中
3. 注入一个 `@AuthenticationPrincipal` 注解的方法参数 (该注解在Spring Security的 `org.springframework.security.core.annotation` 包中)
4. 使用 `org.springframework.security.core.context.SecurityContextHolder` 来获取安全上下文

1的Principal可以通过get方法获取信息，但是这个信息是有限的，只有用户名。要完整的User对象还要用Repository再查询一次。

2的Authentication对象可以通过getPrincipal方法来直接获取User对象.

3的注解方法是最方便的，在方法参数中添加User后，使用注解标注即可，Spring会自动注入User对象。

4的做法是最底层的做法，好处是可以在非Controller的地方使用，例如在Service中使用。
