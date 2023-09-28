---
layout: post
title: "在Spring中构建与使用REST服务"
date: 2023-05-04 16:08 +0800
category: [ 读书笔记, "spring实战第六版" ]
tag: [ Spring,创建REST服务,REST ]
---

> 本博文中的内容不包含安全相关的内容。其统一在另一篇博文中进行讨论。
{:.prompt-info}

## 概念

书中没有具体展开所有的REST请求方式，没有介绍HEAD和OPTIONS请求方式,(待补充)

GET、POST、DELETE很容易理解，但是更新有PUT和PATCH。

- POST 从无到有添加新数据，预期是数据库中不存在该数据，如果存在则会报错。(例如注册一个新的用户)
- PUT 字面理解为将资源直接put到指定位置  
  为了保证这一点，当存在资源时，Put会覆盖原有资源，不存在时则会创建新的资源。
- PATCH 则是对资源进行部分更新,字面意思上就是打补丁  
  PUT 和PATCH进行区分的一个很好的例子是例如同样是提交一个表单作为更新操作，当存在一些null属性时，若为put则直接更新，而在patch中需要对null属性进行判断，若为null则不更新。

## 用MVC注解创建REST服务

简而言之，REST API 与网站没有太大区别。两者都涉及对 HTTP 请求的响应。但关键的区别在于，网站是用 HTML 响应这些请求，而REST
API通常以面向数据的格式（如 JSON 或 XML）进行响应。

即与普通Controller不同的是，一个REST
Controller需要返回一个数据对象，而不是一个视图。所以这需要在方法上使用@ResponseBody注解。但若是你确定本控制器中的所有的方法都是返回数据对象的REST
API，那么可以在类上使用@RestController注解，这样就不需要每个方法都使用@ResponseBody注解了。

### GET

```java
@RestController
@RequestMapping(path="/api/tacos", produces="application/json")
@CrossOrigin(origins="*")
public class TacoController {
  private TacoRepository tacoRepo;

  public TacoController(TacoRepository tacoRepo) {
    this.tacoRepo = tacoRepo;
  }

  @GetMapping(params="recent")
  public Iterable<Taco> recentTacos() {
    PageRequest page = PageRequest.of( 0, 12, Sort.by("createdAt").descending());
    return tacoRepo.findAll(page).getContent();
  }
}
```

这里有一个细节是：代码RequestMapping注解中声明了produces属性，来指定返回的数据类型(或是consumes属性，指定接收的数据类型)
。这样就可以使得同名的控制器方法不会被同一个请求路径的不同请求方式所覆盖，而是根据请求方式的不同，调用不同的方法。(
类似于重载)

### POST

```java
@PostMapping(consumes="application/json")
@ResponseStatus(HttpStatus.CREATED)
public Taco postTaco(@RequestBody Taco taco) {
  return tacoRepo.save(taco);
}
```

> 注意这里的`@RequestBody`注解是非常重要的,它会告诉Spring我们要的东西在请求体中，并且要将其反序列化为指定对象。如果不指定这个注解，那么Spring会将其解析为请求参数或表单参数。
 {:.prompt-warning}

这里的一个小优化是：使用@ResponseStatus注解，来指定返回的状态码表明有新数据被创建，而不是使用默认的200。

### PUT/PATCH

