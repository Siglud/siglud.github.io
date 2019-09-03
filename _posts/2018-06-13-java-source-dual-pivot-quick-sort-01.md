---
layout: post
title:  "Java 源代码阅读之 DualPivotQuicksort（一）"
date:   2018-06-13 21:21:00 +0800
author: Siglud
categories:
  - Java
tags:
  - Java
comment: true
share: true
---

## 双标快排
虽然文件名被取名为DualPivotQuickSort.java，但是内容却并不完全是双标快排的好嘛，应该是命名的时候太过随意了，先来看看它整体的构造

这个类一开始就被申命为final的，并且是私有构造函数，所有的方法都是静态方法，所以是一个典型的工具类

文件本身非常长，但是除了一些常量之外所有的方法只分为两种：一是sort，二是doSort，而长的原因就是因为对各种类型有不同的处理方式（虽然大多数时候是相同的）

## int 类型的排序
```java
// Use Quicksort on small arrays
if (right - left < QUICKSORT_THRESHOLD) {
    sort(a, left, right, true);
    return;
}
```
这个 QUICKSORT_THRESHOLD 为 286，也就是总长度小于286的时候使用另外一套排序模式，先来看看这个对小的数组的排序方法。
乖乖，光看长度就是300+行，铁定更不是它注释所说的简单的 Quicksort。

### 长度小于 47 的情况
首先看他针对更小的一个Array实例的选择
```java
if (length < INSERTION_SORT_THRESHOLD) {
    if (leftmost) {
        /*
            * Traditional (without sentinel) insertion sort,
            * optimized for server VM, is used in case of
            * the leftmost part.
            */
        for (int i = left, j = i; i < right; j = ++i) {
            int ai = a[i + 1];
            while (ai < a[j]) {
                a[j + 1] = a[j];
                if (j-- == left) {
                    break;
                }
            }
            a[j + 1] = ai;
        }
    } else {
        /*
            * Skip the longest ascending sequence.
            */
        do {
            if (left >= right) {
                return;
            }
        } while (a[++left] >= a[left - 1]);

        /*
            * Every element from adjoining part plays the role
            * of sentinel, therefore this allows us to avoid the
            * left range check on each iteration. Moreover, we use
            * the more optimized algorithm, so called pair insertion
            * sort, which is faster (in the context of Quicksort)
            * than traditional implementation of insertion sort.
            */
        for (int k = left; ++left <= right; k = ++left) {
            int a1 = a[k], a2 = a[left];

            if (a1 < a2) {
                a2 = a1; a1 = a[left];
            }
            while (a1 < a[--k]) {
                a[k + 2] = a[k];
            }
            a[++k + 1] = a1;

            while (a2 < a[--k]) {
                a[k + 1] = a[k];
            }
            a[k + 1] = a2;
        }
        int last = a[right];

        while (last < a[--right]) {
            a[right + 1] = a[right];
        }
        a[right + 1] = last;
    }
    return;
}
```
这个 INSERTION_SORT_THRESHOLD 为47，然后判断这个 leftmost ，也就是需要排序的这个字段是否位于整个的最左边（基本上意味着left=0，right>left)的含义，如果是的，那么做一个简单的插入排序，如果不是在最左边的时候，首先它找出最大的连续序列（升序），然后将left指向非连续序列的第一个元素。

第二个循环很长很难看，加上反复的使用++这种标识符，很蛋疼，但是一步步看也不难。

首先把k指向left（此时的left为第一个非连续序列的第一个，而不是最初的那个left了）， 然后把left + 1
把左边的这个值（k所指向的）设置为a1，它紧挨着的右侧设置为a2，调整a1和a2的顺序，使其能够满足 a1 > a2
然后判断这个时候比这个较大的值最为接近的值，把比这个值更大的值都扔到后面去，并且把a1放到最适合的位置，并保留下当时的前面一个指针为k
然后从当前的k往前继续找比a2更小的值，同样的操作，直到定为最为适合a2的值

