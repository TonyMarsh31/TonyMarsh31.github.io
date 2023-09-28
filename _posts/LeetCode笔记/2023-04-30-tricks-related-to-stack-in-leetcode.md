---
layout: post
title: "LeetCode - 栈相关的技巧"
date: 2023-04-30 20:39 +0800
categories:
  - LeetCode笔记
tags: [ LeetCode笔记, Stack, 单调栈 ]
math: true
---

## 热身

### [143. 重排链表](https://leetcode-cn.com/problems/reorder-list/)

> 此题的最优思路是先找到链表的中点(快慢指针)，然后将后半部分的链表反转，最后将两个链表交错合并。但是这里为了符合主题，我们使用栈来实现
{:.prompt-tip}

单链表要从尾部遍历，可以将Node元素都存入Stack，然后再pop即可。 其实难点不在Stack的部分，而是链表指针之类的细节问题,所以不赘述了.

### [225. 用队列实现栈](https://leetcode-cn.com/problems/implement-stack-using-queues/)

使用两个队列的做法在官方解没来得及看，一个队列的做法是将队列的元素依次出队，然后再入队，直到最后一个元素，这个元素就是栈顶元素。

### [155. 最小栈](https://leetcode-cn.com/problems/min-stack/)

在[自己实现数据结构 - 栈拓展]({% post_url /数据结构与算法/自己实现简单数据结构/2023-03-29-队列与栈 %})中已经说过了，这里不赘述了。

## 经典题目

栈的特点就是 先进后出，经典题目就是一些表达式运算、括号合法性检测等问题。下面列出几个使用栈的经典场景。

### [20. 有效的括号](https://leetcode-cn.com/problems/valid-parentheses/)

经典的括号匹配问题，使用栈来解决。遇到左括号就入栈，遇到<u>对应的</u>右括号就出栈，最后栈为空则合法，否则不合法。

### [150. 逆波兰表达式求值](https://leetcode-cn.com/problems/evaluate-reverse-polish-notation/)

逆波兰表达式求值，使用栈来解决。遇到数字就入栈，遇到运算符就出栈两个数字进行运算，然后将结果入栈，最后栈中只剩一个元素，就是结果。

## 单调栈套路

### 下一个更大/小 的元素

单调栈特性：每次新元素入栈时，栈中的元素都是单调递增或者单调递减的。单调栈的基本实现在[自己实现数据结构 - 栈拓展]({%
post_url /数据结构与算法/自己实现简单数据结构/2023-03-29-队列与栈 %})中也提到过。

由于Stack的特性，在求下一个更大/小元素的需求中，在initStack的时候需要将源数据倒着入栈的，这样在出栈时就是正序了。代码示例如下：

```java
Stack<Integer> s = new Stack<>(); 
// 倒着往栈里放
for (int i = n - 1; i >= 0; i--) {
    while (!s.isEmpty() && s.peek() <= nums[i]) { /
        s.pop(); //若比自己小，则pop掉
    }
    res[i] = s.isEmpty() ? -1 : s.peek(); // 更新下一个最大元素的值
    s.push(nums[i]); //push自己
}
```

如果是求下一个最小元素，那么条件改成<就行

### 上一个更大/小 的元素

把元素init的顺序有倒序改为正序即可

```java
Stack<Integer> stk = new Stack<>(); 
for (int i = 0; i < n; i++) {
  // 删掉 nums[i] 前面较小的元素
  while (!stk.isEmpty() && stk.peek() <= nums[i]) {
    stk.pop();
  }
  // 现在栈顶就是 nums[i] 前面的更大元素
  res[i] = stk.isEmpty() ? -1 : stk.peek();
  stk.push(nums[i]);
}
return res;
```

同理，如何求上一个更小的元素，条件改成< 即可

>
有一些使用单调栈的做法是让单调栈只存index，然后在出栈时，根据index取值，这样可以节省空间，不过代码会复杂一点。个人觉得这种做法的空间复杂度是不变的，离开场景谈具体空间优化没有意义，可以当做一个补充知识。个人为了方便写和读，还是使用元素值入栈的做法。   
> 但是有一些题目中，是要明确计算index的差值的(如接下来的739)，这种情况下，使用index入栈是比较好的做法。
{:.prompt-tip}

