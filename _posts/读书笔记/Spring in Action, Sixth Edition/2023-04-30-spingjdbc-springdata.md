---
layout: post
title: "Spring中的数据持久化处理-SpringData"
date: 2023-04-30 01:04 +0800
category: [ 读书笔记, "spring实战第六版" ]
tag: [ SpringJDBC,SpringData ]
---

> 本文旨在强调个人觉得重要的部分，而省略一些基础的准备工作，例如准备数据库等内容。如需了解更为详尽的信息，请参考原著。  
> 以及本文并非是关于在Spring架构中进行持久化处理的最佳实践总结，相较于Mybatis提供的解决方案而言，本节中的SpringDataJDBC是一种规范的low
> level的实现，而不是一种解决方案。而JPA是在思路上和Mybatis不同的一种解决方案，对于Mybatis和JPA两者的比价不再本文的讨论范围内。
{:.prompt-info}

> 本文中没有涉及到 复杂表关系的处理，例如一对多，多对多等，因为这些内容是关于数据库设计的，而不是关于持久化处理的。作者也没有在原著中做特别的说明。(TODO)
{:.prompt-info}

## 一些概念

在处理关系数据时，Java 开发人员有多个选择。两个最常见的选择是 JDBC 和 JPA。SpringData为这两个选择提供了支持。

具体来说，JDBC和JPA是一种规范的定义(JDBC在等级上比JPA更加low level)，而SpringData是对其一种实现。  
同时SpringData是一个大家族，其还对不同的数据库和规范有拓展实现：例如SpringDataRedis,MonoMongoDB等等。
本节主要讲的是关系型数据库的持久化处理，所以主要讲SpringDataJDBC和SpringDataJPA。

## RAW - 原生JDBC

只使用JDBC规范的话，一个sql操作需要获取sql连结，创建statement，执行sql，处理结果，关闭statement，关闭sql连接。这些操作都是重复的,同时还需要处理异常和手动关闭资源。

```java
@Override
public Optional<Ingredient> findById(String id) {
  Connection connection = null;
  PreparedStatement statement = null;
  ResultSet resultSet = null;
  try {
    connection = dataSource.getConnection();
    statement = connection.prepareStatement(
        "select id, name, type from Ingredient");
    statement.setString(1, id);
    resultSet = statement.executeQuery();
    Ingredient ingredient = null;
    if(resultSet.next()) {
      ingredient = new Ingredient(
        resultSet.getString("id"),
        resultSet.getString("name"),
        Ingredient.Type.valueOf(resultSet.getString("type")));
    }
    return Optional.of(ingredient);
  } catch (SQLException e) {
    // ??? What should be done here ???
  } finally {
    if (resultSet != null) {
      try {
        resultSet.close();
      } catch (SQLException e) {}
    }
    if (statement != null) {
      try {
        statement.close();
      } catch (SQLException e) {}
    }
    if (connection != null) {
      try {
        connection.close();
      } catch (SQLException e) {}
    }
  }
  return null;
}
```

所以Spring提供了一个JdbcTemplate来简化这些操作。  
但注意这里还没到SpringDataJDBC的部分，Spring提供的JdbcTemplate只是一个原生的JDBC操作的工具类,只是对于繁琐重复工作的简化,而没有其他的增强,
这点从其需要添加的依赖上也能看出来，使用JdbcTemplate需要添加的依赖是`spring-jdbc`，而非`spring-data-jdbc`
。（具体为`spring-boot-starter-jdbc`）

```java
@Override
public Ingredient findById(String id) {
  return jdbcTemplate.queryForObject(
    // 参数1 sql
    "select id, name, type from Ingredient where id=?",
    // 参数2 结果处理器
    new RowMapper<Ingredient>() {
      public Ingredient mapRow(ResultSet rs, int rowNum)
          throws SQLException {
        return new Ingredient(
          rs.getString("id"),
          rs.getString("name"),
          Ingredient.Type.valueOf(rs.getString("type")));
      };
    // 参数3 sql中的参数
    }, id);
}
```

以上就是使用JdbcTemplate的代码，可以看到，使用JdbcTemplate后，代码量大大减少，同时也不需要手动关闭资源了。

jdbcTemplate还有query,update等方法，这里就不具体展开了。主要的格式就是传三个参数:(预准备)sql语句，结果处理器，sql语句中的参数。
结果处理器可以预先写好然后用lambda调用来简化和重用代码。

---

