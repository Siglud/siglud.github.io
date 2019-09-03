---
layout: post
title:  "Java 源代码阅读之 DualPivotQuicksort（二）"
date:   2018-06-17 20:19:00 +0800
author: Siglud
categories:
  - Java
tags:
  - Java
comment: true
share: true
---

## long 类型的排序
基本上表现和int的方式是一模一样的

## short 类型的排序
这个表现就和 int 有很大不同了，因为考虑到short的总可能性是比较少的，所以它使用了 counting sort

```java
if (right - left > COUNTING_SORT_THRESHOLD_FOR_SHORT_OR_CHAR) {
    int[] count = new int[NUM_SHORT_VALUES];

    for (int i = left - 1; ++i <= right;
        count[a[i] - Short.MIN_VALUE]++
    );
    for (int i = NUM_SHORT_VALUES, k = right + 1; k > left; ) {
        while (count[--i] == 0);
        short value = (short) (i + Short.MIN_VALUE);
        int s = count[i];

        do {
            a[--k] = value;
        } while (--s > 0);
    }
} else { // Use Dual-Pivot Quicksort on small arrays
    doSort(a, left, right, work, workBase, workLen);
}
```
对于长度超过3200的数组，采用计数排序，计数排序的方式实现也很简单，构建一个count作为辅助存储空间，遍历目标array，在对应的位置上做+1，记录对应的数值的数量，然后再遍历count，计算出每个位置的值，然后根据value的值把对应的连续位置都置为这个值

如果长度长度小于快排阈值（286），这个流程和int的排序完全类似，也是一个47以下走插入，再往上走双标的一个过程

如果长度大于286，但是小于3200，它依然会使用改版的TimSort。（代码基本上也是原封不动的照抄过去的）

## char 类型的排序
类似上面对short的排序方式

## byte 类型的排序
又有一点不一样，对于大于29的数组它就是计数排序的，但是对于比它小的数组，它则是简单的使用了插入排序，可能因为Byte的总数量特别少吧，char和short总量是 1 << 16， 但是byte就只有 1 << 8，这样小的内存空间感觉用起来没有太多的压力吧。

## float 类型的排序
float又和其他的类型有点不一样了，最大的问题就是在于它是浮点类型，它甚至能用来表示正负无穷大（NaN），所以它先执行这个代码把所有的NaN挪到最后

```java
while (left <= right && Float.isNaN(a[right])) {
    --right;
}
for (int k = right; --k >= left; ) {
    float ak = a[k];
    if (ak != ak) { // a[k] is NaN
        a[k] = a[right];
        a[right] = ak;
        --right;
    }
}
```
然后对前面这一段非NaN的数值执行doSort这个真正的排序算法，方式也是差不多的，双标快排、插入排序、快排和TimSort

接下来就是比较恶心的，用来单独的处理正负0的问题的段落了，Java的Float是拥有正负两个0的，而且这两个0如果直接判断是否相等，那么它们是相等的，但是它们其实又并不相等，这样就很蛋疼了，代码中使用了

```java
while (left <= right && Float.floatToRawIntBits(a[left]) < 0) {
    ++left;
}
```
Float.floatToRawIntBits 这个方法来区别正0和负0（Float.floatToRawIntBits(-0.0f) == -2147483648）

```java
for (int k = left, p = left - 1; ++k <= right; ) {
    float ak = a[k];
    if (ak != 0.0f) {
        break;
    }
    if (Float.floatToRawIntBits(ak) < 0) { // ak is -0.0f
        a[k] = 0.0f;
        a[++p] = -0.0f;
    }
}
```
找到所有等于0.0f的值，然后用k作为指针去找这些值里面哪些是-0.0f，然后用p把对应数量的-0.0f挪动到这一堆0.0f的左边，就是最后这一段要做的事情

## double 类型的排序
double类型的排序完全类似float

## 小结
对于基础类型的排序基本上都看完了，总的来说还是挺简单的，方法和套路基本上也是类似的，特殊的方法只体现在
1.总的可能数量特别少的时候
2.一些非常异常的情况下（NaN、±0）