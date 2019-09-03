---
layout: post
title:  "Java 源代码阅读之 ArrayList"
date:   2018-05-22 20:50:00 +0800
author: Siglud
categories:
  - Java
tags:
  - Java
comment: true
share: true
---

## JavaDoc

ArrayList的Java Doc中提到了它支持一项叫做“fail-fast”的特性，就是在迭代的过程中如果发现自己被修改，会抛出异常的特性，因为它本身并非线程安全，由此避免自己被误用，但是也提到了，不要去依赖这项特性去做判断，因为它只是作为一项防御措施而存在

Java Doc中也提到了size、isEmpty、get、set、iterator、listIterator这些操作都是常数级别的时间复杂度，而add则是*基本上算是*常数级时间复杂度，其他的操作粗略的来说都是线性复杂度，其中线性复杂度的常数参数值也要小于LinkedList

## 序列化ID
```java
    private static final long serialVersionUID = 8683452581122892189L;
 ```
 开篇就看到了这个，这个现在应该应用的应该也很少了——毕竟Json基本上已经一统天下了，现在的新系统谁还在用Serializ呢？这个是指定序列化的版本ID用的一个参数，避免出现反序列化的时候不兼容问题而设置的


## 默认的容量
```java
    /**
     * Default initial capacity.
     */
    private static final int DEFAULT_CAPACITY = 10;
```

默认的容量是10


## 当前的容量
```java
    /**
     * The size of the ArrayList (the number of elements it contains).
     *
     * @serial
     */
    private int size;
```

当前的容量是被这个值缓存下来的，它也会在add、remove等操作的时候做动态更新，正是因为它的存在，size的时间复杂度才会变成常数O(1)


## 用于存放值的Array
```java
    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
    transient Object[] elementData; // non-private to simplify nested class access
```

真正用于存放值的是这个elementData，他被设置为transient和protected（默认），为的是能够让内部的那个subList方便访问，而设置为transient的原因是后面它自己实现了writeObject这个方法，所以就不用系统去帮它保存这个类变量了


## 最大长度
```java
    /**
     * The maximum size of array to allocate (unless necessary).
     * Some VMs reserve some header words in an array.
     * Attempts to allocate larger arrays may result in
     * OutOfMemoryError: Requested array size exceeds VM limit
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

它内部限定的最大长度是最大的INT的Max-8，注释也很明白，有些VM会多在Array的Header中保留一些东西，导致如果设置为MAX的话可能会有一些问题，不过你要是强制指定为MAX_VALUE它也会无视这个值


## 动态扩容
```java
    /**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     * @throws OutOfMemoryError if minCapacity is less than zero
     */
    private Object[] grow(int minCapacity) {
        return elementData = Arrays.copyOf(elementData,
                                           newCapacity(minCapacity));
    }

    private Object[] grow() {
        return grow(size + 1);
    }

    /**
     * Returns a capacity at least as large as the given minimum capacity.
     * Returns the current capacity increased by 50% if that suffices.
     * Will not return a capacity greater than MAX_ARRAY_SIZE unless
     * the given minimum capacity is greater than MAX_ARRAY_SIZE.
     *
     * @param minCapacity the desired minimum capacity
     * @throws OutOfMemoryError if minCapacity is less than zero
     */
    private int newCapacity(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity <= 0) {
            if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
                return Math.max(DEFAULT_CAPACITY, minCapacity);
            if (minCapacity < 0) // overflow
                throw new OutOfMemoryError();
            return minCapacity;
        }
        return (newCapacity - MAX_ARRAY_SIZE <= 0)
            ? newCapacity
            : hugeCapacity(minCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE)
            ? Integer.MAX_VALUE
            : MAX_ARRAY_SIZE;
    }
```

Doc里面也写的很清楚了，除非超过最大值，否则每次扩容50%，这里也可以看到如果强制指定ArrayList的长度为Integer.MAX_VALUE，它会无视之前的内部限制条件


## Clone
```java
    /**
     * Returns a shallow copy of this {@code ArrayList} instance.  (The
     * elements themselves are not copied.)
     *
     * @return a clone of this {@code ArrayList} instance
     */
    public Object clone() {
        try {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // this shouldn't happen, since we are Cloneable
            throw new InternalError(e);
        }
    }
```

克隆命令生成的是一个新的ArrayList对象，并且拥有复制了之后的elementData，所以对新的对象的改变是不会影响到之前的对象的。但是如果elementData内部是存放的指针类型的数据，那又是另一回事了，toArray也是同样，其实是把内部的elementData给copy了一份出来，所以其实这两个操作还是挺慢的


## add
```java
    /**
     * This helper method split out from add(E) to keep method
     * bytecode size under 35 (the -XX:MaxInlineSize default value),
     * which helps when add(E) is called in a C1-compiled loop.
     */
    private void add(E e, Object[] elementData, int s) {
        if (s == elementData.length)
            elementData = grow();
        elementData[s] = e;
        size = s + 1;
    }

    /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return {@code true} (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        modCount++;
        add(e, elementData, size);
        return true;
    }
