---
layout: post
title: "在Spring中响应式地持久化数据"
date: 2023-05-22 14:57 +0800
category: [ 读书笔记, "spring实战第六版" ]
tag: [ 响应式编程,响应式持久化数据,Spring,SpringWebFlux ]
---

> 无论是进行响应式开发还是传统开发，Spring Data家族中的项目都保持着较高的一致性。这意味着与
> 之前[SpringData博文]({% post_url /读书笔记/Spring in Action, Sixth Edition/2023-04-30-spingjdbc-springdata %})
> 中的内容相比，本文中的许多内容都会重复出现。因此，本文将着重记录与之前内容不同的部分，并对其进行简要介绍。
{: .prompt-info}

## R2DBC - JDBC的响应式替代

> 作者在《Spring实战第六版》中的写作时间大约在2020年左右。那个时候，R2DBC相对于JDBC在使用上还存在一些不足之处。由于这些问题，作者不得不编写一些额外的代码（补丁代码）来解决这些问题。然而，随着R2DBC的发展，这些问题可能已经得到了解决，R2DBC在使用上的改进可能已经使得这些额外的工作不再需要。
{: .prompt-tip}

R2DBC的全称是"Reactive Relational Database Connectivity"，即"响应式关系数据库连接"
。R2DBC是一种用于在响应式编程环境下访问关系型数据库的规范和API。它提供了一种异步、非阻塞的方式来进行数据库访问，并与响应式流（如Flux和Mono）无缝集成。

简单来说，R2DBC就是JDBC的一个响应式替换方案，支持针对传统的关系型数据库（如MySQL、PostgreSQL、Oracle、SQL
Server等）进行响应式编程,完成非阻塞的持久化。但因为它是建立在响应式基础上的，与JDBC的使用方式有很大的不同。是一个独立的规范，与Java
SE无关。

## 响应式地持久化关系型数据 - 通过R2DBC, 数据库为H2

### 依赖导入

```xml

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-r2dbc</artifactId>
</dependency>
```

还需要一个关系数据库以便进行数据持久化，以及相应的 R2DBC 驱动。我们将使用内存数据库 H2。因此，我们需要增加两个依赖项：H2
数据库库和 H2 R2DBC 驱动。依赖项如下所示

```xml

<dependencies>
  ...
  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
  </dependency>
  <dependency>
    <groupId>io.r2dbc</groupId>
    <artifactId>r2dbc-h2</artifactId>
    <scope>runtime</scope>
  </dependency>
  ...
</dependencies>
```

如果您使用不同的数据库，则需要配置相关依赖项以添加相应的 R2BDC 驱动程序。

### 定义实体

在流程上与使用JDBC时没有太大的不同。但需要注意一下几点：

- Spring Data R2DBC 需要实体属性具有setter方法，所以这意味着属性不能是final的。
- 在之前的Ingredient类中，作者尝试使用自定义的String类型作为ID属性。然而，这种做法在R2DBC这会导致了一些错误。作者并没有很清楚地解释这些错误的具体原因，也让我理解起来困难。
  为了解决这个问题，作者改变了策略，使用了Long类型的ID属性作为数据库的主键。同时，为了保留之前的String主键，作者将其作为一个称为"
  slug"的属性使用。这样，在逻辑上，作者使用的是String类型的"slug"作为主键，而实际在数据库中，使用Long类型的自动生成的主键作为主键。
  需要注意的是，之前的问题可能是由于在R2DBC的早期版本中对主键类型的限制所导致的。然而，在后续的R2DBC版本中，这个问题可能已经得到了改进和完善，使得使用自定义的String类型作为主键也能正常工作。
- 在作者写书地时期，R2DBC还不支持自动创建实体间的关联，所以需要手动维护实体间的关联。同样的，随着R2DBC的发展，这个问题可能已经得到了解决.

由于上述的最后一点，作者花了很多篇幅描述了如何手动维护实体间的关联(作为补丁代码)。

简单来说，使用JDBC/JPA可以直接将 List<Ingredient> 作为属性添加到 Taco。但是在R2DBC中不行。
作者简述了三种宏观思路来建立联系：

1. 保存主键，即修改为List<Long>类型的属性，但是这需要数据库支持将 列属性 设置为数组类型。
2. 使用中间表, 将关系保存到中间表中。
3. 使用JSON类型的属性,直接将List<Ingredient>转为json后
   ，作为varchar类型的属性保存到数据库中。但存在长度限制并无法保证完整性(转为json的过程中可能会丢失一些信息)。

作者最终选择了第一种方案

### 定义响应式Repository

简单来说，就是将之前继承的CurdRepository接口改为ReactiveCrudRepository接口

### 测试

类似与响应式测试中使用的StepVerifier进行测试，这里只需要为测试类 添加一个@DataR2dbcTest注解即可
导入Context中的Repository相关的Bean做测试了。

### 定义 聚合根Repository服务

主要要实现两个方法

- 持久化TacoOrder时，对其List<Taco>属性进行持久化
- 读取TacoOrder时，获取完全的List<Taco>

因为之前SpringDataJDBC等支持List<Taco>类型的属性，可以持久化TacoOrder作为聚合根，所以这些工作由SpringDataJDBC自动完成了。

但由于R2DBC不支持聚合根，所以自然无法持久化List<Taco>这样的数据，所以要单独多创建一个Taco的Repository对Taco手动持久化。然后还有手动实现上述提到的方法。
这一节主要将的就是这些补丁代码的实现。

