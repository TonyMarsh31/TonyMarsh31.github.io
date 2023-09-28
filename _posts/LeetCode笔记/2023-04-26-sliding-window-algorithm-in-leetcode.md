---
layout: post
title: "LeetCode - 滑动窗口技巧"
date: 2023-04-26 17:54 +0800
categories:
  - LeetCode笔记
tags: [ LeetCode笔记, 滑动窗口 ]
math: true
---

>写在最前，所有的滑动窗口问题都是灵魂三问：何时扩大窗口，何时缩小窗口，何时更新结果。理清这个框架，剩下的就是细节问题了。
{: .prompt-tip }

## 字符串区间问题

### 76 [最小覆盖子串](https://leetcode.com/problems/find-k-closest-elements/)

使用两个HahsMap<Character,Integer>统计字符出现的频率 ,分别作为window和need表示窗口区域和作为目标条件。初始化need后开始遍历String进行滑动窗口操作。

加入新字符到windows区间后，更新windowMap。 可以优化为只有新字符是need中的字符时才更新windowMap，因为不在need中的字符不会影响最终结果。

接着判断是否需要shrink，使用一个valid作为windows区间内满足need条件的字符个数，当valid==need.size()
时才尝试shrink。本题在shrink时更新result，result是最小区间的左右index。

shrink方法中先update result再执行shrink操作，因为shrink操作是先移动left指针，再更新windowMap，所以先更新result再shrink。

总之，滑动窗口是一个简单理解，但是实现起来比较复杂的算法，本题的一个洞见是使用HashMap做字符频率统计，一个注意点是shrink操作的顺序。

> window本身并不一定作为存储数据的结构，因为window区域的划定通过left和right指针即可实现，像本题中，windows是一个统计left ~
> right区域中字符出现频率的工作
{: .prompt-tip }

### 567 [字符串的排列](https://leetcode.com/problems/permutation-in-string/)

和76基本上一直的代码框架，调整一些条件判断即可。不做赘述了

### 438 [找到字符串中所有字母异位词](https://leetcode.com/problems/find-all-anagrams-in-a-string/)

同样的，所谓的异位词就是重排列的意思，仍然使用同样的套路模板很容易就可以做出来了.

### 3 [无重复字符的最长子串](https://leetcode.com/problems/longest-substring-without-repeating-characters/)

这一题更加简单，不需要额外的needMap就行，当然有一些更加复杂的解法进行优化，但是本节笔记的主题是滑动窗口，所以不多做赘述。



## 数组区间问题

### 1658 [将 x 减到 0 的最小操作数](https://leetcode.com/problems/minimum-operations-to-reduce-x-to-zero/)

这一题的难点在于转换思路，将题目转为另一种形式（区间问题）,即求最长的子数组，使得子数组的和为sum(nums) -
x。这样就可以使用滑动窗口的思路来解决问题了。

计算windowSum，如果不足则扩大，如果大于则缩小，如果等于则更新result。

> 注意，这道题在转换为求最长子数组和为sum(nums) - x后，可以用前缀和 + Map 来解决，两者的时间复杂度都是O(n)，但是前缀和需要额外空间。
> 但是还有一个转折，就是前缀和能应对源数据中有负数的情况，而滑动窗口只能应对源数据中都是正数的情况。也因为这道题中说明了元素均为正数，所以可以使用滑动窗口。否则使用前缀是一种通用的解法。
{: .prompt-tip }

### 713 [乘积小于K的子数组](https://leetcode.com/problems/subarray-product-less-than-k/)

这一题的区间条件是，子数组的乘积小于K，所以可以使用滑动窗口的思路来解决问题了。

当window乘积小于K时，更新result，然后扩大window，当window乘积大于等于K时，缩小window。
同样的，可以使用滑动窗口的前提是源数据中都是正数，所以可以肯定扩大窗口的时候，窗口内的乘积一定是增加的，缩小窗口的时候，窗口内的乘积一定是减少的。
否则还是使用前缀和(积)技巧 + Map的通用解法