这个过程可以是一个稍微节约了一点时间的插入排序，因为它以两个值为单位进行查找，通过判定这两个值的相对顺序，节约第二个值找到自己正确位置的时间。

最后那一点存在是因为之前的插入排序是两两的进行排序的，最后可能会碰到可怜的单身狗，只能为它单独处理一下，没必要刻意的去判断最后那个单身是否一定存在，反正如果它不存在，最后也只是循环一次而已，很快就退出了，于是没有刻意的对它做优化。


### 长度介于 47~286 的情况
回到长度没有这么短的时候（即286~47之间的时候）

```java
// Inexpensive approximation of length / 7
int seventh = (length >> 3) + (length >> 6) + 1;

/*
    * Sort five evenly spaced elements around (and including) the
    * center element in the range. These elements will be used for
    * pivot selection as described below. The choice for spacing
    * these elements was empirically determined to work well on
    * a wide variety of inputs.
    */
int e3 = (left + right) >>> 1; // The midpoint
int e2 = e3 - seventh;
int e1 = e2 - seventh;
int e4 = e3 + seventh;
int e5 = e4 + seventh;

// Sort these elements using insertion sort
if (a[e2] < a[e1]) { int t = a[e2]; a[e2] = a[e1]; a[e1] = t; }

if (a[e3] < a[e2]) { int t = a[e3]; a[e3] = a[e2]; a[e2] = t;
    if (t < a[e1]) { a[e2] = a[e1]; a[e1] = t; }
}
if (a[e4] < a[e3]) { int t = a[e4]; a[e4] = a[e3]; a[e3] = t;
    if (t < a[e2]) { a[e3] = a[e2]; a[e2] = t;
        if (t < a[e1]) { a[e2] = a[e1]; a[e1] = t; }
    }
}
if (a[e5] < a[e4]) { int t = a[e5]; a[e5] = a[e4]; a[e4] = t;
    if (t < a[e3]) { a[e4] = a[e3]; a[e3] = t;
        if (t < a[e2]) { a[e3] = a[e2]; a[e2] = t;
            if (t < a[e1]) { a[e2] = a[e1]; a[e1] = t; }
        }
    }
}

// Pointers
int less  = left;  // The index of the first element of center part
int great = right; // The index before the first element of right part
```
这段是在做双标快排的准备工作
1. 首先，寻找长度的1/7，用 (1/8 + 1/64)X + 1 来进行近似求值
1. 寻找到中间点，然后用1/7的长度往前后各找2个点，也就是把整个Array切成6份，几个端点从左到右依次被命名为e1~e5
1. 对找到的这几个端点进行插入排序，也就是将e1~e5按照顺序整理好

接下来进行一个选择来确定是否要使用双标快排
```java
if (a[e1] != a[e2] && a[e2] != a[e3] && a[e3] != a[e4] && a[e4] != a[e5])
```
也就是所有的值都不等的时候才会使用双标快排，我们先进入一下这个条件不满足的情况，即找到有一个标和另外一个是相等的

```java
int pivot = a[e3];

/*
    * Partitioning degenerates to the traditional 3-way
    * (or "Dutch National Flag") schema:
    *
    *   left part    center part              right part
    * +-------------------------------------------------+
    * |  < pivot  |   == pivot   |     ?    |  > pivot  |
    * +-------------------------------------------------+
    *              ^              ^        ^
    *              |              |        |
    *             less            k      great
    *
    * Invariants:
    *
    *   all in (left, less)   < pivot
    *   all in [less, k)     == pivot
    *   all in (great, right) > pivot
    *
    * Pointer k is the first index of ?-part.
    */
for (int k = less; k <= great; ++k) {
    if (a[k] == pivot) {
        continue;
    }
    int ak = a[k];
    if (ak < pivot) { // Move a[k] to left part
        a[k] = a[less];
        a[less] = ak;
        ++less;
    } else { // a[k] > pivot - Move a[k] to right part
        while (a[great] > pivot) {
            --great;
        }
        if (a[great] < pivot) { // a[great] <= pivot
            a[k] = a[less];
            a[less] = a[great];
            ++less;
        } else { // a[great] == pivot
            /*
                * Even though a[great] equals to pivot, the
                * assignment a[k] = pivot may be incorrect,
                * if a[great] and pivot are floating-point
                * zeros of different signs. Therefore in float
                * and double sorting methods we have to use
                * more accurate assignment a[k] = a[great].
                */
            a[k] = pivot;
        }
        a[great] = ak;
        --great;
    }
}

/*
    * Sort left and right parts recursively.
    * All elements from center part are equal
    * and, therefore, already sorted.
    */
sort(a, left, less - 1, leftmost);
sort(a, great + 1, right, false);
```

