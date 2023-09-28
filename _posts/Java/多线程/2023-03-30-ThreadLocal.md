---
layout: post
title: 'ThreadLocal'
date: 2023-03-30 23:14 +0800
category: [Java,多线程]
tag: [多线程,ThreadLocal]
---
> 本篇博客中的一些内容(图片)并非原创，更严格的说本篇内容就是对于参考博文的自我消化。  
  credit: [一枝花算不算浪漫的博文](https://juejin.cn/post/6844904151567040519#heading-5) ❤️
 {: .prompt-info } 

## 数据结构

每一个Thread都有自己的ThreadLocalMap,  
逻辑上Map的key为ThreadLocal，value为存储在ThreadLocal中的值。  
形式上Map是一个Entry数组，Entry继承自ThreadLocal的<u>**弱引用**</u>，然后Entry的内部字段中保存了该ThreadLocal的对应存储的值。

补充: Entry数组在逻辑上是一个环形数组，这与其线性探查法的实现特性有关。

![ThreadLocal数据结构](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/java/多线程/ThreadLocal数据结构.1vchug9zjri8.webp)

定义Entry继承自ThreadLocal的弱引用是为了保证了当 ThreadLocal 实例没有被使用(没其他强引用关系)时会被垃圾回收器回收(弱引用的特性)，从而防止了内存泄漏的问题。

但是多线程的环境复杂，仅做这些工作寄托于GC还是会出现问题，例如经典的ThreadLocal被GC后，其对应的Value仍然存在的内存泄露问题 。所以Threadlocal还实现一些清理方法用于清理key已经被GC的Entry，这一点单独在后续的内存泄露处理一小节中进行具体说明。

一个ThreadLocal的内存泄露的模拟 -> [Github](https://github.com/TonyMarsh31/Java-playground/blob/f12fc89ba1e5f025d79cd163838190a65b7e6ded/src/main/java/%E5%A4%9A%E7%BA%BF%E7%A8%8B/ThreadLocal/ThreadLocalTest.java)

### hash算法

Thread用的是黄金分割(斐波那契数)的Hash算法。

```java
public class ThreadLocal<T> {
    private static AtomicInteger nextHashCode = new AtomicInteger();
    private static final int HASH_INCREMENT = 0x61c88647;
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
  
   private final int threadLocalHashCode = nextHashCode();
}
```

不谈HASH_INCREMENT这个魔法数字的原理，这个hash算法的实现非常简单  
就是从0开始，每一个新hash等于上一个hash加上这个魔法数字。

### 冲突处理(线程探查法的解释)

ThreadLocalMap不同于HashMap的结构，ThreadLocalMap的Map实现没有使用拉链法而是线性探查法。    
即当出现hash冲突时，会向不断向后探查直到发现一个新的可插入的空位/或者当找到key相同的slut时进行更新操作。

补充: 因为线性探查法的特性，所以Entry数组在逻辑上是一个环形数组。  
即不断向后探查的过程中若是到了数组尾部，则再从头开始继续探查。  
也因为这个特性，map内部还要实现方法保证总会有空位可以找到以防止 无限探查的死循环。

### Get

一般线性探查法的逻辑就不赘述了。

在探查过程中，如果遇到 key为null的entry，则直接进行一次探测式的过期数据处理。

### Set

Set的逻辑其实很简单，如下图所示

![ThreadLocalSetMethod](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/java/多线程/ThreadLocalSetMethod.6lmgy403h8w0.webp)

但是`ThreadLocalMap.set()` 的具体实现比较繁琐，因为涉及到了线性探查与后文会提到的为防止内存泄露而执行的清理方法的一些准备。线性探查的实现逻辑这里就不赘述了，这仅阐述当在线性探查时发现key为null的待清理Entry的处理逻辑。

> 这里可以提前说明的是，set操作之所以相较于get操作多了许多的逻辑，是因为set本身有update和insert之分，两者在有过期数据的情况下处理逻辑有更大的差异了。同时不同于get可以直接确定清理函数的起点，set中还需要实现这一复杂的方法
{: .prompt-tip }

---

![step1](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/java/多线程/step1.4in6fell5xu0.webp)

首先，当向后探查发现一个待清理节点时，以该节点的index来初始化一些状态: `slotToexpunge` `staleSlot` (理解这些参数的英文名会帮助理解，但这里因为想不出好的中文翻译，就干脆不翻译来防止先入为主了)

![step2](https://cdn.statically.io/gh/TonyMarsh31/image-hosting@master/Blog/java/多线程/step2.5gge26lnf0g0.webp)

接着，向前探查寻找是否有其他需要被清理的节点以更新`slotToexpunge`，直到探查到null  

> slotToExpunge是一个重要的参数，其代表了清理的起点位置，之后要传递给清理函数。StaleSlot只是一个中间变量用于完成接下来的set操作，以及在一些边界条件下进一步更新slotToExpunge的值
{: .prompt-tip } 

然后我们继续做set操作的逻辑，会有两种情况出现，一种是update操作，一种是insert操作  
如果是update，则节点更新后，与`staleSlot`做swap操作  
如果是insert，则直接覆写`staleSlot`  

> 若是对线性探查法的比较熟悉的话，就能敏感地察觉到上面这两个操作就是为了维护线程探查法的规则。因为马上就会调用清理方法把`slotToexpunge`开始的所有待处理节点清理掉，这也就意味着`staleSlot`在清理后会是一个空节点！若是set到了空节点之后的地方，那么将来的get操作将无法获取到set的值，因为按照线性探查法的规则，get在探查到null后便会停止探查而不会继续探查后面的节点
{: .prompt-tip} 

换句话说，新节点必定会被set在staleSlot初始化的位置(在源码中可以直接看到这一逻辑必定执行)，  
只是如果是update操作，则要做swap操作

有一个小细节是如果向后探查发现了新的待清理节点，同时`slotToExpunge == staleSlot`,那就需要更新·`slotToExpunge = i`(i为当前的index)，这是因为`stableSlot`再往前没找到，而原来的`staleSlot`肯定要被set为新的节点，所以`SlotToExpunge`应该为现在这个新找的待清理节点  
且swap操作后，也要检查`slotToExpunge == staleSlot`，是则更新`slotToExpunge = i`， 这种情况是探查的整个过程中，只有`staleSlot`一个待清理节点，则更新`slotToExpunge`为swap后的节点index。

最终set方法内部会在返回之前进行元素清理工作，具体的方法实现在后一节内容中。 

## 防止内存泄露的处理

### 探测式清理

`expungeStaleEntry()`

遍历散列数组，从参数传递的开始位置向后探测清理过期数据直至遇到空节点。

+ 沿途将所有的过期(key为null)的`Entry`整个设置为`null`
+ 沿途将所有正常的数据执行`rehash`

rehash就是字面意义上的再执行一次放入map中的逻辑，  即计算后放入hash与长度取模的位置，若冲突则线性探查。 

### 启发式清理

`cleanSomeSlots()`

见名知意，就是清理 **<u>一些</u>** 可清洁的节点，这个方法只是遍历一定次数的map，**不力求清洁全部的节.   
ThreadLocalMap在新添加一个元素或清除一个过期元素后都会判断是否需要resize.`cleanSomeSlots`方法就会在判断是否需要resize前进行调用

该方法有两个参数，  
第一个参数为起始扫描位置，注意与探测式不同的是，该位置是一个干净的节点  
第二次参数用于计算要遍历几次数组，传递参数n，则最终会遍历log2(n)次，一般这个参数为size

方法内部会统计这一次启发式清理是否清理出了元素，作为boolean类型的返回值

### 补充: 扩容操作中进行了清洁全部数据的操作

之前提到过，`cleanSomeSlots`方法就会在判断是否需要resize前进行调用。  
如果`!cleanSomeSlots(i, sz) && sz >= threshold` 就会执行一次全面的大扫除

>threshold是ThreadLocalMap的过载因子，其数值为数组长度的三分之二
{: .prompt-info}

大扫除具体来说就是遍历每一个节点，需要需要清洁的节点就执行一次探索式清理，此时如果大扫除之后，size仍然大于threshold的四分之三，那么就扩容为原来的两倍，然后rehash

>注意，size的两次比较的值是不同的，可以这么理解  
懒得做大扫除，只有*<u>大于阈值</u>*才做大扫除。  
但是做了大扫除之后尽量扩容，只要*<u>大于四分之三的阈值</u>*就扩容了。这样能防止频繁的大扫除 
{: .prompt-tip}

## InheritableThreadLocal

我们使用`ThreadLocal`的时候，在异步场景下是无法给子线程共享父线程中创建的线程副本数据的。  
为了解决这个问题，JDK 中还有一个`InheritableThreadLocal`类，[一个例子](https://github.com/TonyMarsh31/Java-playground/blob/97d15a4f54c32ec523de463206e7241ad1edb7f9/src/main/java/%E5%A4%9A%E7%BA%BF%E7%A8%8B/ThreadLocal/InheritableThreadLocalDemo.java)

`InheritableThreadLocal`的实现是依靠Thread构造器中获取了父线程的`InheritableThreadLocal`复制给本线程，这带来的一个缺陷是在一些线程复用的线程池场景中，这样做是没有用的。但阿里巴巴开源了一个`TransmittableThreadLocal`组件就可以解决这个问题，这里就不再延伸，感兴趣的可自行查阅资料。

## ThreadLocal的使用场景

1. 数据库连接和事务管理：在多线程应用程序中，每个线程通常需要独立的数据库连接以执行操作。ThreadLocal可以用于存储每个线程的数据库连接实例，保证线程间的连接不会相互干扰。此外，ThreadLocal还可以用于存储事务管理器，确保每个线程的事务操作独立进行。
2. 格式化工具类：一些格式化工具类（如SimpleDateFormat、NumberFormat等）在多线程环境中使用可能会导致线程安全问题。通过将这些工具类的实例存储在ThreadLocal中，可以确保每个线程使用独立的实例，避免线程安全问题。
3. 用户身份信息和会话管理：在Web应用程序中，可以使用ThreadLocal来存储用户的身份信息（如用户名、权限等），以便在整个请求处理过程中保持用户上下文。这样可以在不同的处理组件中轻松获取和使用用户信息，而无需显式传递。
4. 性能监控和日志记录：在一些需要监控和记录性能数据的场景中，ThreadLocal可以用于存储每个线程的性能计数器，以记录线程执行过程中的性能数据。同样，ThreadLocal还可以用于存储线程相关的日志信息，确保日志记录与特定线程相关联。
5. 避免参数传递：在某些场景下，某些数据需要在多个方法之间传递。这可能导致方法签名变得复杂且难以维护。通过将这些数据存储在ThreadLocal中，可以在方法之间轻松共享数据，而无需显式传递参数。

虽然ThreadLocal在多线程环境中非常有用，但需要注意避免内存泄漏。当线程结束时，需要确保清理ThreadLocal中的数据，以防止无效数据占用内存。此外，ThreadLocal不适合用于存储大量数据，因为这可能导致每个线程占用过多内存。