## 单调栈题目

### [1019. 链表中的下一个更大节点](https://leetcode-cn.com/problems/next-greater-node-in-linked-list/)

将链表(中的val取出)转换为数组，然后使用单调栈套路即可。

### [739. 每日温度](https://leetcode-cn.com/problems/daily-temperatures/)

计算：对于每一天，你还要至少等多少天才能等到一个更暖和的气温；如果等不到那一天，填 0

就是计算下一个更大元素的另一种表达,只是返回的不是element，而是index差.还有一个小细节是比较的是temperature,但是存入stack的时index.

### [496. 下一个更大元素 I](https://leetcode-cn.com/problems/next-greater-element-i/)

首先需要计算nums2中每一个元素的 下一个更大的元素，然后将其存入Map中，然后遍历nums1，从Map中取出对应的值即可。

### [503. 下一个更大元素 II](https://leetcode-cn.com/problems/next-greater-element-ii/)

和496 不同的是，这里的是循环数组。

解决方法也很简单，将数组长度翻倍，然后再遍历即可。要注意的是，由于长度变了，所以index是i对length取余数的值。

### [1944. 队列中可以看到的人数](https://leetcode-cn.com/problems/number-of-visible-people-in-a-queue/)

也许一开始看着图，会觉得难以下手，0号看得到124，但是要排除3，这该如何处理呢？ 其实单调栈的特性已经实现这一点了，当2号入栈时，3号就会被pop掉，此时栈内就是24了.

那么接下来就与[739. 每日温度](https://leetcode-cn.com/problems/daily-temperatures/)
类似了，只不过这里计算的index差值是单调栈Stack中的index差值，而不是源数据数组中的差值。

当然，这只是思路上的差异，具体在代码上，我们是无法直接计算Stack中的index差值的，所以我们具体的解法是维护一个count变量，每次单调栈pop的时候，count++，这样就可以间接计算出index差值了。
以及，还有一个小细节是，这道题特殊一点，最后一个比我们高的人也是能被看到的，所以最后还要打个补丁，如果Stack不为空，那么最后还要额外count++一次。

### [1475. 商品折扣后的最终价格](https://leetcode-cn.com/problems/final-prices-with-a-special-discount-in-a-shop/)

题目翻译过来就是找到下一个更小或等于的元素， 然后以 origin - nextSmaller 作为结果存入数组即可。

PS: 一个比较神奇的是，这道题的单调栈解法比暴力解法(最直接地遍历找下一个元素然后做减法)
还要慢，可能是因为单调栈的操作比较多，而且还要维护一个Stack，所以效率不如暴力解法。

---

### 复杂场景 [402. 移掉K位数字](https://leetcode-cn.com/problems/remove-k-digits/)

因为首先是移除，而不是移动，所以无法改变顺序,而不管怎么移除，最终的结果的位数是确定的，
所以相同位数的数字比较大小，那就是要尽量确保高位数字尽可能小，这样才能保证最终的结果尽可能小。

所以如果想让结果尽可能小，那么清除数字分两步：
1、先删除 num 中的若干数字，使得 num 从左到右每一位都单调递增。比如 14329 转化成 129，这需要使用到 单调栈技巧。
2、num 中的每一位变成单调递增的之后，如果 k 还大于 0（还可以继续删除）的话，则删除尾部的数字，比如 129 删除成 12。


### 套路升级 [901. 股票价格跨度](https://leetcode-cn.com/problems/online-stock-span/)

今天股票价格的跨度被定义为股票价格小于或等于今天价格的最大连续日数（从今天开始往回数，包括今天)

一个洞见就是，该天数，就是维护一个单调递增栈时，添加元素时被挤掉的元素个数。同时需要注意的是，这个被挤掉的个数是累加的。例如挤掉的元素之前也挤掉过其他元素，那么这些元素也要计算在内。

所以除了计算pop次数，还要将次数与当前元素绑定后添加到Stack中，这样后续计算的时候，才能做到累加次数。








