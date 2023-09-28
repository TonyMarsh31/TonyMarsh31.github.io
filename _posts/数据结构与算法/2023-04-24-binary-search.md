---
layout: post
title: "二分查找 - Binary Search 的编码细节"
date: 2023-04-24 19:41 +0800
categories:
  - 数据结构与算法
tags: [ 数据结构与算法, 二分查找 ]
math: true

---

二分查找的思想很简单，但是在细节上需要细心地处理。

## 二分查找

一个简单的二分查找框架如下:

```java
int binarySearch(int[] nums, int target) {
    int left = 0; 
    int right = nums.length - 1; // 注意

    while(left <= right) {
        int mid = left + (right - left) / 2;
        if(nums[mid] == target)
            return mid; 
        else if (nums[mid] < target)
            left = mid + 1; // 注意
        else if (nums[mid] > target)
            right = mid - 1; // 注意
    }
    return -1;
}
```

>二分查找的写法是非固定的，可以不按照模板中的那样对right初始化为len -1
>，之后的代码模板中就是将right初始化为length，但是修改后也需要在后续的代码中做多多少少的修改来确保算法的正常执行。
{: .prompt-tip }

一些主要的细节如下

1. 确定left 与 right   
   left一般init为首元素的index，但right可以init为nums.length，也可以init为Len -1.  
   两者的区别就是前者表示扫描的区间是闭合的`[left,right]` ，后者的区间是左闭右开的`[left,right)`
2. 循环的终止条件  
   一个判断终止条件是否正确的方法就是判断当跳出循环时，left right组成的区间内是否还有元素  
   例如案例模板终止时，可能的情况是left = right + 1,此时`[right + 1,right]`的区间是没有元素的，  
   而如果定义为 left < right ,此时跳出循环时 `[right ,right ]`中还包含了一个元素没有被查找
3. mid的计算细节   
   使用mid = left + (right - left) / 2;，而非简单的 mid = (left + right ) /2是因为 `left+right`计算结果可能会过大导致溢出。
4. left和right的更新需不需要+/- 1.  
   这一操作的目的是排除mid元素，而有些算法中不对mid进行+/-1 操作一般都是因为right是开区间
   简单来说就是搜索 `[left , mid - 1]` 还是`[left , mid)`的区别，

> 总的来说，二分查找的细节就是对于left right mid的更新，而其核心逻辑就是扫描区间的更新，即`[left,right]`的更新
{: .prompt-tip }

## 寻找左边界的二分查找

有时候数据中包含了多个targetNum，例如`nums = [1,2,2,2,3]`，`target` 为 2。尽量上述的二分查找代码可以找到2，但无法满足找到最左边/右边的targetNum的目标。

可以在原先框架的基础上做一点简单的优化来实现该函数。(找到target，若有多个找到左边界)

```java
int left_bound(int[] nums, int target) {
    int left = 0;
    int right = nums.length; // 注意

    while (left < right) { // 注意
        int mid = left + (right - left) / 2;
        if (nums[mid] == target) {
            right = mid;
        } else if (nums[mid] < target) {
            left = mid + 1;
        } else if (nums[mid] > target) {
            right = mid; // 注意
        }
    }
    if (left == nums.length) return -1; // 在极端条件下(非常大的target)，left会不断右移直至== right
    return nums[left] == target ? left : -1;
}
```

简单来说，上述改动就是在找到target后仍然进行搜索更新的逻辑。可以合并一些条件判断的处理以简化代码格式。

注意与前一个代码模板不同的是，right初始化为了nums.length，这意味着我们二分查找的区间是左闭右开的`[left,right)`,这也进一步解释了为什么right的更新不需要做 mid-1，因为right本身是不可及的，所以`[left,newMid)`本身就排除了mid

## 寻找右边界的二分查找

```java
int right_bound(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (nums[mid] < target) {
            left = mid + 1;
        } else if (nums[mid] > target) {
            right = mid - 1;
        } else if (nums[mid] == target) {
            left = mid + 1;
        }
    }
    if (left - 1 < 0) return -1;
    return nums[left - 1] == target ? (left - 1) : -1;
}
```

代码上的逻辑差的不是很多，但注意最后返回的是 nums[left - 1],这是因为在最后一次 nums[mid] == target的操作中，执行了left = mid + 1;
