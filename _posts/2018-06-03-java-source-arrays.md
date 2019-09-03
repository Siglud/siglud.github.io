---
layout: post
title:  "Java 源代码阅读之 Arrays"
date:   2018-06-03 19:44:00 +0800
author: Siglud
categories:
  - Java
tags:
  - Java
comment: true
share: true
---

## Arrays类Java Doc
Arrays类相当的长，相比ArrayList的不到1600行，Arrays作为一个工具类，有8K多行，而且还不算上专门为它服务的一些工具类（比如一些专门的排序类，比如DualPivotQuicksort、ArraysParallelSortHelpers等），可以说是相当大的一个类，所以可能也会分为好几次来写这个东西了。因为是工具类的原因，大部分方法都是static的。Java Doc里面重申了允许在算法上做出各种不同的改变，但是不允许改变一些核心的implementation notes，比如排序可以不必使用归并排序，但是需要遵循“稳定”这一原则。这也是除了少部分不可变类型的sort用了快排之外，其他的排序大都数都没有使用快排（代码中用双标快排比较多）的缘故

## 对基础数据类型的排序
```java
public static void sort(int[] a) {
        DualPivotQuicksort.sort(a, 0, a.length - 1, null, 0, 0);
    }
```
基本上很简单的重复代码，就不多说明了，对于int、long、short、char、byte、float、double这样的基础数据类型，都是一句话，直接用双标快排。因为双标快排是一个对于内存友好的排序方式，虽然它理论上的算法复杂度O(Nlog(N))略高于快排，但是依托于它对内存读取数量的减少，在现在CPU速度远大于内存读取速度的今天，它的实际排序速度要高快排

## 并行排序
```java
public static void parallelSort(byte[] a) {
        int n = a.length, p, g;
        if (n <= MIN_ARRAY_SORT_GRAN ||
            (p = ForkJoinPool.getCommonPoolParallelism()) == 1)
            DualPivotQuicksort.sort(a, 0, n - 1);
        else
            new ArraysParallelSortHelpers.FJByte.Sorter
                (null, a, new byte[n], 0, n, 0,
                 ((g = n / (p << 2)) <= MIN_ARRAY_SORT_GRAN) ?
                 MIN_ARRAY_SORT_GRAN : g).invoke();
    }
```
首先判断需要排序的数组长度是否小于等于8192，然后判断当前的forkJoin Pool的线程池是否等于1，满足任何一种情况就回归普通的双标快排，否则则用Forkjoin Pool来做分段的双标快排+归并排序，因为类型不同的原因，这重复的代码也是写了无数遍，接下来的对某一段区域的数组进行并行排序也是同样的代码再重复，除了对泛型的Object数组进行排序的时候，双标快排变成了TimSort

```java
public static <T extends Comparable<? super T>> void parallelSort(T[] a) {
        int n = a.length, p, g;
        if (n <= MIN_ARRAY_SORT_GRAN ||
            (p = ForkJoinPool.getCommonPoolParallelism()) == 1)
            TimSort.sort(a, 0, n, NaturalOrder.INSTANCE, null, 0, 0);
        else
            new ArraysParallelSortHelpers.FJObject.Sorter<>
                (null, a,
                 (T[])Array.newInstance(a.getClass().getComponentType(), n),
                 0, n, 0, ((g = n / (p << 2)) <= MIN_ARRAY_SORT_GRAN) ?
                 MIN_ARRAY_SORT_GRAN : g, NaturalOrder.INSTANCE).invoke();
    }
```

## 对Object对象进行排序
```java
public static void sort(Object[] a) {
        if (LegacyMergeSort.userRequested)
            legacyMergeSort(a);
        else
            ComparableTimSort.sort(a, 0, a.length, null, 0, 0);
    }
```
除非特殊要求才使用归并排序，否则使用TimSort，而这个特殊要求已经被指定为To be removed in a future release了


