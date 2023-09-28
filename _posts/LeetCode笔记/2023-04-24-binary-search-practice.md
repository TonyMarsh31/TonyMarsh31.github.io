---
layout: post
title: "LeetCode - 二分查找技巧"
date: 2023-04-24 20:57 +0800
categories:
  - LeetCode笔记
tags: [ LeetCode笔记, 二分查找 ]
---

算法题中的二分查找技巧往往是作为一种通用的技巧出现，毕竟其本质就是让每次search之后能筛掉一半的数据。

## 一个小坑: 二分边界搜索 并不等同于 寻找最接近的元素

### [658. 找到 K 个最接近的元素](https://leetcode.cn/problems/find-k-closest-elements)

找到最接近的元素后，以该元素为中心，双指针扩散添加元素直至有k个之后即可。

思路说起来很简单，但注意实现的一些细节。

例如找到接近的元素这一操作的实现，如果元素本身就存在，那么直接二分查找到左边界即可。  
但是target可能不存在，所以还需要打一个补丁，因为有时候可能会错过那个最接近的元素。  
例如arr = [0,0,1,2,3,3,4,7,7,8]，target = 5，此时二分边界搜索找到的是arr[7]元素7,但是arr[6]元素4更接近5   
这是因为最后一次条件判断时，left = 6 right = 7 mid = 6 ,arr[mid] < target,所以left = mid + 1 = 7,退出循环

最终findClose打完补丁后是

```java
private int findClosest(int[] arr, int x) {
    int left = 0, right = arr.length;
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] >= x) {
            right = mid;
        } else left = mid + 1;
    }
    if (left == arr.length) left--; //极端情况下，left会不断右移直至越界，此时需要将left减一
    // 在理想情况下，即数组中存在x时，返回下标最小的x的下标
    if (arr[left] == x) return left;
    // 但x可能不存在于数组中，此时需要进一步判断,找到最接近x的数
    // 由于算法设计缺陷，可能会跳过最接近x的数，因此需要进一步判断
    int diffLeft = left > 0 ? Math.abs(arr[left - 1] - x) : Integer.MAX_VALUE;
    int diff = left < arr.length ? Math.abs(arr[left] - x) : Integer.MAX_VALUE;
    // 返回与目标值差距最小的元素
    return diffLeft <= diff ? left - 1 : left;
}
```

还有一点就是双指针扩散的时候注意边界条件，双指针应该指向新的元素用于判定  
一种做法是将 (left,right)作为开区间初始化并扩散.  
但是本案例中因为已经找到了最接近的元素，所以可以将 [left,right)作为左闭右开区间初始化并扩散.  
在左边右开区间中 ，right - left 就是区间个数.  
在两边均开的区间中， right - left + 1 才是区间个数.  

## 二维中的二分查找

### [74. 搜索二维矩阵](https://leetcode.cn/problems/search-a-2d-matrix)

主要工作是将二维表示的坐标mapping为index，例如 `(i, j)` 可以映射成一维的 `index = i * n + j`

实现 **get**(**int**[][] matrix, **int** index)函数后就可以直接根据index做二分查找了

### [240. 搜索二维矩阵 II](https://leetcode.cn/problems/search-a-2d-matrix-ii)

洞见是对于二维矩阵来说，搜索操作的起点不固定为左上角，

## 二分查找的核心 - 快速筛选元素

### [剑指 Offer 53 - I. 在排序数组中查找数字 I](https://leetcode.cn/problems/zai-pai-xu-shu-zu-zhong-cha-zhao-shu-zi-lcof)

统计一个数字在排序数组中出现的次数。

使用bs找左边界和右边界之后，右减左+1就是res

### [剑指 Offer 53 - II. 0～n-1中缺失的数字](https://leetcode.cn/problems/que-shi-de-shu-zi-lcof)

常规的二分搜索让你在 `nums` 中搜索目标值 `target`，但这道题没有给你一个显式的 `target`，怎么办呢？

其实，二分搜索的关键在于，**你是否能够找到一些规律，能够在搜索区间中一次排除掉一半**。这道题的关键是，你是否能够找到一些规律，能够判断缺失的元素在哪一边？

洞见就是 `nums[mid]` 和 `mid` 的关系，如果 `nums[mid]` 和 `mid` 相等，则缺失的元素在右半边，如果 `nums[mid]` 和 `mid`
不相等，则缺失的元素在左半边。

## 峰值元素

### [162. 寻找峰值](https://leetcode.cn/problems/find-peak-element)

寻找峰值可以不用考虑 `left` 和 `right`，单纯考虑 `mid` 周边的情况。

如果走势下行（`nums[mid] > nums[mid+1]`），说明 `mid` **本身就是峰值或其左侧有一个峰值**，所以需要收缩右边界（`right = mid`）；

如果走势上行（`nums[mid] < nums[mid+1]`），则说明 `mid` **右侧有一个峰值**，需要收缩左边界（`left = mid + 1`）。

因为题目说了 `nums` 中不存在相等的相邻元素，所以不用考虑 `nums[mid] == nums[mid+1]` 的情况

### [852. 山脉数组的峰顶索引](https://leetcode.cn/problems/peak-index-in-a-mountain-array)

与162类似，就是找Max峰值

## 一些较为复杂的确认边界题型

> 后续的题目，可以先画图，然后判断如何更新搜索范围就很直观了
> {:  .prompt-tip }

### [33. 搜索旋转排序数组](https://leetcode.cn/problems/search-in-rotated-sorted-array)

题目明确要对数时间复杂度，所以必须要bf，难点在于mid可能落在不同的地方需要分类讨论。

### [81. 搜索旋转排序数组 II](https://leetcode.cn/problems/search-in-rotated-sorted-array-ii)

## 复杂的场景题 - 二分查找的一般套路

有些具体的场景题目可能一开始看不出来要用二分查找，比如让你求吃香蕉的「最小速度」，让你求轮船的「最低运载能力」，但其实求最值的过程就是搜索一个边界的过程，此时就可以使用二分查找技巧。

一般来说，要从题目中抽象出一个自变量 `x`，一个关于 `x` 的函数 `f(x)`，以及一个目标值 `target`。同时，`x, f(x), target`
还要满足以下条件：

**1、`f(x)` 必须是在 `x` 上的单调函数（单调增单调减都可以）**。

**2、题目是让你计算满足约束条件 `f(x) == target` 时的 `x` 的值**。

如此便可判断可以使用二分查找的套路进行解决。

剩下的就是判断left 和right的初始化，以及找的是左边界还是右边界了

### L875 爱吃香蕉的珂珂

定义速度为 x 时，需要 f(x) 小时吃完所有香蕉。题目便转换为 在f(x) == target的x左边界

x作为二分查找的区间，其边界就是left和right的初始值，在本题中 left =0 right= 最多的一坨香蕉个数

### L1011 在 D 天内送达包裹的能力

类似的，定义运载能力为 x 时，需要 f(x) 天完成。题目便转换为 在f(x) == target的x左边界

运载能力最小为weight中的最大值，最大为所有weight的总值。

### L410 分割数组的最大值

仔细观察后，其实是和L1011是一模一样的题目 maxGroupCaptity越大，groupNumber越小

f(maxGroupCaptity) = m 的左边界，maxGroupCaptity的min是nums中的最大元素，max是所有元素和。