一个小地方在于，当找到一个新的window时，此时result更新的个数不是 +=1 ,而是 += right - left,   
因为此时windows的所有包含最新一个元素 `nums[right]`的子区间都是满足条件的，其个数正好为length = right - left。(
如果不包含最新元素，那么就一定已经在之前的循环中被计算过了)
例如[1,2,3,4] 满足，则[4],[3,4],[2,3,4],[1,2,3,4]都满足

### 219 [存在重复元素 II](https://leetcode.com/problems/contains-duplicate-ii/)
区间的要求是: 存在重复元素，且两个重复元素的index差值小于等于k。使用滑动窗口的思路来解决很方便。

那么还是三问： 当窗口大小大于K时，扩大，当窗口大小等于K时，检查是否存在重复元素，是则直接返回，否则缩小窗口，让算法继续执行。

当然，具体的代码中一些地方有优化，例如我们的算法中窗口大小始终是小于等于K的，所以在新加入元素为重复元素时，可以直接返回true。

### 220 [存在重复元素 III](https://leetcode.com/problems/contains-duplicate-iii/)
219的升级版，要求区间内存在两个值的差值小于等于t，同时index的差值小于等于k。

思路：当窗口大小小于K时，扩大以包含更多元素，当窗口大小<= k时，判断是否存在两个值的差值小于等于t。

那么还有一个问题是，如何在window中快速判断是否有两个元素之差<= t,这里就用TreeMap提供的floorKey和ceilingKey方法来解决了。
其API的实现，JDK中是通过红黑树实现的，也可以用更简单的二叉堆实现，这里更具体的就不展开了。

## 替换问题

### 1004 [最大连续1的个数 III](https://leetcode.com/problems/max-consecutive-ones-iii/)

很直白的滑动窗口问题，搞清楚窗口的扩展和收缩条件即可。

当可替换次数大于等于 0 时，扩大窗口，让进入窗口的 0 都变成 1，使得连续的 1 的长度尽可能大。
当可替换次数用完时，更新result，然后缩小窗口，空出新的可替换次数，用于继续算法.

当然，实际代码中，可以不用实现0替换为1的这种操作，而是只要计算窗口内0的个数，然后直接和K比较即可。

### 424 [替换后的最长重复字符](https://leetcode.com/problems/longest-repeating-character-replacement/)
类似的题目，此时的难点在于如何替换的问题。

要求的是最长，那么最优的方案是将其他字符替换为出现评率最高的字符，这样就可以使得窗口内的字符都是一样的，从而使得窗口内的字符个数最多。

同样的，一个技巧在于,***我们不需要真的执行替换操作，而是计算 需要进行替换的次数与K做比较即可。*** 即 right - left - countOfMostFeqChar > K


## 一个骚操作
### 395 [至少有K个重复字符的最长子串](https://leetcode.com/problems/longest-substring-with-at-least-k-repeating-characters/)
区间要求：子串中每个字符出现的次数都不少于K次。

在本题的场景中，我们想尽可能多地装字符，即扩大窗口，但不知道什么时候应该开始收缩窗口。   
比如窗口中有些字符出现次数虽然不满足 k，但有可能再扩大扩大窗口就能满足K了，所以无法根据窗口中字符出现次数来判断是否收缩窗口。

理论上讲，这种情况就不能用滑动窗口模板了，但有时候我们可以自己添加一些约束，来进行窗口的收缩。
题目说让我们求每个字符都出现至少 k 次的子串，我们可以再添加一个约束条件：求每个字符都出现至少 k 次，且仅包含 count 种不同字符的最长子串。  

```java
// 在 s 中寻找仅含有 count 种字符，且每种字符出现次数都大于 k 的最长子串  
int logestKLetterSubstr(String s, int k, int count) {
```

此时，我们就可以使用滑动窗口模板了，当窗口中字符类型大于 count 时，我们就收缩窗口，然后统计窗口中字符出现次数是否都大于 k，如果是，就更新结果。

接着，回来解决本题，我们只需要枚举 count，然后求出满足条件的最长子串即可。  
因为 s 中只包含小写字母，所以 count 的取值也就是 1~26，所以最后用一个 for 循环把这些值都输入 logestKLetterSubstr 计算一遍，求最大值就是题目想要的答案了。
滑动窗口算法的时间复杂度是 O(N)，循环 26 次依然是 O(26N) = O(N)。
