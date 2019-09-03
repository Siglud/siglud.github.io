---
layout: post
title:  "一道LeetCode题引发的思考"
date:   2019-05-16 19:54:00 +0800
author: Siglud
categories:
  - Java
tags:
  - Java
comment: true
share: true
---

最近偶尔小刷一把LeetCode，一道毫无难度的题无意中引发了我的注意。

题目本身不难，奇怪的是我写出来的结果：

```
Runtime: 3ms, faster than 100.00% of Java online submissions for Roman to Integer.
Memory Usage: 35MB, less than 100.00% of Java online submissions for Roman to Integer.
```

我不否认我刷Easy模式的LeetCode的目的就是为了刷100%，某些时候为了获得高速度甚至会使用一些比较猥琐的写法，但是这个100%来的有点反常规，不由让我有点好奇。

我的实现是这样的
```java
class Solution {
    public int romanToInt(String s) {
        int last = 0;
        int res = 0;
        for (int n = s.length() - 1; n >= 0; n--) {
            char i = s.charAt(n);
            switch (i) {
                case 73:
                    if (last > 1) {
                        --res;
                    } else {
                        ++res;
                    }
                    last = 1;
                    break;
                case 86:
                    if (last > 5) {
                        res -= 5;
                    } else {
                        res += 5;
                    }
                    last = 5;
                    break;
                case 88:
                    if (last > 10) {
                        res -= 10;
                    } else {
                        res += 10;
                    }
                    last = 10;
                    break;
                case 76:
                    if (last > 50) {
                        res -= 50;
                    } else {
                        res += 50;
                    }
                    last = 50;
                    break;
                case 67:
                    if (last > 100) {
                        res -= 100;
                    } else {
                        res += 100;
                    }
                    last = 100;
                    break;
                case 68:
                    if (last > 500) {
                        res -= 500;
                    } else {
                        res += 500;
                    }
                    last = 500;
                    break;
                case 77:
                    res += 1000;
                    last = 1000;
                    break;
                default:
                    break;
            }
        }
        return res;
    }
}
```
于此同时，我相信大部分的解决方案是这样的：
```java
class Solution {
    public int romanToInt(String s) {
        Map<Character, Integer> map = Map.of('I', 1, 'V', 5, 'X', 10, 'L', 50, 'C', 100, 'D', 500, 'M', 1000);

        int prev = 0;
        int sum = 0;
        for (int i = s.length() -1; i >= 0; i--) {
            int curr = map.get(s.charAt(i));
            if (prev > curr) {
                sum -= curr;
            } else {
                sum += curr;
            }
            prev = curr;
        }

        return sum;
    }
}
```

我写的那个比较快，大概是因为以下两个因素：
1. Switch的速度超过了Map，或者说至少不能太慢过Map的get，这样至少可以减少一个建立Map的时间
1. 我在Integer上的避免拆装箱带来了一定优势

那么问题来了，为什么switch的速度能超过Map？直觉上来说，Map的get复杂度是O(1)的，而switch则是一个接一个的比较，复杂度是O(n)才对。

问题的答案需要去字节码里面寻找了，首先把它反编译掉

```
javap -c Res
```

结果有点长，就截取switch的那一段来看看
```
27: tableswitch   { // 67 to 88
            67: 208
            68: 229
            69: 271
            70: 271
            71: 271
            72: 271
            73: 128
            74: 271
            75: 271
            76: 187
            77: 258
            78: 271
            79: 271
            80: 271
            81: 271
            82: 271
            83: 271
            84: 271
            85: 271
            86: 147
            87: 271
            88: 166
        default: 271
    }
128: iload_2
129: iconst_1
130: if_icmple     139
133: iinc          3, -1
136: goto          142
139: iinc          3, 1
142: iconst_1
143: istore_2
144: goto          271
147: iload_2
```
虽然我不懂字节码的语法，但是我也大概能猜出来它在干什么

注意到那个tableswitch，和我真正写的不一样，把它中间的“空隙”给补全了，结合一些资料可以知道，这个tableswitch的时间复杂度可是真正的O(1)的，完全不逊于Map的效率，大概猜测它的数据结构可能是一个数组（这里没去看源码，仅限猜测）。

而与此同时，如果switch的项目之间选择差距非常大，那么Java会把它编译成一个lookupswitch，而就算是编译为lookupswitch，它在各个项目之间也是使用的二分查找来降低时间复杂度O(lgn)，也就是说它会把所有的选项先排序，然后再去查找，而并不是傻傻的直接去一个个找的。

结论：Java的编译器果然都是优化到极致的东西。