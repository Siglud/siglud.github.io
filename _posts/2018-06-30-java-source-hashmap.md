---
layout: post
title:  "Java 源代码阅读之 HashMap的底层数据结构"
date:   2018-06-30 20:10:00 +0800
author: Siglud
categories:
  - Java
tags:
  - Java
comment: true
share: true
---

## 引言
HashMap 也是常用的数据结构类了，了解它的性能、优势和劣势基本上是必须的常识。

## Java Doc
首先看看Doc写了些啥。HashMap主要是实现了Map这个接口，然后它可以容忍null作为key或者value，他和HashTable的区别在于是否容忍null值和同步方面，这个类不保证任何顺序。在做get和put的操作的时候，可以认为时间复杂度是O(1)。如果在其中进行迭代操作，时间复杂度依赖于buckets的数量*总元素的数量。（也就是迭代的时候其实是比较慢的），所以如果迭代的性能对你很重要，切不可把初始的容量设置得太高，然后LOAD_FACTOR这个因素也会影响到它。然后有两个最基本的参数可以影响到一个HashMap，一个是initial capacity，另一个是load factor，前者是初始的Buckets的数量，后者是决定什么时候buckets翻倍和重新hash的因子，如果当前使用量/buckets的值 > load factor，那么当前HashMap中所有的值都需要被重新Hash，然后容量增加一倍。

默认的load factor是0.75，一般情况下，它工作得很好，如果太高，它会造成过多的重复key而降低get与put的性能，而太小则会浪费更多的空间，在你设置init capacity的时候应该充分考虑它的存在，否则可能会造成rehash而损失性能

如果hashmap中有存在非常大数量的键值对，那么一开始就创建一个足够大的容量是非常有必要的，它会更高效也更加节约空间（避免反复rehash和过大的扩张空间）但是于要注意，如果作为key的部分有太多重复的hashCode()结果会非常直接的降低整体性能，但是如果作为key的部分能够支持Comparable这个接口，那么它可能会以排序的方式去降低性能损耗。（也就是Key部分实现Comparable这个接口其实可以潜在的提高HashMap的性能）

然后作者重申了一下这个实现是没有同步操作的，具体表现在如果有线程“修改”了这个Map的结构的时候，一定要同步操作，而所谓的“修改”是指增加或者删除新的Key，而真正的修改一个已有Key的Value反倒不算是“修改”，因为它其实对整个结构而言反倒没啥影响。然后它强烈建议你如果需要使用同步操作的时候对包裹使用这个map的对象进行加锁

然后和ArrayList其实相同，如果在迭代的过程中，Map本身的结构发生了变化，它会马上抛出ConcurrentModificationException错误，阻止你延伸错误。但是也和ArrayList一样，它不能保证这个一定有效，它只是起到一个拦水坝的作用，并不能保证100%没问题。

另外有一段添加的Doc在serialVersionUID后面，应该是1.8之后加入的，简述了Tree Node对整个HashMap实现的影响，包括了性能、使用范围之类，因为没有具体提到太多，就不在这里多说了——反正后面看代码自然就明白是怎么做的了

很多注释其实都是从HashTable里面直接抄过来的，甚至连名字都忘了改-_-|||

## 类常量（初始化设置量）
默认的容量是 1 << 4 也就是16

默认的最大容量是 1 << 30

默认的Load factory是0.75

由普通的List Node变成 Tree Node的阈值为8，名字为  TREEIFY_SHRESHOLD

同理，由Tree Node的模式变回List Node 的阈值为6，不设置为和上面同样的一个值是为了防止反复摇摆

MIN_TREEIFY_CAPACITY表示整个容量大于多少的时候才允许Node变味Tree Node，否则只会扩容

## 构造器
默认有4个构造器，分别是：空的；指定初始容量的；指定初始容量和loadFactor的；传入了Map类型初始值的

只要不指定loadFactor，那么这个值就使用默认值

## 静态内部类
有两个非常重要的静态内部类，也是存放所有的数值节点的数据类型，一个是Node，一个是TreeNode

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```
一般Node的代码很好理解，其实对我们使用者而言，只要注意它其实存放了几个值就好了
1.Value的Hash它是直接缓存了的，并不需要每次都Hash，如果有必要ReHash，它也不会去再Hash一遍原有的值
1.Key也被存放到了Node中，减少了从Value反向找到Key的时间
1.冲突的Value互相之间是以链表的方式连接的

TreeNode的代码就要复杂的多（光行数就有近600行）
```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V>
```
首先可以看到TreeNode是继承LinkedHashMap.Entry的，内部整体上它是实现了红黑树的数据结构

```java
/**
    * Returns root of tree containing this node.
    */
