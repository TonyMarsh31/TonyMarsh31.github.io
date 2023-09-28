---
layout: post
title: LinkedList from scratch
date: 2023-03-24 00:15 +0800
category: [数据结构与算法, 自己实现简单数据结构]
tag: [LinkedList]
---

> 仅说明思路, code in my GitHub repository: [click here](https://github.com/TonyMarsh31/DataStructure/blob/master/%E6%95%B0%E7%BB%84%E5%92%8C%E9%93%BE%E8%A1%A8/MyLinkedList.java)
{: .prompt-tip }

# 双链表

## 数据结构定义

首先是Node了, `val` 以及前后`node引用`, 构造用来赋值value，前后node由链表维护   
然后是`size`，主要用于指定位置add的时候检查index的合法性    
接着是能帮上很多忙的`哨兵节点`(虚拟头尾检点)  
链表本身的构造中初始化头尾节点，互相指向对方，将size初始化为0

## 工具

检查索引合法性：ElementIndex | PositionIndex.  
ElementIndex  `(0,size)` |  PositionIndex `(0,size]`.   
前者在remove和get方法中使用 ，后者在add方法中使用.  
getNodeByIndex, 直接根据index获取node节点，然后进行引用操作完成CRUD，一个小的优化点就是判断`index` 与 `size >> 1` 即half size的大小决定从开还是从尾开始遍历。

## CRUD

有了工具函数之后CRUD的实现很简单，引用操作也没有难度，没有什么坑，但注意要维护size变量

## 其他实现 

迭代器里，维护p Node表示当前节点，  
`hasNext()` 用  `p!=tail`.  
`Next()` 中，保存p.value 用于返回，然后`p = p.next`

# 单链表

## 数据结构定义

Node中只有 `val` 和`next`引用了   
其他不变，头尾`两个哨兵节点`，维护`size`变量

## 工具

与双链表中的一致,`getNodeByIndex()`这次只能从头遍历了

## CRUD

主要的细节在引用的操作上

+ `add()` 中，要获得前一个节点，然后使用 `cur.next = pre.next; pre.next = cur;` 完成操作  
  注意上述语句的执行顺序不能改变，否认会丢失 `cur.next`  
  如果怕出错，可以先将 `pre.next` 存到局部变量中作为 `next`  ，然后`pre.next = cur; cur.next = next`就不会出错了 
+ `Remove()` 同样，要获得前一个节点，然后获取`cur`的值存储到局部变量用作方法返回值   
  remove本身的操作可以用 `pre.next = pre.next.next;`完成
+ `getSet`操作有 `GetNodeByIndex`之后就很好解决了
