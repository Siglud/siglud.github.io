---
layout: post
title:  "Java 源代码阅读之 TimSort"
date:   2018-06-24 13:30:00 +0800
author: Siglud
categories:
  - Java
tags:
  - Java
comment: true
share: true
---

## 引子
还是之前看Arrays这个类留下的相关类，之前看的DualPivotQuickSort主要是针对原始类型，而对于所有的Object，Java使用的是TimSort排序来处理，这个TimSort在Java中又分为两个略有不同的类，一个是TimSort.java另外一个是ComparableTimSort.java，这二者基本上是一致的，唯一不同的就是，如果类型实现了Comparable接口并且没有明确的去指定比较方法，就会去用ComparableTimSort，否则就去用TimSort，二者在算法上并没有本质的不同点

## doc
还是先看看JavaDoc，doc中稍微介绍了一下TimSort和它的优点，说它在随机性比较强的Arrays的处理上比传统的归并排序要更为高效，并且它是一个稳定、时间复杂度最差也为nlg(n)的排序算法，算法导论告诉我们，这基本上就是基于比较的排序算法的极限值了。

然后说这个排序算法呢，基本上是Tim Peter给Python写的List的排序算法的Java版改写，也列举了相关的C的实现的地址和相关论文的地址。最后提到，这个实现是假定你给定的序列已经充分的满足了需要去做TimSort的基础条件，小的序列就自行去做插入排序，别过来调用这个类

## 构造器
```java
private TimSort(T[] a, Comparator<? super T> c, T[] work, int workBase, int workLen) {
    this.a = a;
    this.c = c;

    // Allocate temp storage (which may be increased later if necessary)
    int len = a.length;
    int tlen = (len < 2 * INITIAL_TMP_STORAGE_LENGTH) ?
        len >>> 1 : INITIAL_TMP_STORAGE_LENGTH;
    if (work == null || workLen < tlen || workBase + tlen > work.length) {
        @SuppressWarnings({"unchecked", "UnnecessaryLocalVariable"})
        T[] newArray = (T[])java.lang.reflect.Array.newInstance
            (a.getClass().getComponentType(), tlen);
        tmp = newArray;
        tmpBase = 0;
        tmpLen = tlen;
    }
    else {
        tmp = work;
        tmpBase = workBase;
        tmpLen = workLen;
    }

    /*
        * Allocate runs-to-be-merged stack (which cannot be expanded).  The
        * stack length requirements are described in listsort.txt.  The C
        * version always uses the same stack length (85), but this was
        * measured to be too expensive when sorting "mid-sized" arrays (e.g.,
        * 100 elements) in Java.  Therefore, we use smaller (but sufficiently
        * large) stack lengths for smaller arrays.  The "magic numbers" in the
        * computation below must be changed if MIN_MERGE is decreased.  See
        * the MIN_MERGE declaration above for more information.
        * The maximum value of 49 allows for an array up to length
        * Integer.MAX_VALUE-4, if array is filled by the worst case stack size
        * increasing scenario. More explanations are given in section 4 of:
        * http://envisage-project.eu/wp-content/uploads/2015/02/sorting.pdf
        */
    int stackLen = (len <    120  ?  5 :
                    len <   1542  ? 10 :
                    len < 119151  ? 24 : 49);
    runBase = new int[stackLen];
    runLen = new int[stackLen];
}
```
构造器其实挺短的，或者说长得其实是注释（orz），这么一大串的注释也指明了关注点——如果你变动了MIN_MERGE这个用于指定最小RUN长度的数值，那么你需要同步的去改这个stackLen这个的值，特别是当你把它改小了的时候，如果stackLen不够大就会溢出

其他的都是在初始化一些临时空间，特别是tmp这个值，它会用传入的待排数组的长度的一倍和INITIAL_TMP_STORAGE_LENGTH（256）中的小值作为初始长度值。