这段注释超多的，代码思路其实也和快排差不多，很容易理解的
1.首先把第三个设置为排序的轴
2.执行快排思路，把所有大于轴的扔到右边，把所有小于轴的扔到左边
3.对左边和右边这两个部分分别调用自己——（最后在长度缩减到一定程度时一定会退化成刚才分析的那个插入排序）

值得注意的是，它这个地方采用的是不用辅助数组的快速排序，所以小于轴的部分它是老老实实的去移动的，但是出现了大于轴的时候，它会立刻从右侧开始减去所有比轴大的部分并且移动右指针。然后把小于轴的右侧的这个值也同时挪动到左侧去，然后把这个找到的k指向的这个大于轴的值挪动到右侧空出来的这个位置上去。

另外它的注释中也提到了，为了避免浮点数所产生的迷之相等，应该避免对直接赋值，但是我不明白的是为毛要莫名的做这个赋值？你不写这个条件分支不行么

再返回来看看各个轴都不相等的情况，这个时候就是双标快排了，先看第一部分

```java
int pivot1 = a[e2];
int pivot2 = a[e4];

/*
    * The first and the last elements to be sorted are moved to the
    * locations formerly occupied by the pivots. When partitioning
    * is complete, the pivots are swapped back into their final
    * positions, and excluded from subsequent sorting.
    */
a[e2] = a[left];
a[e4] = a[right];

/*
    * Skip elements, which are less or greater than pivot values.
    */
while (a[++less] < pivot1);
while (a[--great] > pivot2);

/*
    * Partitioning:
    *
    *   left part           center part                   right part
    * +--------------------------------------------------------------+
    * |  < pivot1  |  pivot1 <= && <= pivot2  |    ?    |  > pivot2  |
    * +--------------------------------------------------------------+
    *               ^                          ^       ^
    *               |                          |       |
    *              less                        k     great
    *
    * Invariants:
    *
    *              all in (left, less)   < pivot1
    *    pivot1 <= all in [less, k)     <= pivot2
    *              all in (great, right) > pivot2
    *
    * Pointer k is the first index of ?-part.
    */
outer:
for (int k = less - 1; ++k <= great; ) {
    int ak = a[k];
    if (ak < pivot1) { // Move a[k] to left part
        a[k] = a[less];
        /*
            * Here and below we use "a[i] = b; i++;" instead
            * of "a[i++] = b;" due to performance issue.
            */
        a[less] = ak;
        ++less;
    } else if (ak > pivot2) { // Move a[k] to right part
        while (a[great] > pivot2) {
            if (great-- == k) {
                break outer;
            }
        }
        if (a[great] < pivot1) { // a[great] <= pivot2
            a[k] = a[less];
            a[less] = a[great];
            ++less;
        } else { // pivot1 <= a[great] <= pivot2
            a[k] = a[great];
        }
        /*
            * Here and below we use "a[i] = b; i--;" instead
            * of "a[i--] = b;" due to performance issue.
            */
        a[great] = ak;
        --great;
    }
}

// Swap pivots into their final positions
a[left]  = a[less  - 1]; a[less  - 1] = pivot1;
a[right] = a[great + 1]; a[great + 1] = pivot2;

// Sort left and right parts recursively, excluding known pivots
sort(a, left, less - 2, leftmost);
sort(a, great + 2, right, false);
```

