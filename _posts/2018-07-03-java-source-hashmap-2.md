---
layout: post
title:  "Java 源代码阅读之 HashMap的基本操作"
date:   2018-07-03 21:54:00 +0800
author: Siglud
categories:
  - Java
tags:
  - Java
comment: true
share: true
---

## 调整大小
```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```
Resize是一个无参数的操作，是根据当前内部的Array的长度、当前的最大长度（threshold）来确定下一轮的threshold以及把之前的数据迁移过去的一个过程。

只要不超过最大和最小值限制，一般情况下，Map会扩大一倍；之后在调整后的Tab中调整里面的元素，根据内容的数据结构不同，调用了不同的方式。

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
数据插入的代码，这代码也是写得鬼畜无比，比如这个连续的赋值+计算
```java
n = (tab = resize()).length;
```
流程很简单的，如果HashMap为空，则先给他扩容，再往下走

如果table对应的位置为空，直接放入，结束

首先考察当前元素与即将插入的元素相等的情况，这种情况下不作操作

如果是TreeNode，调用TreeNode的putTreeVal操作，最终也会调用Tree的插入然后平衡，如果碰到了同样的值已经存在，那么它会以返回值的形式赋值给e，如果像这样最终没有实际插入，是不会做可能的resize调整的，resize一定是在插入完成之后再操作，这也是为什么Threshold一定要小于1的原因。

如果是链表，则考虑直接插入、遇到了相同值或者当长度达到了需要Treeify的时候变红黑树的三种状态

最后做一些收尾工作，调整当前的总大小（this.size）、调整当前的数据总量统计（modCount）、需要的时候做resize、以及给LinkedHashMap做了一个接口，允许做最后的一些处理。

```java
public void forEach(BiConsumer<? super K, ? super V> action) {
    Node<K,V>[] tab;
    if (action == null)
        throw new NullPointerException();
    if (size > 0 && (tab = table) != null) {
        int mc = modCount;
        for (Node<K,V> e : tab) {
            for (; e != null; e = e.next)
                action.accept(e.key, e.value);
        }
        if (modCount != mc)
            throw new ConcurrentModificationException();
    }
}
```

遍历的代码，首先在Tab中遍历，然后通过next指针遍历所有Key Hash重复的节点，最后那个modCount的校验是为了防止有人在遍历的过程中改动了map的一个预防措施。

总结一下：

1. 如果要初始化有一个指定大小的HashMap，记得用 (Count /  loadFactory(默认值0.75)) + 1，然后求大于这个值的最大的2^n，这样才会保证Map在使用的过程中不会扩容（源代码中的方法是函数tableSizeFor，这个函数是静态函数可以直接调用）。
1. 尽量的避免重复的Hash码，会极大的降低Map的效率
1. HashMap只会扩容不会做容量缩减的，当大量的节点被Remove，Map的内存利用效率可能会极端低下。可以考虑重新建立一个