## JDK 10归并排序实现
```java
    private static void mergeSort(Object[] src,
                                  Object[] dest,
                                  int low,
                                  int high,
                                  int off) {
        int length = high - low;

        // Insertion sort on smallest arrays
        if (length < INSERTIONSORT_THRESHOLD) {
            for (int i=low; i<high; i++)
                for (int j=i; j>low &&
                         ((Comparable) dest[j-1]).compareTo(dest[j])>0; j--)
                    swap(dest, j, j-1);
            return;
        }

        // Recursively sort halves of dest into src
        int destLow  = low;
        int destHigh = high;
        low  += off;
        high += off;
        int mid = (low + high) >>> 1;
        mergeSort(dest, src, low, mid, -off);
        mergeSort(dest, src, mid, high, -off);

        // If list is already sorted, just copy from src to dest.  This is an
        // optimization that results in faster sorts for nearly ordered lists.
        if (((Comparable)src[mid-1]).compareTo(src[mid]) <= 0) {
            System.arraycopy(src, low, dest, destLow, length);
            return;
        }

        // Merge sorted halves (now in src) into dest
        for(int i = destLow, p = low, q = mid; i < destHigh; i++) {
            if (q >= high || p < mid && ((Comparable)src[p]).compareTo(src[q])<=0)
                dest[i] = src[p++];
            else
                dest[i] = src[q++];
        }
    }
```
对长度小于7的数组，直接进行插入排序。用带符号的右移位方式获取中值（之后可以看到这种方式求1/2的地方基本上处处可见），然后对需要排序的两个部分各自做一次归并排序（但是颠倒了目标dest和src，让排序结果直接体现在src中），这样通过递归处理，最后切出来的分块一定会小于INSERTIONSORT_THRESHOLD（也就是7），那么会进入插入排序，排序完成之后会回到上一步，对两个插入排序完毕的数组进行归并，首先它判断极限情况，就是当中间值的前一位（第一个归并排序的尾部）也小于第二个归并排序的头部的时候，认为此时两个序列本身已经是有序的，那么只做Copy就好，否则，进行真正的归并排序流程，用一个标志为i在两个序列中轮流寻找下一位应该被插入的对象，最后完成归并。

程序的两点应该就是那一步对既有已排序完成的状态的预判，代码简单速度也很快，但是对于很多已经基本排序完毕的情况下，这个会极大的提高速度

## 查找
```java
public static int binarySearch(float[] a, int fromIndex, int toIndex,
                            float key) {
    rangeCheck(a.length, fromIndex, toIndex);
    return binarySearch0(a, fromIndex, toIndex, key);
}

// Like public version, but without range checks.
private static int binarySearch0(float[] a, int fromIndex, int toIndex,
                                float key) {
    int low = fromIndex;
    int high = toIndex - 1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        float midVal = a[mid];

        if (midVal < key)
            low = mid + 1;  // Neither val is NaN, thisVal is smaller
        else if (midVal > key)
            high = mid - 1; // Neither val is NaN, thisVal is larger
        else {
            int midBits = Float.floatToIntBits(midVal);
            int keyBits = Float.floatToIntBits(key);
            if (midBits == keyBits)     // Values are equal
                return mid;             // Key found
            else if (midBits < keyBits) // (-0.0, 0.0) or (!NaN, NaN)
                low = mid + 1;
            else                        // (0.0, -0.0) or (NaN, !NaN)
                high = mid - 1;
        }
    }
    return -(low + 1);  // key not found.
}
}
```
直接调用的binarySearch0，这个binarySearch0就是一个标准的二分查找了，唯一可圈点的就是带符号的右移位取中间值。而刻意的摘录对float的比较，是为了看它在比较两个float的时候的特殊处理。因为float存在精度问题，因此一般我们会用Math::abs来比较，但是这个对于一个基础库来说是不合适的，也不如它这样处理得快


## 复制
```java
@HotSpotIntrinsicCandidate
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked")
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0,
                        Math.min(original.length, newLength));
    return copy;
}
```
很好理解的代码，问题是——那个@HotSpotIntrinsicCandidate是什么东西？看看它的注释

```java
/**
 * The {@code @HotSpotIntrinsicCandidate} annotation is specific to the
 * HotSpot Virtual Machine. It indicates that an annotated method
 * may be (but is not guaranteed to be) intrinsified by the HotSpot VM. A method
 * is intrinsified if the HotSpot VM replaces the annotated method with hand-written
 * assembly and/or hand-written compiler IR -- a compiler intrinsic -- to improve
 * performance. The {@code @HotSpotIntrinsicCandidate} annotation is internal to the
 * Java libraries and is therefore not supposed to have any relevance for application
 * code.
 .......
 **/
```
说明了这段代码在HotSpot下会被替换成手写的汇编代码以进行优化，对于泛型的Copy处理都用到了这个优化

# asList
```java
public static <T> List<T> asList(T... a) {
    return new ArrayList<>(a);
}
```
虽然是看源代码，但是源代码太长了就不摘录了，这里会发现它返回的是一个自己实现了AbstractList的ArrayList，而不是调用的同级下面的ArrayList，比如它获取Size的方法就不是将Size缓存下来的，而是直接拿的length


## 小结
代码虽然很长，但是实质内容却没有想象中的那么多，大部分都是因为类型的原因而重复的代码。而实质性的代码都被挪动到了一些辅助类里面，而某些辅助类命名还真是挺烂的，说的就是你DualPivotQuicksort！明明叫双标快排的，但是感觉更加应该命名为ArraysSortHelpers，不过可能更多的是来自历史的遗留吧，毕竟命名显得更加规范的ArraysParallelSortHelpers类是1.8才加入的