这代码注释简直比我的文字还多，甚至还附了图解释它在干嘛，其实总的来说，他就在做一点很简单的工作，把小于第一个轴的数扔到左边，把大于第二个轴的扔到右边，然后把这两个轴的位置确定下来（一个在被调整之后的指针的右侧，一个在调整的指针左侧），然后对这些扔过去的大块和小块分别调用自己把他们排序完毕。

其实看懂了上面的快排的代码，这个其实和上面没有太多的区别（唯一就是对右侧的判断变成了pivot2）。就不说更多的细节了，代码进行到这里之后，理论上整个排序的状态应该是两头已经排序完成，中间还没排序完成的状态。

这时候又有一个选择，如果 左侧排序完毕的部分长度少于1/7(即代码中的e1)并且右侧排序完毕之后的长度也小于1/7，（要知道，上面的代码我们可是拿的第二个和第4个点作为轴的，出现这个情况是为啥呢？我们推测可能是太多的值都等于某一个数值了？或许这个值就是pivot1和pivot2）

于是它专门写了一个代码来把这些刚好等于pivot1和pivot2的值扔出来，让它们缩减到该到的位置上（因为之前的双标快排其实对相等的情况处理相当粗暴，如果两个值等于两边的标，它只是简单的归到中间去而已，并没有对它做更多的处理因为双标快排最不擅长处理的其实就是这两个标）

这个代码就不贴了，其实本质上和上面的双标快排的代码一模一样，只是把大于号换成了等于号，相当于是对着两个标相等的情况再单独的去做个处理，然后减小less和greater中间的范围。


### 长度大于等于 286 的情况
好，终于走出了小于286的情况了，接下来回到之前的代码，继续往下看，首先了解一下，接下来基本上就是Timsort了——或者说极其类似TimSort的归并排序（毕竟我们看了这么多，其实它对所有的包括插入排序、双标快排、快排都有各种魔改）

```java
/*
* Index run[i] is the start of i-th run
* (ascending or descending sequence).
*/
int[] run = new int[MAX_RUN_COUNT + 1];
int count = 0; run[0] = left;

// Check if the array is nearly sorted
for (int k = left; k < right; run[count] = k) {
    // Equal items in the beginning of the sequence
    while (k < right && a[k] == a[k + 1])
        k++;
    if (k == right) break;  // Sequence finishes with equal items
    if (a[k] < a[k + 1]) { // ascending
        while (++k <= right && a[k - 1] <= a[k]);
    } else if (a[k] > a[k + 1]) { // descending
        while (++k <= right && a[k - 1] >= a[k]);
        // Transform into an ascending sequence
        for (int lo = run[count] - 1, hi = k; ++lo < --hi; ) {
            int t = a[lo]; a[lo] = a[hi]; a[hi] = t;
        }
    }

    // Merge a transformed descending sequence followed by an
    // ascending sequence
    if (run[count] > left && a[run[count]] >= a[run[count] - 1]) {
        count--;
    }

    /*
        * The array is not highly structured,
        * use Quicksort instead of merge sort.
        */
    if (++count == MAX_RUN_COUNT) {
        sort(a, left, right, true);
        return;
    }
}
```
这个是前期的准备工作，Timsort的准备工作是要把既有的序列切成一个个的RUN，每个RUN可能是一个升序序列也可能是一个降序序列，这里就是用来干这个的

首先介绍变量：run这个数组是用来存放切开的位置的（所以它一开始初始化了MAX_RUN_COUNT + 1份，也就是最多允许切 MAX_RUN_COUNT 出来，这个值是67）；count是run的尾指针，k是指向数组a的指针