[开头](#概念)已经说明了两者同作为更新操作，但在细节上的不同。  
然而这只是一种规范，具体的实现还是需要开发者完成。

以下是一个PUT的例子：

```java
@PutMapping(path="/{orderId}", consumes="application/json")
public TacoOrder putOrder(
          @PathVariable("orderId") Long orderId,
          @RequestBody TacoOrder order) {
  order.setId(orderId);
  return repo.save(order);
}
```

以下是一个PATCH的例子：

```java
@PatchMapping(path="/{orderId}", consumes="application/json")
public TacoOrder patchOrder(@PathVariable("orderId") Long orderId,
          @RequestBody TacoOrder patch) {

  TacoOrder order = repo.findById(orderId).get();
  if (patch.getDeliveryName() != null) {
  order.setDeliveryName(patch.getDeliveryName());
  }
  if (patch.getDeliveryStreet() != null) {
  order.setDeliveryStreet(patch.getDeliveryStreet());
  }
  
  ...
  
  return repo.save(order);
}
```

PATCH的实现往往会复杂一点，上述案例中没有解决的问题是：

- 如果patch中就是要将某些属性置为null，那么这里的实现就会出现问题。### DELETE
- 如果patch的对象是一个嵌套对象，那么若想要添加或删除一个子对象，请求还需要附上完整的父对象的结构。

对与这些问题的解决方式有很多，书中没有具体展开，仅提到了一种思路是patch发送一个指令，而不是一个修改后的对象可以解决这些问题。

~~(
我的个人实践经历中没有实现过复杂的patch操作。对于复杂更新操作我的做法是先获取源数据，让用户直接在源数据上进行修改，然后使用PUT方法，将整个对象进行替换。当然，最终还是要根据实际的应用场景来决定使用哪种方式。)~~

更新：复杂patch是有必要的，因为先get后put进行替换的更新实现的缺点有：1.客户端复杂度高，可能带来不好的用户体验。2.数据一致性风险,在get和reput之间，数据可能被其他客户端修改。3.网络传输的数据量大，可能会影响性能。4.安全性问题

### DELETE

代码示例略。

同样一个小优化是：使用 `@ResponseStatus(HttpStatus.NO_CONTENT)` 进行了注解，以确保响应的 HTTP 状态是 204（NO CONTENT）

## 用SpringDataREST为Repository自动创建REST服务

在书中即之前的博文介绍过SpringData可以为我们自动创建Repository的实现，而SpringDataREST可以更近一步，根据Repository的定义，自动创建REST服务。
而达成这一步只需要引入依赖即可。是的，就是这么简单。(假设Repository已经定义好了,具体来说就是继承CRUDRepository)

```java
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-rest</artifactId>
</dependency>
```

### 上手即用

Spring创建的REST服务的路径是根据Repository的名字来确定的，比如有一个名为IngredientRepository的Repository，那么Spring会创建一个名为ingredients的REST服务。

测试访问 /ingredients即可完成一次对Ingredient的GET请求获取所有的Ingredient。

> 所有自动生成的REST服务都是支持传递分页参数与排序参数的。  
> 例如`/tacos?sort=createdAt,desc&page=0&size=12`
{:.prompt-tip}

值得一提的是SpringDataREST的实现依赖于Spring HATEOAS。HATEOAS是一种规范，其在json中除了数据之外还附带了一些链接，这些链接可以让客户端发现和访问其他资源。
这样做的目的是为了让客户端不需要了解REST
API的结构就可以使用,就像浏览网页一样，不需要知道网页的结构，只需要点击链接即可访问其他页面。但这同样也一定程度上添加了额外数据，增加了复杂度。不过Spring
HATEOAS已经帮我们做好了这些事情，我们只需要使用即可。而如何自定义实现HATEOAS规范的REST服务不在书中与本博文的讨论范围内，有兴趣的可以自行查阅资料。


---
至此，其实已经完成了一个完整的REST服务，可以直接访问 /api 来获取SpringDataREST自动创建的REST服务的根路径。

但是，你可能还要就细节上的一些问题进行解决。例如其提供的REST服务的路径可能与我们自定义实现的REST服务的路径冲突，这时候就需要对其进行配置。
还有一个小问题是，SpringDataREST节点生产的名字规范是将Repository的名称复数话，但有些名字复数化后不符合英语语法，例如taco的复数被错误的定义为tacoes。这也需要进行调整。

当然还有很多重要的问题需要调整。例如如何让SpringDataREST不要暴露某些属性,不过正如最开始所说，所有的安全问题会在另一篇博文中统一进行讨论。

### 细节配置

添加路径前缀：

```yaml
spring:
  data:
    rest:
      base-path: /data-api
```

这样SpringData自动生成的REST服务的路径就会变成 /data-api/... 了。访问 /data-api/api 可以查看所有的REST服务的根路径。

修改节点名字：

使用java注解对Repository的实现对象进行标注，示例如下

```java
@Data
@Entity
@RestResource(rel="tacos", path="tacos")
public class Taco {
  ...
}
```

结果：

```json
"tacos" : {
"href": "http://localhost:8080/data-api/tacos{?page,size,sort}",
"templated": true
}
```

taco的REST服务地址被正确地自定义为tacos，而不是自动生成的tacoes

## 使用RestTemplate消费REST服务

可以在config中注入RestTemplate的bean，然后在需要使用的地方直接使用即可。

```java
@Bean
public RestTemplate restTemplate() {
  return new RestTemplate();
}
```

### 套路: 方法参数重载与返回类型

RestTemplate的使用只要调用对应的方法即可。不过其不同重载参数有不同的细微差别，这里统一进行说明。

![RestTemplate参数重载](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/RestTemplate参数重载.1mts3d9fb8xs.webp)

URL参数一般就是String,然后指定返回类型也直接传递一个Class对象即可。

主要是参数传递:如果URL中指定为url传参，那么在String中写明插值表达式，然后在参数列表中传递对应的参数即可(注意是顺序的)。
或者传递一个Map的Wrapper包装。(这在之后的使用示例中有很多，此处就不贴代码了)

但是一个url需要传递的参数比较复杂的情况下，使用URL类进行构建会更好，这样就不用写很复杂的urlString了
例如 `"http://localhost:8080/ingredients?page={page}&size={size}&descBy={descBy}"` 这种写起来麻烦了
使用URL构造器，

```java
  URI url = UriComponentsBuilder
        .fromUriString("http://localhost:8080/ingredients")
        .queryParam("page", page)
        .queryParam("size", size)
        .queryParam("descBy", descBy);
        build();
```

除此之外，其返回类型也有不同的类型，   
...ForObject返回的就是一个对象，而...ForEntity返回的是一个ResponseEntity对象，其包含了响应的状态码，响应头，响应体等信息。

---

### 使用示例

```java 
public Ingredient getIngredientById(String ingredientId) {
  return rest.getForObject("http://localhost:8080/ingredients/{id}",
                    Ingredient.class, ingredientId);
}
```

参数是可变参数，可以传递多个参数，但是要注意顺序。

```java 
public Ingredient getIngredientById(String ingredientId) {
  Map<String, String> urlVariables = new HashMap<>();
  urlVariables.put("id", ingredientId);
  return rest.getForObject("http://localhost:8080/ingredients/{id}",
                  Ingredient.class, urlVariables);
}
```

另一种选择是传递一个Map，这样就不用关心顺序了。

```java 
public Ingredient getIngredientById(String ingredientId) {
  Map<String, String> urlVariables = new HashMap<>();
  urlVariables.put("id", ingredientId);
  URI url = UriComponentsBuilder
          .fromHttpUrl("http://localhost:8080/ingredients/{id}")
          .build(urlVariables);
  return rest.getForObject(url, Ingredient.class);
  
  //另一个例子
    URI url = UriComponentsBuilder
        .fromUriString("http://localhost:8080/ingredients")
        .queryParam("page", page)
        .queryParam("size", size)
        .queryParam("descBy", descBy);
        build();
}
```

如果URL实在过于复杂，可以使用URL构造器进行构建。

```java 
public Ingredient getIngredientById(String ingredientId) {
  ResponseEntity<Ingredient> responseEntity =
    rest.getForEntity("http://localhost:8080/ingredients/{id}",
              Ingredient.class, ingredientId);
  log.info("Fetched time: " +
              responseEntity.getHeaders().getDate());
  return responseEntity.getBody();
}
```

使用getForEntity获取ResponseEntity对象，然后可以获取响应头，响应状态码等信息。

```java 
public java.net.URI createIngredient(Ingredient ingredient) {
  return rest.postForLocation("http://localhost:8080/ingredients",
                        ingredient);
}
```

postForLocation方法可以返回一个URI对象，这个URI对象就是新创建的资源的URI。

```java 
public Ingredient createIngredient(Ingredient ingredient) {
  ResponseEntity<Ingredient> responseEntity =
    rest.postForEntity("http://localhost:8080/ingredients",
                ingredient,
                Ingredient.class);
  log.info("New resource created at " +
                responseEntity.getHeaders().getLocation());
  return responseEntity.getBody();
}
```