先来梳理一下思路，List<Taco>不能直接被持久化，所以类似与之前讲到的思路，最终持久化的是List<TacoID>。
接着我们最终操作TacoOrder时，还是需要能直接获取完整的List<Taco>。所以在TacoOrder中还是需要有一个List<Taco>属性。
最终，TacoOder如下所示
```java
@Data
public class TacoOrder {

  ...
  
  private Set<Long> tacoIds = new LinkedHashSet<>();

  @Transient
  private transient List<Taco> tacos = new ArrayList<>();

  public void addTaco(Taco taco) {
    this.tacos.add(taco);
    if (taco.getId() != null) {
      this.tacoIds.add(taco.getId());
    }
  }
}
```
- 开始，用户创建taco，然后创建taco订单，这些会被传递到服务端。
- 此时TacoOrder有基本的订单信息，没有Taco信息。
- 然后第一次调用addTaco方法，将Taco添加到TacoOrder中，这样TacoOrder中的 `List<Taco> tacos`属性就有了Taco信息。
- 注意此时因为Taco没有持久化，所以其没有id属性，所以`Set<Long> tacoIds`是空的。
 
---

- 然后我们直接操作此时的TacoOrder进行持久化。
- 在具体的持久化方法中，自然要先持久化Taco，然后Taco有了id信息后，进行一次addTaco方法后，TacoOrder在保留`List<Taco> tacos`的同时，`Set<Long> tacoIds`也有了Taco的id信息。
- 接着因为有了`Set<Long> tacoIds` 那么直接持久化TacoOrder即可。

这就是整个持久化TacoOrder的流程。 查找TacoOrder是获取完整的List<Taco>的流程比较简单就不赘述了

代码如下所示
```java
public Mono<TacoOrder> save(TacoOrder tacoOrder) {
    return Mono.just(tacoOrder)
      .flatMap(order -> {
        List<Taco> tacos = order.getTacos(); // 获取Tacos
        order.setTacos(new ArrayList<>()); // 重置Tacos列表
        return tacoRepo.saveAll(tacos) // 持久化Tacos
            .map(taco -> {            // 持久化完成后，返回Flux<SavedTaco>，SavedTaco是有id的 (仅方便理解，没有新的SavedTaco对象)
              order.addTaco(taco); // 将SavedTaco重新add回 重置之后的Tacos列表
              return order;        // map的结果，返回Flux<TacoOrder>
          }).last();               // 确定只有一个TacoOrder，所以使用last()，返回Mono<TacoOrder>
      })
      .flatMap(orderRepo::save);  // 然后持久化TacoOrder
  }
  
public Mono<TacoOrder> findById(Long id) {
  return orderRepo
    .findById(id)   // 此时的Mono<TacoOrder> 中只有 Set<Long> tacoIds 有值，List<Taco> tacos 为空
    .flatMap(order -> {
      return tacoRepo.findAllById(order.getTacoIds()) // 根据Set<Long> tacoIds 获取Tacos
        .map(taco -> {                              // 把Tacos add到 List<Taco> tacos
          order.addTaco(taco);
          return order;                              // map的结果，返回Flux<TacoOrder>  (map的工作是次要的，只是为了确保返回类型一致，主要的工作就是上述的re add操作)
        }).last();                                    // 确定只有一个TacoOrder，所以使用last()，返回Mono<TacoOrder>
  });
```

Again,这一整个流程在使用SpringDataData/JPA等时是不需要的，因为SpringDataJDBC可以直接持久化List<Taco>属性。
我们目前做的都是补丁代码。同时测试代码也不赘述了，书中有。

> 其实整个流程的复杂度还是有一点的，理解上比较费劲，但是最终使用响应式的编码方式，其代码量不是很多。此时可以体会到不同于传统的编码方式，响应式编码方式的特点。
{:.prompt-tip}

## 响应式地持久化非关系型数据 - MongoDB 与 Cassandra

就像本博文开头说的那样，SpringData家族中的成员都提供了较高的一致性。 我们不需要做很多的修改(可以说是完全没有更改)，其就已经是响应式的了。
具体来说，实体定义完全不用改。然后就是继承的Repository接口改为响应式的接口即可。

因为是非关系型数据库，所以没有像使用R2DBC那样的补丁代码。其他的也没有什么特别需要强调的了。

## 总结

* Spring Data supports reactive persistence for a variety of database types including relational databases (with R2DBC), MongoDB, and Cassandra.
* Spring Data R2DBC offers a reactive option for relational persistence, but doesn’t yet directly support relationships in domain classes.
* For lack of direct relationship support, Spring Data R2DBC repositories require a different approach to domain and database table design.
* Spring Data MongoDB and Spring Data Cassandra offer a near-identical programming model for writing reactive repositories for MongoDB and Cassandra databases.
* Using Spring Data test annotations along with StepVerifier, you can test automatically created reactive repositories from the Spring application context.
 
---

* Spring Data 支持多种数据库类型的响应式持久化，包括关系型数据库（使用 R2DBC）、MongoDB 和 Cassandra。
* Spring Data R2DBC 提供了关系型持久化的响应式选项，但尚不直接支持直接持久化域类中的关系。
* 由于缺乏直接的关系支持，Spring Data R2DBC 存储库需要一种不同的方法来设计域和数据库表。
* Spring Data MongoDB 和 Spring Data Cassandra 为编写 MongoDB 和 Cassandra 数据库的响应式存储库提供几乎相同的编程模型。
* 使用 Spring Data 的测试注解以及 StepVerifier，您可以测试从 Spring 应用程序上下文中自动创建的响应式存储库。
