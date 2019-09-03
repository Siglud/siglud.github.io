---
layout: post
title:  "LeetCode的一道题 First Missing Positive"
date:   2018-12-22 19:11:00 +0800
author: Siglud
categories:
  - python
tags:
  - Python
  - Leetcode
comment: true
share: true
---

在Leetcode上看到了这么一题：

要求是找一个List中不存在的，最小的整数，但是却额外的要求了这么一段：

```
Your algorithm should run in O(n) time and uses constant extra space.
```

我最初的解决思路其实挺简单的，做一个类似计数排序的Set，每找到一个就把里面对应的点亮一个，最后再去找谁是第一个没有点亮的即可

思想是很简单的，写出来就变成了位操作了。

```python
class Solution:
    def firstMissingPositive(self, nums):
        """
        :type nums: List[int]
        :rtype: int
        """
        total = len(nums)
        bit_set = 3 << total
        flatter = (2 << (total + 1)) - 1
        for i in nums:
            if i > total:
                continue
            if i <= 0:
                continue
            bit_set |= 1 << (total - i)
        bit_set ^= flatter
        if bit_set == 0:
            return total + 1
        else:
            return total - len(bin(bit_set)) + 3
```

但是很奇怪的是，理论上位运算是非常的快的，但是我的结果却跑了56ms，甚至还不如下面这个代码（运行时间 44ms）：


```python
class Solution:
    def firstMissingPositive(self, nums):
        """
        :type nums: List[int]
        :rtype: int
        """
        if not nums:
            return 1
        total = len(nums)
        bit_set = set([])
        for i in nums:
            if i > total:
                continue
            if i <= 0:
                continue
            bit_set.add(i)
        for i in range(1, total + 2):
            if i not in bit_set:
                return i
```

下面这个就稍微慢一点，时间用了68ms

```python
class Solution:
    def firstMissingPositive(self, nums):
        """
        :type nums: List[int]
        :rtype: int
        """
        if not nums:
            return 1
        total = len(nums)
        bit_list = [False] * (total + 2)
        bit_list[0] = True
        for i in nums:
            if 0 >= i or i > total:
                continue
            bit_list[i] = True
        for index, i in enumerate(bit_list):
            if not i:
                return index
```

位运算就真的慢那么一点么？还是说因为我图方便用了一个bin函数？

把最后一句换成这个试试看？

```python
return total - len(format(bit_set, 'b')) + 1
```

变成60ms，更慢了……orz

直接迭代试试看呢？改成这样

```python
class Solution:
    def firstMissingPositive(self, nums):
        """
        :type nums: List[int]
        :rtype: int
        """
        total = len(nums)
        bit_set = 3 << total
        flatter = (2 << (total + 1)) - 1
        for i in nums:
            if i > total:
                continue
            if i <= 0:
                continue
            bit_set |= 1 << (total - i)
        bit_set ^= flatter
        if bit_set == 0:
            return total + 1
        else:
            for i in range(1, total + 1):
                if bit_set | (1 << (total - i)) == bit_set:
                    return i
```
这样确实快了不少，但是依然用了52ms

看来Python的位运算果然是没有太多的性能方面的优势

但是当然了，虽然是很节约，其实还是并不满足constant extra space这个条件的

就算是很节省，只需要1/8的空间，但是也并不是常数级别的，真正的解法应该是这样了：

有点类似 Java 针对Byte的原地排序计数排序策略

```python
class Solution:
    def firstMissingPositive(self, nums):
        """
        :type nums: List[int]
        :rtype: int
        """
        if not nums:
            return 1
        n = len(nums)
        i = 0
        while i < n:
            if nums[i] > 0 and nums[i] < n and nums[i] != i + 1 and nums[i] != nums[nums[i] - 1]:
                y = nums[i]
                nums[i], nums[y - 1] = nums[y - 1], y
            else:
                i += 1
        for index, i in enumerate(nums):
            if index != i - 1:
                return index  + 1
        return n + 1
```

当然了，这个虽然理论值很快，但是——嗯，其实速度还不如用set呢，速度为56ms——当然了，也不排除是因为CPU抖动的问题，因为我提交了第二次，它的运行时间又变成了48ms，所以，为什么用set这么快？Python怎么可能快得过C嘛