## 核心
整个类除了sort方法都是private的，这个sort自然是核心，其实这个代码并不长，忽略其中的函数细节，其实也很容理解TimSort的思想

```java
static <T> void sort(T[] a, int lo, int hi, Comparator<? super T> c,
                        T[] work, int workBase, int workLen) {
    assert c != null && a != null && lo >= 0 && lo <= hi && hi <= a.length;

    int nRemaining  = hi - lo;
    if (nRemaining < 2)
        return;  // Arrays of size 0 and 1 are always sorted

    // If array is small, do a "mini-TimSort" with no merges
    if (nRemaining < MIN_MERGE) {
        int initRunLen = countRunAndMakeAscending(a, lo, hi, c);
        binarySort(a, lo, hi, lo + initRunLen, c);
        return;
    }

    /**
        * March over the array once, left to right, finding natural runs,
        * extending short natural runs to minRun elements, and merging runs
        * to maintain stack invariant.
        */
    TimSort<T> ts = new TimSort<>(a, c, work, workBase, workLen);
    int minRun = minRunLength(nRemaining);
    do {
        // Identify next run
        int runLen = countRunAndMakeAscending(a, lo, hi, c);

        // If run is short, extend to min(minRun, nRemaining)
        if (runLen < minRun) {
            int force = nRemaining <= minRun ? nRemaining : minRun;
            binarySort(a, lo, lo + force, lo + runLen, c);
            runLen = force;
        }

        // Push run onto pending-run stack, and maybe merge
        ts.pushRun(lo, runLen);
        ts.mergeCollapse();

        // Advance to find next run
        lo += runLen;
        nRemaining -= runLen;
    } while (nRemaining != 0);

    // Merge all remaining runs to complete sort
    assert lo == hi;
    ts.mergeForceCollapse();
    assert ts.stackSize == 1;
}
```

首先它先确定是否待排序的总长度是否比最小的RUN的长度还要小，如果只有这么点长度，那么就直接使用针对一个RUN使用的排序方式（函数后面再分析）

接下来初始化一个TimSort实例，计算它的最小RUN长度（小于这个值的都需要做binarySort补齐长度，除非总长度不够了）

把这个生成好的RUN压到RUN的Array中，对这个列表按照规则进行归并，然后如此重复直到所有的都被压入RUN Array中，最后对所有的RUN调用mergeForceCollapse做一次归并。

大致的流程总结就是
1.生成RUN
1.把RUN放入RUN的列表，然后调整这个RUN列表
1.最后把所有的RUN都归并起来

嗯，binarySort就懒得仔细看了，它其实就是一个二分查找版本的插入排序，它拥有一个基础的RUN雏形，然后对剩余的长度挨个对既有的RUN雏形进行二分查找，然后插入到找到的位置的流程，本身注释就超长，甚至注释的文本超过了代码好几倍，就不多看了。

另一个很简单的是countRunAndMakeAscending，这个类似之前DualPivot里面的连续上升、下降序列然后都调整为升序的那段代码一样，最后返回值是这个升序序列的尾部。代码本身也很简单，略过

pushRun函数就三行，也没啥好说的，就是它会同时记录下RUN的长度，避免每次都会去求Array的Size

来看看mergeCollapse这个函数

