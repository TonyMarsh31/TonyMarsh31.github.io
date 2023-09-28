---
layout: post
title: "SSR场景下,使用SpringValidation完成属性验证"
date: 2023-04-30 00:10 +0800
category: [ 读书笔记, "spring实战第六版" ]
tag: [ Spring,属性验证,表单验证 ]
---

个人之前接触的都是前后端分离的架构，Validation的工作是直接在前端完成的。在Spring实战这本书中作者一开始做的是SSR项目,其使用了SpringValidation完成属性验证。这是我第一次看到这种方式，所以记录一下。

## 步骤

1. 添加依赖
2. 在需要验证的类上添加注解,声明验证的规则
3. 在需要实施验证的地方添加@Valid注解
4. 在SRR的页面上添加错误信息

## 书本中的SSR项目 TacoCloud的例子

### 添加依赖

```xml

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

### 在Taco和TacoOrder上添加注解

代码太多了，就不贴了，主要用到了这几个注解

- @NotNull, @NotBlank 两者都是用来验证字符串的，但是@NotBlank会忽略空格
- @Size 用来验证元素的个数，例如String的字符数，List的元素个数
- @Digits 用来验证数字的位数，可以细分到整数位数和小数位数
- @Pattern 用来验证字符串的格式，可以使用正则表达式

还有一个是@CreditCardNumber，用来验证信用卡号的，但是这个注解是hibernate-validator提供的拓展注解，不属于JavaBean
Validation规范的一部分。不过SpringValidation默认添加了hibernate-validator的实现，所以可以直接使用,不需要额外添加依赖。

### 在Controller接受Taco和TacoOrder的地方添加@Valid注解

> 这一步骤是必要的，不然不会触发验证
{:.prompt-tip}

具体的代码如下

```java
  @PostMapping
  public String processTaco(@Valid @ModelAttribute("taco") Taco taco, Errors errors) {
    if (errors.hasErrors()) {
      return "design";
    }
    // Save the taco...
    // We'll do this in chapter 3
    log.info("Processing taco: " + taco);

    return "redirect:/orders/current";
  }
```

在方法参数中，Taco前面添加了@Valid注解，这样Spring就会在调用这个方法之前，对Taco进行验证。验证的结果会被放到Errors对象中，然后我们可以根据Errors对象中的hasErrors()
方法来判断是否有错误。

### 在页面模板上显示错误信息

在之前的控制器中，如果验证失败，还是会返回到design页面，所以我们需要在design页面上显示错误信息.

SSR的页面模板是Thymeleaf，所以我们用到了Thymeleaf的错误信息显示功能 th:errors

```html
<label for="ccNumber">Credit Card #: </label>
<input type="text" th:field="*{ccNumber}"/>
<span class="validationError"
      th:if="${#fields.hasErrors('ccNumber')}"
      th:errors="*{ccNumber}">CC Num Error</span>
```

th:errors 属性引用 ccNumber 字段，并且假设该字段存在错误，它将用验证消息替换 <span> 元素的占位符内容。

## 思考，这种方式的问题

其实本质上还是SSR的问题，不管参数合法与否，请求都会被发送，Validation都会在后端进行验证，这样会增加服务器的压力。

TradeOff换来的优点是否值得我还不清楚, 当然，这不是最佳实践的讨论，这只是书中的教学案例，我目前还没看到书中后面的地方，不知道作者是怎么处理的。
