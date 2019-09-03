---
layout: post
title:  "Kotlin+Spring 开发坑点一览（二）"
date:   2018-08-05 19:05:00 +0800
author: Siglud
categories:
  - kotlin
tags:
  - kotlin
  - spring
  - Mockito
comment: true
share: true
---

## Validation
Validation 也是Kotlin遇到的坑之一，而这个也和 Kotlin 的写法有关，特别是 data class 的写法

一段简单的校验：

```kotlin
data class User(@NotBlank val name: String, val id: Int)
```

它生成的 Java 代码如下（删除了 copy、hashCode、toString 等方法）

```java
public final class User {
   @NotNull
   private final String name;
   private final int id;

   @NotNull
   public final String getName() {
      return this.name;
   }

   public final int getId() {
      return this.id;
   }

   public User(@NotBlank @NotNull String name, int id) {
      Intrinsics.checkParameterIsNotNull(name, "name");
      super();
      this.name = name;
      this.id = id;
   }
}
```

我们惊喜的发现， @NotBlank 跑到了构造器上。这样的注解铁定是没办法生效了。

解决也很简单

```kotlin
data class User(@get:NotBlank val name: String, val id: Int)
```
这样就会把 @NotBlank 正确的注解到 getName() 上了

## Mockito
Kotlin 和 Mockito 唯一的不兼容点只是在于它的空安全方面的问题，如果硬要扣细枝末节，那么 when 这个关键字被占用也算是其中之一了，但是只少不是妨碍使用

而这个不兼容主要体现在 Any() 上， Any() 在 Mockito 上可能会被认为是 Null， 这样就导致 Kotlin 会抱怨：

```
java.lang.IllegalStateException: ArgumentMatchers.any() must not be null
```

但是其实问题在于 Mockito 给出的提示完全是找不到北的

```
You cannot use argument matchers outside of verification or stubbing.
Examples of correct usage of argument matchers:
    when(mock.get(anyInt())).thenReturn(null);
    doThrow(new RuntimeException()).when(mock).someVoidMethod(anyObject());
    verify(mock).someMethod(contains("foo"))

This message may appear after an NullPointerException if the last matcher is returning an object 
like any() but the stubbed method signature expect a primitive argument, in this case,
use primitive alternatives.
    when(mock.get(any())); // bad use, will raise NPE
    when(mock.get(anyInt())); // correct usage use

Also, this error might show up because you use argument matchers with methods that cannot be mocked.
Following methods *cannot* be stubbed/verified: final/private/equals()/hashCode().
Mocking methods declared on non-public parent classes is not supported.
```
很明显这些提示都是错的，真正的原因还是出在Nullable上

解决的方法最简单的就是使用 [mockito-kotlin](https://github.com/nhaarman/mockito-kotlin) ， 它贴心的帮你把 when 换成了 whenever，也贴心的解决了 any() 的问题