final TreeNode<K,V> root() {
    for (TreeNode<K,V> r = this, p;;) {
        if ((p = r.parent) == null)
            return r;
        r = p;
    }
}
```
寻找root节点的函数，对当前节点无限上溯直到parent为null，则为根节点

```java
/**
* Ensures that the given root is the first node of its bin.
*/
static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
    int n;
    if (root != null && tab != null && (n = tab.length) > 0) {
        int index = (n - 1) & root.hash;
        TreeNode<K,V> first = (TreeNode<K,V>)tab[index];
        if (root != first) {
            Node<K,V> rn;
            tab[index] = root;
            TreeNode<K,V> rp = root.prev;
            if ((rn = root.next) != null)
                ((TreeNode<K,V>)rn).prev = rp;
            if (rp != null)
                rp.next = rn;
            if (first != null)
                first.prev = root;
            root.next = first;
            root.prev = null;
        }
        assert checkInvariants(root);
    }
}
```
这也是一个辅助的函数，先不看别的，看看这是什么鬼？

```java
int index = (n - 1) & root.hash;
```
为什么要把长度与自己的hash按位求与？其实这只是一个trick，为了定位到Root，于是这又是个什么概念呢？

首先要从确定Key的Hash说起

```java
/**
 * Computes key.hashCode() and spreads (XORs) higher bits of hash
 * to lower.  Because the table uses power-of-two masking, sets of
 * hashes that vary only in bits above the current mask will
 * always collide. (Among known examples are sets of Float keys
 * holding consecutive whole numbers in small tables.)  So we
 * apply a transform that spreads the impact of higher bits
 * downward. There is a tradeoff between speed, utility, and
 * quality of bit-spreading. Because many common sets of hashes
 * are already reasonably distributed (so don't benefit from
 * spreading), and because we use trees to handle large sets of
 * collisions in bins, we just XOR some shifted bits in the
 * cheapest possible way to reduce systematic lossage, as well as
 * to incorporate impact of the highest bits that would otherwise
 * never be used in index calculations because of table bounds.
 */
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
又是一个典型的注释比代码长的……

这段其实也挺简单的，hash算法就是key当前的hashcode与自己的hash code无符号右移16位的结果再做XOR，那么明显的，因为不带符号的原因，高位的16位全是0，那么高位的1会得以保留，而低位就相当于是与自己的高位在做XOR