使用JdbcTemplate看上去很不错，已经很简化了，但是在面对复杂的场景时，代码还是会变得很复杂，例如书中处理保存订单的例子中，其作为一个事务要同时保存3张表的信息。

代码就不贴了，可以看[原著](https://silvershaded.github.io/Spring-Save/Chapter-03/3.1-Reading-and-writing-data-with-JDBC/3.1.4-Inserting-data.html)
其中代码复杂度集中在对于sql语句的定义与参数传递，例如一个订单中包含了10多个信息，那么就需要定义10多个参数，同时还要保证参数的顺序与sql语句中的顺序一致。以及还有对于聚合对象的处理操作。
例如订单信息中还有商品信息，那么就需要额外的处理。

此时就可以看出， Spring 的 JdbcTemplate 使操作关系数据库比使用纯 JDBC
要简单的多。但即使使用JdbcTemplate，一些持久化任务仍然具有挑战性。尤其是在聚合中持久化嵌套对象时。要是有办法解决这个问题就好了。

## 使用Spring Data JDBC

步骤：

- 添加依赖
- 定义Repository接口，使其继承自`org.springframework.data.repository.CrudRepository`,泛型中指定第一个参数为实体类，第二个参数为主键类型
- 为实体类中添加@Table和@Id注解(如果属性名已经保持一致则可以省略)，并且为不匹配的属性名添加@ColumnName注解

接口中继承CrudRepository中定义了如save,findById,findAll,delete等方法，为实体类添加注解的步骤让Spring框架可以完成ORM的操作。
至此，经过上述步骤后，Spring在运行时会为接口动态代理实现方法,使其具有CRUD的能力,直接用就行了.

```java
public interface IngredientRepository extends CrudRepository<Ingredient, String> {}
// 在需要用的地方注入后直接使用CurdRepository中定义的方法即可
```

Spring Data JDBC
固然简化了持久化操作，但它并不是一个完整的解决方案。例如现在我们想要实现一些更复杂的，CrudRepository中没有的操作，例如根据name查询，根据type查询等等,那么就还是需要自己在接口中声明后，手动实现。
一些简单的拓展实现可以用@Query注解来实现，但是更复杂的还是需要自己写实现类才行。

## 使用Spring Data JPA

在使用步骤上和Spring Data JDBC一样，只是需要添加的依赖不同，Spring Data JPA需要添加的依赖是`spring-boot-starter-data-jpa`。
以及对于实体类的注解也不同，需要使用`javax.persistence`包下的注解。并需要配置一对多，多对一等关系。

重点在于，使用Spring Data JPA后，可以在接口中按照一定规则(JPA语法)来定义方法，Spring会自动实现这些方法，这样就不需要自己写实现类了。
这个规则一般是 findBy + 属性名 + 条件,具体还有很多规则，这里不具体展开了

一些例子有： `readOrdersByDeliveryZipAndPlacedAtBetween` 、 `findByDeliveryToAndDeliveryCityAllIgnoresCase`
等复杂的逻辑只要按照规则命名就可以了。

同时还有一点，除了上述用法,当一个方法的语句实在很复杂时，可以使用JPA提供的一个更高抽象等级的sql语言，JPQL，可以在接口中使用@Query注解来使用JPQL语句，

JPQL是一种面向对象的查询语言，它和SQL语句类似，但是操作的是对象，而不是表。JPQL语句的执行结果也是对象，而不是表的行数据。JPQL语句中的表名是类名，字段名是属性名，它们都是面向对象的，而不是面向数据库的。

例如
需要查询所有订单对应的客户的名称，可以使用以下 JPQL 语句：

```sql
SELECT o.customer.name FROM Order o
```

在这个 JPQL 语句中，o.customer 表示订单关联的客户实体对象，.name 表示客户实体对象的名称属性。 相应的 SQL 语句需要通过 JOIN
操作来实现：

```sql
SELECT c.name FROM order o JOIN customer c ON o.customer_id = c.id
```

## 总结

本文中使用到的工具在抽象层级上不断提高，从最底层的JDBC到JdbcTemplate，再到Spring Data JDBC，最后到Spring Data
JPA，每一层都在上一层的基础上提供了更高的抽象，使得代码更加简洁，更加易于维护。

回忆起自己曾经使用的MybatisPlus技术栈，其内部也是用到了SpringDataJdbc，但是没有按照JPA的规范实现,而是自己实现了一套规范。不过学到了JPQL这个新东西，也算是有所收获了。
