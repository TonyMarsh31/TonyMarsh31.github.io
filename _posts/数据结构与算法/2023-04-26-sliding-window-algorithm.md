---
layout: post
title: "滑动窗口 - Sliding Window Algorithm"
date: 2023-04-26 17:36 +0800
categories:
  - 数据结构与算法
tags: [ 数据结构与算法, 滑动窗口 ]
math: true
---

## 思路

滑动窗口算法的思路十分简单，就是维护一个固定大小的窗口，然后在遍历数据的过程中不断调整窗口的大小，同时动态更新所需要计算的结果。

通常来说，滑动窗口算法应用于解决数组或列表中的连续子串或子数组问题，比如最长子串、最小子串、最长子数组、最小子数组等等。

滑动窗口算法的具体实现需要用到两个指针，即left和right指针，它们的范围初始时固定为数据的开始位置。然后，程序会尝试通过移动right指针，不断拓展窗口的大小，直到满足某一特定条件（如子串的和大于等于目标值、子串长度等）。

接着，程序会尝试通过移动left指针，收缩窗口的大小，直到不再满足上述条件，同时在收缩窗口的过程中，可以执行一些特定的操作，例如更新结果、记录子串等。

最后，重复步骤2-4，直到right指针到达数据的末尾，最终返回滑动窗口算法求解的结果。

第 2 步相当于在寻找一个「可行解」，然后第 3 步在优化这个「可行解」，最终找到最优解

```java
int left = 0, right = 0;
// left = right = 0,左闭右开区间

while (left < right && right < s.size()) {
    // 增大窗口 直至满足条件
    window.add(s[right]);
    right++;

    while (window needs shrink) {
        // 缩小窗口
        window.remove(s[left]);
        left++;
        // 在缩小窗口的过程中，可以执行一些特定的操作,例如更新结果、记录子串等
    }
}
```

> 一般来说，滑动窗口算法的时间复杂度为O(N)，其中N为数据的长度。因为left和right指针最多各移动N次。
{: .prompt-tip }

## 使用框架

一般不会对滑动窗口算法的思路有困惑，更多地是对于其实现过程中各种细节问题的处理感到困难。

例如何向窗口中添加新元素、如何缩小窗口、在窗口滑动的哪个阶段进行结果更新等问题。

这些问题的解决方法并不是固定的，需要根据具体的问题来确定。不过一个通用的技巧是，可以在滑动窗口算法的过程中，将窗口中的元素存储在一个数据结构中，例如哈希表或者优先队列，这样可以在O(
1)的时间复杂度内完成元素的添加和删除操作。

```java
void slidingWindow(string s) {
    // 用合适的数据结构记录窗口中的数据
    unordered_map<char, int> window;

    int left = 0, right = 0;
    while (right < s.size()) 
        // c 是将移入窗口的字符
        char c = s[right];
        winodw.add(c)
        // 增大窗口
        right++;
        // 进行窗口内数据的一系列更新
        ...

        // 判断左侧窗口是否要收缩
        while (left < right && window needs shrink) {
            // d 是将移出窗口的字符
            char d = s[left];
            winodw.remove(d)
            // 缩小窗口
            left++;
            // 进行窗口内数据的一系列更新
            ...
        }
    }
}
```
## 例题
[LeetCode中的滑动窗口技巧]({% post_url /LeetCode笔记/2023-04-26-sliding-window-algorithm-in-leetcode %})