然后它又是怎么放入那个存放各个Hash的Table的呢？再看一个代码片段

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
```
这段代码没摘录，只是写了一个最朴素的情况——就是当table刚好空出来没有值（没有出现冲突的情况），但是已经很明显了

答案就是计算出了Hash值之后，(tableLength - 1) & hash就是它确定存放对应值的方式。

继续回到之前的代码…… rn、rp分别代表了rootNext和rootPrev，这个动作就是把root插到原本的所在地的节点（first）的前面，然后把root的next和prev互相连接起来。

```java
final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
    TreeNode<K,V> p = this;
    do {
        int ph, dir; K pk;
        TreeNode<K,V> pl = p.left, pr = p.right, q;
        if ((ph = p.hash) > h)
            p = pl;
        else if (ph < h)
            p = pr;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        else if (pl == null)
            p = pr;
        else if (pr == null)
            p = pl;
        else if ((kc != null ||
                    (kc = comparableClassFor(k)) != null) &&
                    (dir = compareComparables(kc, k, pk)) != 0)
            p = (dir < 0) ? pl : pr;
        else if ((q = pr.find(h, k, kc)) != null)
            return q;
        else
            p = pl;
    } while (p != null);
    return null;
}
```
比较核心的find，比较了非常多的异常情况，不过理解起来不难
1. 当前hash大于给定hash，往左子树上找
1. 当前hash小于给定hash，往右子树上找
1. 当hash相等，并且当前的key等于给定的key，则认为当前节点即为所求）
1. 当hash相等，并且左子树没值了，往右子树上找
1. 当hash相等，并且右子树没值了，往左子树上找
1. 当提供了key的比较函数，并且调用这个比较函数不会得到一个null的时候，通过这个比较函数来比较给定的key与当前的key作比较，通过它的值是否小于0来判断向左还是向右查找
1. 如果也没有提供比较函数，接下来就是无脑递归了，去右节点上找个遍，如果找到了，就给它返回，否则就去左节点上找个遍
1. 如果任何时候当前节点值为null，则声明为找不到这个节点，返回值为null

这里告诉了我们什么？不要乱用奇怪的key去做Map的Key值，最好给定能够简单的比较大小的Key值，否则在冲突较多的情况下，可能会退化成一个O(n)的遍历查找

```java
/**
    * Forms tree of the nodes linked from this node.
    */
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        if (root == null) {
            x.parent = null;
            x.red = false;
            root = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;
                K pk = p.key;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                            (kc = comparableClassFor(k)) == null) ||
                            (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);

                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    moveRootToFront(tab, root);
}
```

这就是把Array转为红黑树的代码了，其实挺简单的，首先判断一下Root是否存在，如果不存在，那么当前节点就是Root了，否则呢，把它的Hash值与Root的Hash值相比较，即自顶向下的插入，然后通过 balanceInsertion来让整个树恢复平衡

```java
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                            TreeNode<K,V> x) {
    x.red = true;
    for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
        if ((xp = x.parent) == null) {
            x.red = false;
            return x;
        }
        else if (!xp.red || (xpp = xp.parent) == null)
            return root;
        if (xp == (xppl = xpp.left)) {
            if ((xppr = xpp.right) != null && xppr.red) {
                xppr.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {
                if (x == xp.right) {
                    root = rotateLeft(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateRight(root, xpp);
                    }
                }
            }
        }
        else {
            if (xppl != null && xppl.red) {
                xppl.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {
                if (x == xp.left) {
                    root = rotateRight(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateLeft(root, xpp);
                    }
                }
            }
        }
    }
}
```
让树恢复平衡的算法。参数xp=当前节点的父节点，xpp=祖父节点，xppl和xppr分别为左右叔节点

代码的特征是很多复制都跟判断放在了一起，所以导致代码有点晦涩，但是搞清楚上面的参数释义之后整体上看起来还是很清晰的。

再看看左旋和右旋的代码
```java
 static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                    TreeNode<K,V> p) {
    TreeNode<K,V> r, pp, rl;
    if (p != null && (r = p.right) != null) {
        if ((rl = p.right = r.left) != null)
            rl.parent = p;
        if ((pp = r.parent = p.parent) == null)
            (root = r).red = false;
        else if (pp.left == p)
            pp.left = r;
        else
            pp.right = r;
        r.left = p;
        p.parent = r;
    }
    return root;
}
```
简直是精炼到极致的代码——特别的喜欢用连续的赋值以及把赋值和判断语句串联在一起（虽然我并不推崇这种做法）

首先看看变量的意义，r=当前节点的右子节点，rl=r的左子节点，pp=当前节点的父节点

旋转的时候它是不会反复迭代的，只做一个动作，如果当前节点为null了或者r不存在了，就不旋转了（废话，左旋转的核心就是要让r替代p的位置，r不存在了自然无从旋转了）

如果rl节点存在（这个地方，它会修改p的右孩子的指向，指向r的左孩子），修改rl的父节点指向p；如果pp节点不存在（这个地方，它会顺带修改r的父节点的指向，指向pp，即准备让r接替p的位置），则说明r要变成root啦，root节点自然是不允许是红色的，染色成为黑色；如果pp节点存在，且p是pp的左侧（即当前在整个树的左边），把pp的左节点指向r，否则就把右节点指向r——这一段实际就是让r替代掉p，取决于当前p的装填去修改父节点的状态。

最后把r节点（已经取代了p节点的位置）的右儿子指向p，p的父指向p（修改它俩之间的关系）

基本算法还有一个平衡删除，不过代码太长了就不贴了，而且代码风格相对一致，剩下就看看一些比较的，红黑树中平时见不到的代码

来看看这个split的代码

```java
/**
 * Splits nodes in a tree bin into lower and upper tree bins,
 * or untreeifies if now too small. Called only from resize;
 * see above discussion about split bits and indices.
 *
 * @param map the map
 * @param tab the table for recording bin heads
 * @param index the index of the table being split
 * @param bit the bit of hash to split on
 */
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
    TreeNode<K,V> b = this;
    // Relink into lo and hi lists, preserving order
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        next = (TreeNode<K,V>)e.next;
        e.next = null;
        if ((e.hash & bit) == 0) {
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;
        }
        else {
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
    }

    if (loHead != null) {
        if (lc <= UNTREEIFY_THRESHOLD)
            tab[index] = loHead.untreeify(map);
        else {
            tab[index] = loHead;
            if (hiHead != null) // (else is already treeified)
                loHead.treeify(tab);
        }
    }
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}
```
这段代码是干嘛的呢？如果一个Node变成了一棵红黑树，而这个时候HashMap整体遭遇到了扩容，这个时候就有一个重新确定每个节点应该放到什么地方的一个过程，同时根据UNTREEIFY_THRESHOLD的设置，因为扩容了之后可能不再需要Tree了，仅仅只是需要一个普通的链表就足够，有一个反向变成链表的过程。

发现写得真的是太长了，还是分成两部分来写吧，下次写HashMap的函数。