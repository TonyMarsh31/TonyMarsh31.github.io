---
layout: post
title: Arraylist from scratch
mermaid: true
category: [ 数据结构与算法, 自己实现简单数据结构 ]
tag: [ Arraylist ]
---

> 仅说明思路 code in my GitHub repository: [click here](https://github.com/TonyMarsh31/DataStructure/blob/master/%E6%95%B0%E7%BB%84%E5%92%8C%E9%93%BE%E8%A1%A8/MyArrayList.java)
{: .prompt-tip }

底层数据容器 数组  `T[] data`  
额外数据结构： `size`: 表示目前已存储的元素个数

## 工具函数:

+ `System.arraycopy()`   
  jdk提供的高效数组移动方法  
  参数 ： 1源 2位置 3目标 4目标位置 5长度
  ```java
  // 做add时的搬移，移出来一个空位
  System.arraycopy(data, index, data, index + 1, size - index);
  
  // 做remove时的搬移，因为删了index处的元素，所以搬移的size-1
  System.arraycopy(data, index + 1, data, index, size - index - 1);
  ```

+ 索引合法性检查

  ElementIndex为 `[0,size)` PositionIndex为`[0.size]`  
  注意比较的对象是size，而不是data.length，两者的概念不要搞混  
  在做add 的时候，判断入参是否满足PositionIndex范围  
  在做remove的时候，则判断入参是否为ElementIndex范围  
  数组元素从0开始计数，所以不能等于size，最大为size-1  
  而Position可以等于size，即此时等于指向数组尾部的空位

+ resize
  resize就是新创建一个数组，长度为入参，然后用System.arraycopy  
  注意做个参数判断，若新的长度比size小就不执行，因为不一定每次都是扩容，也有可能做shrink  

+ 是否要resize判断  -> ensureCap()  
  一般在每次做add或remove的时候先做该判断  
  Size == data.length? resize cap * 2  
  Size < cap /4  && cap /2 > 0 ？  resize cap/2

## Crud

先checkIndex , 然后ensureCap ,然后做数据操作  
添加则先搬移，后插入  
remove则先取出，后搬移  
get和set不需要做ensureCap，只需要做index检查  
注意！，记得维护size变量做自增或自减

## 迭代器实现
内部维护变量p标志当前迭代到的位置，初始为0  
hasNext ->  return p != size  
next -> return data[p++]