```java
private void mergeCollapse() {
    while (stackSize > 1) {
        int n = stackSize - 2;
        if (n > 0 && runLen[n-1] <= runLen[n] + runLen[n+1] ||
            n > 1 && runLen[n-2] <= runLen[n] + runLen[n-1]) {
            if (runLen[n - 1] < runLen[n + 1])
                n--;
        } else if (n < 0 || runLen[n] > runLen[n + 1]) {
            break; // Invariant is established
        }
        mergeAt(n);
    }
}
```
函数短但是挺重要的。n可以认为是当前栈的倒数第二个值，n+1就是栈尾；第一个if只是用来确定n是否需要向前推进的，先看退出条件，也就是第二个if，只有当n < 0，也就是当前栈总长度低于1了——（其实没必要，while条件里面写了 stackSize > 1），或者当前倒数第二个元素长度要大于栈尾元素长度，同时，也要不满足第一个if，也就是倒数第三个元素长度要大于最后两个元素长度之和，同时它前面的一个元素（如果存在的话），也要满足前一个元素长度大于后两个元素的长度之和。也就是最终形成一个第一个元素最长，后面的两个的和都比不上它的长度的状态同时后面这两个元素长度也是依次递减的。否则就进行mergeAt操作，mergeAt是对第N和第N+1个元素进行merge

感觉说的都要混乱了，但是它其实只想做简单的事情
1.保持栈的第一个元素是最长的
1.保持栈的第二个元素是第二长的，并且它何第三个元素加起来也没有第一个长
1.尽量合并两个短的元素

```java
private void mergeAt(int i) {
    assert stackSize >= 2;
    assert i >= 0;
    assert i == stackSize - 2 || i == stackSize - 3;

    int base1 = runBase[i];
    int len1 = runLen[i];
    int base2 = runBase[i + 1];
    int len2 = runLen[i + 1];
    assert len1 > 0 && len2 > 0;
    assert base1 + len1 == base2;

    /*
        * Record the length of the combined runs; if i is the 3rd-last
        * run now, also slide over the last run (which isn't involved
        * in this merge).  The current run (i+1) goes away in any case.
        */
    runLen[i] = len1 + len2;
    if (i == stackSize - 3) {
        runBase[i + 1] = runBase[i + 2];
        runLen[i + 1] = runLen[i + 2];
    }
    stackSize--;

    /*
        * Find where the first element of run2 goes in run1. Prior elements
        * in run1 can be ignored (because they're already in place).
        */
    int k = gallopRight(a[base2], a, base1, len1, 0, c);
    assert k >= 0;
    base1 += k;
    len1 -= k;
    if (len1 == 0)
        return;

    /*
        * Find where the last element of run1 goes in run2. Subsequent elements
        * in run2 can be ignored (because they're already in place).
        */
    len2 = gallopLeft(a[base1 + len1 - 1], a, base2, len2, len2 - 1, c);
    assert len2 >= 0;
    if (len2 == 0)
        return;

    // Merge remaining runs, using tmp array with min(len1, len2) elements
    if (len1 <= len2)
        mergeLo(base1, len1, base2, len2);
    else
        mergeHi(base1, len1, base2, len2);
}
```
基本上是很核心的呃函数了，合并两个RUN的过程

首先它调整runBase和runLen两个记录值，然后在第一个array中寻找第二个array的第一个值应该出现的位置，同样的对array2也做一遍（不过这次是在array2中找array1的最后一个值）

然后把找出来的值取出来，各自缩短对应的array的长度，因为两个array是各自已经排序好了的，那么第一个array中如果小于第二个array的中的最小值）那么自然是不用参与排序了，同理，第二个array中如果如果大于array1的部分自然也不需要参与，它们的位置本身就是对的

然后根据两个array的长度，选取不同的作为基array，各自调用mergeHi和mergeLo来合并