这段话其实就是在连续的值切片，然后将尾指针放入到run这个数组中去，并且把所有的子序列都调整为升序；并且对于调整过升序之后的序列再尝试做一次合并（如果逆序了之后能够和之前的序列继续保持升序的话）

但是如果切片的数量超过了67份的话，它就会退化为之前的双标快排

之后它需要处理一下两种特殊的情况，一个是当有0个RUN的时候，说明根本就没有走到需要切开的地方，本身就是有序的，自然是要退出的；另一个是只有一个RUN同时最后一个尾标志位大于需要排序的长度，说明了这个之前被切成了两个RUN，但是颠倒了顺序之后，它们又正好拼在一起成为了一个，这也是特殊情况，说明已经排序完毕了，可以退出。

然后针对一些特殊情况（比如最后一个RUN是一个全相等的RUN，或者一个单元素在末尾的RUN）给这样的RUN列表补上一个尾标

最后是合并操作

```java
// Determine alternation base for merge
byte odd = 0;
for (int n = 1; (n <<= 1) < count; odd ^= 1);

// Use or create temporary array b for merging
int[] b;                 // temp array; alternates with a
int ao, bo;              // array offsets from 'left'
int blen = right - left; // space needed for b
if (work == null || workLen < blen || workBase + blen > work.length) {
    work = new int[blen];
    workBase = 0;
}
if (odd == 0) {
    System.arraycopy(a, left, work, workBase, blen);
    b = a;
    bo = 0;
    a = work;
    ao = workBase - left;
} else {
    b = work;
    ao = 0;
    bo = workBase - left;
}

// Merging
for (int last; count > 1; count = last) {
    for (int k = (last = 0) + 2; k <= count; k += 2) {
        int hi = run[k], mi = run[k - 1];
        for (int i = run[k - 2], p = i, q = mi; i < hi; ++i) {
            if (q >= hi || p < mi && a[p + ao] <= a[q + ao]) {
                b[i + bo] = a[p++ + ao];
            } else {
                b[i + bo] = a[q++ + ao];
            }
        }
        run[++last] = hi;
    }
    if ((count & 1) != 0) {
        for (int i = right, lo = run[count - 1]; --i >= lo;
            b[i + bo] = a[i + ao] 
        );
        run[++last] = right;
    }
    int[] t = a; a = b; b = t;
    int o = ao; ao = bo; bo = o;
}
```

先解释一下参数：k是当前RUN的上界指针，last是当前的RUN的修改指针（用来每次修改RUN的新界标，并且计算RUN的总长度，所以会有一个外层的 count = last）， hi 是每次需要合并的两个RUN之中的最高上界，mi 是每次需要合并的两个RUN之间的界限，（所以后面有 p < mi 这个换边条件）， i 是当前的合并的数量指针（所以它每次都和bo相加），p 是下半部分的指针，q是上半部分指针。

剩下就很好懂了，其实它就是一个不断的把整个RUN里面的每个子序列做两两归并的一个过程（如果有多余的，那么用最后一个判断它是否存在，直接给他复制进去）

整个流程并不是一个非常严格实现TimSort的排序算法（因为Timsort要求每次尽量都用最小的两个序列去归并）而不是无脑的归并临近的两个，不过就整个流程上看，还是参考了TimSort比较多，比标准的TimSort少了一个补足RUN长度的过程，没有严格的去遵守压栈和归并的原则，只是从概率上做到了“差不多”——嘛，毕竟还有一个专门的文件来实现TimSort，这里只是没有采用而已

## 小结
Java对于int数组的排序采用了非常多的方式：

1.对于长度小于47的，采用插入排序
1.对于长度长度介于48~286的，采用双标快排（特殊情况会用单标快排），递归的时候会考虑情况1
1.长度超过286的，使用类似TimSort的排序，但是如果RUN数量太多，会退化为双标快排

不知不觉写了好长，感觉不拆开都不行了，接下来主要看看针对其他类型的不同处理方式

