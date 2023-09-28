---
layout: post
title: 'CompletableFuture 使用笔记'
date: 2023-04-02 15:30 +0800
category: [ Java,多线程 ]
tag: [ 多线程,CompletableFuture ]
---
> credit: [CompletableFuture入门](https://javaguide.cn/java/concurrent/completablefuture-intro.html) 
{: .prompt-info }

## 介绍

`CompletableFuture` 同时实现了 `Future` 和 `CompletionStage` 接口。  
这意味着`CompletableFuture` 除了提供 `Future` 特性之外，还提供了函数式编程的能力，可以方便地管理与进一步处理任务。

## 创建

### 使用new关键字

注意使用这种方法创建的`CompletableFuture`对象是一个空的对象，其内部没有定义任何操作，且无法进行初始化与执行任务。通过这种方式创建的 `CompletableFuture` 对象一般作为中间容器，用于直接用`complete()`方法存储其他方法的运行结果。

例如用于存储一些RPC调用的结果

```java
CompletableFuture<RpcResponse<Object>> resultFuture = new CompletableFuture<>();
resultFuture.complete(rpcResponse); 
```

> 注意，complete()只能被有效执行一次，即只要执行一次后，future.get()的结果就固定了，之后不论是使用接下来提到的then操作或者再次执行complete()方法，调用get()方法时获取到的结果都是第一次complele的结果
{: .prompt-info }

简而言之，直接用new关键字创建的CompletableFuture无法定初始化与执行任务，只能直接对任务结果进行赋值。

补充： 上述示例代码中的操作可以用静态方法`completedFuture()`内联为

```java
CompletableFuture<String> future = CompletableFuture.completedFuture("result");
```

### 使用静态工厂方法

基于 `CompletableFuture` 自带的静态工厂方法：`runAsync()` 、`supplyAsync()` 。

使用静态工厂方法创建的CompletableFuture会立即运行参数接口中定义的任务。两个方法的区别在于前者的参数是Supplier函数式接口，后者是一个Runnable接口。具体来说，前者执行的任务有返回值，后者无返回值

工厂方法还有可以添加Executor参数来制定创建的CompletableFuture执行的异步任务在哪一个线程池中执行。不指定的话就在一个默认的线程池中运行(在生产环境下不推荐使用，应该手动指定一个线程池)

### 补充 ：异步任务

在之前的静态工厂方法中出现了Async关键字，其表示运行的任务是一个异步任务，其会在非本线程(默认或指定的线程池的线程中)执行。在之后的进一步then逻辑方法中也有相同的Async与否的区别。

基于该特性，一个非Async的同步方法会阻塞后面的方法  
而一个Async方法会在其他线程中执行，所以不会阻塞后续的操作。  

在函数式编程的风格中，我们可以使用流式编程的技巧(也称链式编程)，使用 `.` 运算符将多个操作作为流水线处理  
例如

```java
 CompletableFuture.supplyAsync(() -> "doing something first")
                .thenApply(s -> s + " and then do something else") 
                .thenAccept(System.out::println);
```

注意，在一条流水线(链条)上的操作，无论使用Async与否，都会在同一个线程中阻塞执行操作。  
即上述的代码中，就算使用了thenApplyAsync，因为是链式操作，这个函数还是会在链条最头部的线程中执行操作。

与之对应的示例如下

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "doing something first");
future.thenRunAsync(() -> System.out.println("doing something else"));
future.thenRunAsync(() -> System.out.println("doing something else"));
```

此时没有使用链式操作，则使用Async会在不同的线程中进行操作。

## 函数式操作

### 对结果/中间结果 的进一步处理

之前的代码示例中用到了各种各样的 以then开头的方法，这里进行统一的介绍

当我们获取到异步计算的结果（或中间结果）之后，还可以对其进行进一步的处理，比较常用的方法有下面几个：

> 这些方法都可以加上Async后缀来进行进行异步操作,且使用Async时，可以添加一个Executor参数表示在指定的线程池的线程中执行异步操作。
{: .prompt-info }

- `thenApply()`.  
  参数传递一个Function函数式接口，即接受前一个操作的结果做参数，同时返回一个新的操作结果。
- `thenAccept()`  
  参数传递一个Comsumer函数式接口，即接受前一个操作的结果做参数，执行一个无返回值的操作。
- `thenRun()`  
  参数传递一个Runnable函数式接口，不接受参数，执行一个无返回值的操作。
- `whenComplete()`  
  参数传递一个BiComsumer函数式接口，内部接受的两个参数一个是前一个操作的返回值，一个是(若有的话)异常对象，执行一个无返回值的操作。  
  正如其名，当执行whenComplete方法的时候，其只是添加了一个处理器，只有当Future处于完成阶段时才会处理执行逻辑。(`complete()` 、`completeExceptionally()` 、 `cancel()`等操作都视作为完成)

> 虽然whenComplete()虽然可以接触到Exception对象，但对于异常的处理有更加专门的做法下在下一节中会介绍。whenComplete()通常用于在异步任务完成后执行一些必要的清理资源、释放锁或日志记录等操作，而不需要关心异步任务是否成功或失败。
{: .prompt-tip }

### 异常处理

除了`whenComplete()` 之外，以下两个方法也能接触到Exception对象

+ `handle()`  
  参数一个BiComsumer，可以接受上一个操作的结果与(若有的话)异常对象
+ `exceptionally()`    
  参数是一个Function函数接口，接受一个异常对象

注意，`handle()`方法无论出现异常与否，其都会执行。(当无异常发时，其接受到一个null作为异常对象)  
而`exceptionally()`只有当出现Exception对象时才会执行

---

除了上面的异常处理，还有一个方法直接将一个异常对象作为任务的最终结果  
 `completeExceptionally()` ，其方法参数是一个`Exception`对象，  
该方法将Future对象标记为完成，而其任务的结果(get的返回值)是其方法传递的那个`Exception`对象  

## 多任务管理

### 组合任务

之前的函数式操作都是针对于同一个CompletableFutre对象，而在一些场景中，我们有多个不同的异步任务，其之间是有一定相互关系的，例如需要将一个任务的结果传递给另一个任务或者将多个任务的结果组合后再传递给另一个任务。

使用`thenCompose()` 或 `thenCombine()`就可以实现上述提到的操作，其对两个CompletableFuture对象进行组合   
(同样的，可以使用Async执行异步的组合操作)

Compose和Combine的区别在于: 

> 写完之后发现有些繁琐，可以直接看代码也许就心领神会了
{: .prompt-info}

+ 逻辑上：  
  Compose强调一种先后顺序，第一个任务的结果作为条件传递给第二个任务使用。即只有第一个任务成功完成后，才会继续执行compose的第二个任务。而Combine()方法中，两个CompletableFuture对象是同时执行的，无论哪一个抛出异常，结果都会被处理。
+ 方法参数不同：  
  Compose()方法接收一个Function类型的参数，它接收第一个CompletableFuture对象的结果并返回一个新的CompletableFuture对象。而combine()方法则接收一个BiFunction类型的参数，它接收两个CompletableFuture对象的结果并返回一个新的CompletableFuture对象。
+ 返回结果可能不同：  
  Compose()方法的返回结果是一个新的CompletableFuture对象，它的结果类型可能与输入的CompletableFuture对象不同，因为第二个CompletableFuture对象的执行结果会影响最终结果。而combine()方法的返回结果类型通常与输入的CompletableFuture对象相同，因为它只是对两个对象进行了简单的操作。

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> 10);
CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> 20);

CompletableFuture<Integer> composedFuture = future1.thenCompose(result1 ->
        CompletableFuture.supplyAsync(() -> result1 + 10));
System.out.println(composedFuture.get());  // 输出 20

CompletableFuture<Integer> combinedFuture = future1.thenCombine(future2, (result1, result2) ->
        result1 + result2);
System.out.println(combinedFuture.get());  // 输出 30
```


### 并行任务

你可以通过 `CompletableFuture` 的 `allOf()`这个静态方法来并行运行多个 `CompletableFuture` 。

实际项目中，我们经常需要并行运行多个互不相关的任务，这些任务之间没有依赖关系，可以互相独立地运行。

比说我们要读取处理 6 个文件，这 6 个任务都是没有执行顺序依赖的任务，但是我们需要返回给用户的时候将这几个文件的处理的结果进行统计整理。像这种情况我们就可以使用并行运行多个 `CompletableFuture` 来处理。

```java
CompletableFuture<Void> task1 = CompletableFuture.supplyAsync(()->{ 自定义业务操作 });
......
CompletableFuture<Void> task6 = CompletableFuture.supplyAsync(()->{ 自定义业务操作 });
CompletableFuture<Void> headerFuture=CompletableFuture.allOf(task1,.....,task6);
headerFuture.get();
System.out.println("all done. ");
```

---

经常和 `allOf()` 方法拿来对比的是 `anyOf()` 方法。

从字面上来说就很好理解了，`allOf()` 方法会等到所有的 `CompletableFuture` 都运行完成之后再返回
而`anyOf()` 方法不会等待所有的 `CompletableFuture` 都运行完成之后再返回，只要有一个执行完成即可！