```java
private void mergeLo(int base1, int len1, int base2, int len2) {
    assert len1 > 0 && len2 > 0 && base1 + len1 == base2;

    // Copy first run into temp array
    T[] a = this.a; // For performance
    T[] tmp = ensureCapacity(len1);
    int cursor1 = tmpBase; // Indexes into tmp array
    int cursor2 = base2;   // Indexes int a
    int dest = base1;      // Indexes int a
    System.arraycopy(a, base1, tmp, cursor1, len1);

    // Move first element of second run and deal with degenerate cases
    a[dest++] = a[cursor2++];
    if (--len2 == 0) {
        System.arraycopy(tmp, cursor1, a, dest, len1);
        return;
    }
    if (len1 == 1) {
        System.arraycopy(a, cursor2, a, dest, len2);
        a[dest + len2] = tmp[cursor1]; // Last elt of run 1 to end of merge
        return;
    }

    Comparator<? super T> c = this.c;  // Use local variable for performance
    int minGallop = this.minGallop;    //  "    "       "     "      "
outer:
    while (true) {
        int count1 = 0; // Number of times in a row that first run won
        int count2 = 0; // Number of times in a row that second run won

        /*
            * Do the straightforward thing until (if ever) one run starts
            * winning consistently.
            */
        do {
            assert len1 > 1 && len2 > 0;
            if (c.compare(a[cursor2], tmp[cursor1]) < 0) {
                a[dest++] = a[cursor2++];
                count2++;
                count1 = 0;
                if (--len2 == 0)
                    break outer;
            } else {
                a[dest++] = tmp[cursor1++];
                count1++;
                count2 = 0;
                if (--len1 == 1)
                    break outer;
            }
        } while ((count1 | count2) < minGallop);

        /*
            * One run is winning so consistently that galloping may be a
            * huge win. So try that, and continue galloping until (if ever)
            * neither run appears to be winning consistently anymore.
            */
        do {
            assert len1 > 1 && len2 > 0;
            count1 = gallopRight(a[cursor2], tmp, cursor1, len1, 0, c);
            if (count1 != 0) {
                System.arraycopy(tmp, cursor1, a, dest, count1);
                dest += count1;
                cursor1 += count1;
                len1 -= count1;
                if (len1 <= 1) // len1 == 1 || len1 == 0
                    break outer;
            }
            a[dest++] = a[cursor2++];
            if (--len2 == 0)
                break outer;

            count2 = gallopLeft(tmp[cursor1], a, cursor2, len2, 0, c);
            if (count2 != 0) {
                System.arraycopy(a, cursor2, a, dest, count2);
                dest += count2;
                cursor2 += count2;
                len2 -= count2;
                if (len2 == 0)
                    break outer;
            }
            a[dest++] = tmp[cursor1++];
            if (--len1 == 1)
                break outer;
            minGallop--;
        } while (count1 >= MIN_GALLOP | count2 >= MIN_GALLOP);
        if (minGallop < 0)
            minGallop = 0;
        minGallop += 2;  // Penalize for leaving gallop mode
    }  // End of "outer" loop
    this.minGallop = minGallop < 1 ? 1 : minGallop;  // Write back to field

    if (len1 == 1) {
        assert len2 > 0;
        System.arraycopy(a, cursor2, a, dest, len2);
        a[dest + len2] = tmp[cursor1]; //  Last elt of run 1 to end of merge
    } else if (len1 == 0) {
        throw new IllegalArgumentException(
            "Comparison method violates its general contract!");
    } else {
        assert len2 == 0;
        assert len1 > 1;
        System.arraycopy(tmp, cursor1, a, dest, len1);
    }
}
```

虽然略长，但是其实做的事情也很简单
1.首先判断有待合并的序列长度为1的特殊情况
1.接下来是那个长长的do while循环，其实流程和上面差不多，首先合并到任一个待合并队列的长度（或者总长度相加）达到 minGallop 的长度。然后和上面一样，去做一个快速的查找首尾序列的可直接复制段的工作
1.反复重复2的工作
1.检查一下最后是否都合并完成了

几个比较有意思的代码段都在while语句的判断里面

```java
while ((count1 | count2) < minGallop)
```
两个int位与，然后做判断，可以粗略的认为是相加或者二者中的更大一方

```java
while (count1 >= MIN_GALLOP | count2 >= MIN_GALLOP);
```
两个bool的与，和逻辑与的效果相同，速度可能还要快一点

还有一个就是注释也提到了的

```java
T[] a = this.a; // For performance
```
访问函数变量比访问类变量速度快