```

这里可以看出扩容发生在容量不足的那一刻（不会提前做扩容），而这个modCount变量就是用来作为fail-fast的指示器以反馈ArrayList的数据出现了变更的。同时注意到Doc里面提到了，为什么要把同样一个add(E e, Object[] elementData, int s)从add中拆出来，是为了满足Java的C1 JIT的inline优化开启判断条件，默认的值是35个bytecode以下的内容会被inline进去，所以它就把最为常用的add方法的长度缩短，尽可能的让它享受inline之后带来的速度提升。


## Remove
```java
    /**
     * Removes the first occurrence of the specified element from this list,
     * if it is present.  If the list does not contain the element, it is
     * unchanged.  More formally, removes the element with the lowest index
     * {@code i} such that
     * {@code Objects.equals(o, get(i))}
     * (if such an element exists).  Returns {@code true} if this list
     * contained the specified element (or equivalently, if this list
     * changed as a result of the call).
     *
     * @param o element to be removed from this list, if present
     * @return {@code true} if this list contained the specified element
     */
    public boolean remove(Object o) {
        final Object[] es = elementData;
        final int size = this.size;
        int i = 0;
        found: {
            if (o == null) {
                for (; i < size; i++)
                    if (es[i] == null)
                        break found;
            } else {
                for (; i < size; i++)
                    if (o.equals(es[i]))
                        break found;
            }
            return false;
        }
        fastRemove(es, i);
        return true;
    }
```

删除操作中很罕见的出现了一个Code Block，按照我惯例的写法，我应该会写成

```java
int i = 0
boolean found = false
for (; i< size; i++) {
    if (o == null && es[i] == null) {
        found = true
        break;
    } else if (o.equals(es[i])) {
        found = true
        break;
    }
}
if (!found) {
    return false
}
fastRemove(es, i)
```

我的代码有两个问题：
1.  多做了一倍无意义的判断，因为if这个语句其实是非A即B的选择，如果把它放在循环内部，那么就是判断2N次，如果放到循环外部，则只用判断两次，当然这个可以自己改进
1.  我需要多一个变量出来判断是否命中，代码不如用Code Block简洁（当然，这个也看个人，或者有人认为Code Block用的很少，本身妨碍理解，我觉得也是一种意见

除此之外，也注意到它用本地的final变量es去缓存了elementData这个类变量，用来加快速度。真是非常细心的去优化过代码


## clear
```java
    /**
     * Removes all of the elements from this list.  The list will
     * be empty after this call returns.
     */
    public void clear() {
        modCount++;
        final Object[] es = elementData;
        for (int to = size, i = size = 0; i < to; i++)
            es[i] = null;
    }
```

clear操作不会缩减整个容量的大小，只会把所有的项目都置为null而已。我觉得如果只是为了快速的释放内存而去做这个clear操作，还不如等着GC去释放好了


## subList
代码太长就不列了，subList算是一个奇葩，ArrayList中很多时候都会不吝啬的使用System.arraycopy，但是唯独它，为了它实现了一个部分List的镜像访问，所有对subList的操作都会被反馈到主的ArrayList中去，而它作为一个inner class则是基本上拷贝了主代码中的全部方法的代码（当然了，基本上所有的操作都加上了checkForComodification用来判断原List是否被改动过）


## Tiny Bitset
```java
private static long[] nBits(int n) {
        return new long[((n - 1) >> 6) + 1];
    }
    private static void setBit(long[] bits, int i) {
        bits[i >> 6] |= 1L << i;
    }
    private static boolean isClear(long[] bits, int i) {
        return (bits[i >> 6] & (1L << i)) == 0;
    }
```
后面为了实现removeIf这个函数的功能，在这里实现了一个小型的bit set，很有意思的代码，用一个long数组来存放bit set，因为long有64个bit，所以对传入的初识大小n右移6位加一得到目标数组的大小，而这个目标数组就是用来存放bit set的

设置的时候，自然是>>6获取应该存放的位置，然后把目标的1同样左移对应的位置得到数值，然后用位或得到结果。既然存放的时候用的是或，那么自然获取的时候就要用位与了。

嗯，它并没有实现设置为0的选项（因为用不上），如果我来写应该是这样把：

```java
    private static void setFalse(long[] bits, int i) {
        bits[i >> 6] &= ~(1L << i);
    }
```

## removeIf
```java
    /**
     * Removes all elements satisfying the given predicate, from index
     * i (inclusive) to index end (exclusive).
     */
    boolean removeIf(Predicate<? super E> filter, int i, final int end) {
        Objects.requireNonNull(filter);
        int expectedModCount = modCount;
        final Object[] es = elementData;
        // Optimize for initial run of survivors
        for (; i < end && !filter.test(elementAt(es, i)); i++)
            ;
        // Tolerate predicates that reentrantly access the collection for
        // read (but writers still get CME), so traverse once to find
        // elements to delete, a second pass to physically expunge.
        if (i < end) {
            final int beg = i;
            final long[] deathRow = nBits(end - beg);
            deathRow[0] = 1L;   // set bit 0
            for (i = beg + 1; i < end; i++)
                if (filter.test(elementAt(es, i)))
                    setBit(deathRow, i - beg);
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            expectedModCount++;
            modCount++;
            int w = beg;
            for (i = beg; i < end; i++)
                if (isClear(deathRow, i - beg))
                    es[w++] = es[i];
            shiftTailOverGap(es, w, end);
            return true;
        } else {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            return false;
        }
    }
```

优化得比较鬼畜的一个代码，首先它先无脑的用filter跑一遍，它仅仅只做比较，找到第一个符合条件的值的位置，然后停下来，因为只做比较，那么这段代码明显要比后面的一大长串跑的快的多，同时，它避免了没有任何意义的内存分配——它在后面会分配一个长度为剩下大小的bitSet用来存放找到的结果，但是如果一个都没找到，那么干脆连这个bitSet也省下了。


## 结语
整个看下来，其实核心的思想很容易理解，但是体现在细节上的巧思还是让人觉得很震撼，可以说给我这个基本上天天都在写CRUD的程序员很多思考，优化的思想可以体现在各个角落，甚至是变成的习惯思维上。
