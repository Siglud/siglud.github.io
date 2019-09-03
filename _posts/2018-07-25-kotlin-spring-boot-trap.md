---
layout: post
title:  "Kotlin+Spring 开发坑点一览（一）"
date:   2018-07-10 22:51:00 +0800
author: Siglud
categories:
  - kotlin
tags:
  - kotlin
  - spring
  - mybatis
comment: true
share: true
---

## 引子

Kotlin是个好东西，写起来快得多，代码少敲很多的同时也带来了一些回避不了的问题——那就是第三方库的兼容问题，而本身这些问题其实蛮可以不用存在的，而这些问题的焦点基本上都集中在了它的两个特性上

1. class 默认是 final 的
2. data class 好用，但是它无法给出默认的空构造器

这两个问题导致了很多第三方的包的反射机能失效，解决的方法也很简单，如果自己写，那么就是到处都open起来，因为近年来kotlin自身的发展，第三方也纷纷给与兼容和支持。

但是问题是！这些兼容和支持大多都没有写入到文档中，都是在issue中存在的。当然，也有好的，比如Spring官方就会专门为Kotlin的兼容和特性写过文章（[链接](https://spring.io/blog/2017/01/04/introducing-kotlin-support-in-spring-framework-5-0)），说明了它在这个方面是下了功夫的，于是Spring配合Kotlin基本上就是没啥问题的。

相比之下mybatis就比较坑了，它有么有做相关的工作呢？我敢肯定，它铁定是做了的，但是它文档非常的落后，很多工作默默做了，但是却憋着不说，基本上只能靠大家关注社区、重新看doc、看代码来解决。

于是把工作中遇到的坑在这里罗列一下，算是对自己踩过的坑留个印记吧

## mybatis 结果集自动映射 data class 
老生常谈的问题了，问题原因在于空构造器问题，网上给出的都是老的解决方案，即通过给所有的data class的属性增加默认值的方式，这样kotlin在编译出JVM的字节码的时候会加上一个空构造器，方便做反射使用——但是，我不想写这些莫名其妙的默认值怎么办呢？

其实也有办法，mybatis自3.4之后加入了这样一个注解：

```java
/**
 * The marker annotation that indicate a constructor for automatic mapping.
 *
 * @author Tim Chen
 * @since 3.4.3
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.CONSTRUCTOR})
public @interface AutomapConstructor {
}
```

可以指定constructor了，于是，把data class改成这样好了

```kotlin
data class User @AutomapConstructor constructor(
    val uid: Int,
    val name: String
)
```
为什么会产生这种诡异的情况呢？

这还是需要从MyBatis的代码说起

```java
  private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)
      throws SQLException {
    final Class<?> resultType = resultMap.getType();
    final MetaClass metaType = MetaClass.forClass(resultType, reflectorFactory);
    final List<ResultMapping> constructorMappings = resultMap.getConstructorResultMappings();
    if (hasTypeHandlerForResultObject(rsw, resultType)) {
      return createPrimitiveResultObject(rsw, resultMap, columnPrefix);
    } else if (!constructorMappings.isEmpty()) {
      return createParameterizedResultObject(rsw, resultType, constructorMappings, constructorArgTypes, constructorArgs, columnPrefix);
    } else if (resultType.isInterface() || metaType.hasDefaultConstructor()) {
      return objectFactory.create(resultType);
    } else if (shouldApplyAutomaticMappings(resultMap, false)) {
      return createByConstructorSignature(rsw, resultType, constructorArgTypes, constructorArgs, columnPrefix);
    }
    throw new ExecutorException("Do not know how to create an instance of " + resultType);
  }
```

MyBatis的代码都比较简单粗暴，而且是基本上没有注释的，看惯了JDK的代码，再看它无疑能感受到差距——好吧，吐槽的先不说，先看看代码

核心是那一段 if 和 else if 的集合体，这个是用来选择到底要使用什么方法来构造目标对象的核心

首先它先判断是否有明确定义的 TypeHandler，这个我一般都省略不写，如果真的写了反倒没那么多事情了。

其次判断是否有明确定义的构造器 Mapping，这个一般我也没写，毕竟追求的是自动构造，

接下来判断结果是否是一个 Interface 或者它有一个空构造器，那么进入 objectFactory.create(resultType) 这个方法，否则则进入 createByConstructorSignature 这个方法。我们的 data class 默认是没有空构造器的，所以一定会进入使用构造函数的方式。

构造函数的构造方法如下：

```java
  private Object createByConstructorSignature(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs,
                                              String columnPrefix) throws SQLException {
    final Constructor<?>[] constructors = resultType.getDeclaredConstructors();
    final Constructor<?> annotatedConstructor = findAnnotatedConstructor(constructors);
    if (annotatedConstructor != null) {
      return createUsingConstructor(rsw, resultType, constructorArgTypes, constructorArgs, columnPrefix, annotatedConstructor);
    } else {
      for (Constructor<?> constructor : constructors) {
        if (allowedConstructor(constructor, rsw.getClassNames())) {
          return createUsingConstructor(rsw, resultType, constructorArgTypes, constructorArgs, columnPrefix, constructor);
        }
      }
    }
    throw new ExecutorException("No constructor found in " + resultType.getName() + " matching " + rsw.getClassNames());
  }
```
首先判明它有构造函数被加上了 @AutomapConstructor 这个注解了么？如果有，进入 createUsingConstructor 方法，如果没有，遍历所有的构造方法。

于是我们先来看看不加任何注解的情况，其实这个函数本身并不重要，重要的是它确定这个构造器可用的那个判断，也就是 allowedConstructor ：

```java
  private boolean allowedConstructor(final Constructor<?> constructor, final List<String> classNames) {
    final Class<?>[] parameterTypes = constructor.getParameterTypes();
    if (typeNames(parameterTypes).equals(classNames)) return true;
    if (parameterTypes.length != classNames.size()) return false;
    for (int i = 0; i < parameterTypes.length; i++) {
      final Class<?> parameterType = parameterTypes[i];
      if (parameterType.isPrimitive() && !primitiveTypes.getWrapper(parameterType).getName().equals(classNames.get(i))) {
        return false;
      } else if (!parameterType.isPrimitive() && !parameterType.getName().equals(classNames.get(i))) {
        return false;
      }
    }
    return true;
  }
```

判断的流程也是非常粗暴直接
1. 拿到所有的JDBC返回的参数和构造器的参数作比较，如果完全一致，直接返回 true
1. 如果参数数量都不一样，直接返回 false ，也就是跳到下一个
1. 如果数量相同，那么一个个的作比较，如果是原始数据类型，那么去拿它的包装数据类型比一下，如果不是，则直接比类名

这里就有一个问题来了。

其实这里的数据类型比较其实是无视了 Type Mapper 的，哪怕是你定义了全局的 Type Mapper， 它其实是无视的。 这就是很多人被卡住的原因

那么为什么加了 @AutomapConstructor 的注解就可以了呢？继续看代码

```java
private Object createUsingConstructor(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix, Constructor<?> constructor) throws SQLException {
    boolean foundValues = false;
    for (int i = 0; i < constructor.getParameterTypes().length; i++) {
      Class<?> parameterType = constructor.getParameterTypes()[i];
      String columnName = rsw.getColumnNames().get(i);
      TypeHandler<?> typeHandler = rsw.getTypeHandler(parameterType, columnName);
      Object value = typeHandler.getResult(rsw.getResultSet(), prependPrefix(columnName, columnPrefix));
      constructorArgTypes.add(parameterType);
      constructorArgs.add(value);
      foundValues = value != null || foundValues;
    }
    return foundValues ? objectFactory.create(resultType, constructorArgTypes, constructorArgs) : null;
  }
```

其实加与不加只是多了一个判断而已，而其实最终都会走到这个 createUsingConstructor 方法上来， 这个时候我们看到了熟悉的 Type Handler，因为它确认你一定是有把握才会加这个注释的，所以它跨越了各种判断，直接就对着这个构造器尝试去做赋值，这样就可以拿到正确的结果

但是其实这里也有一个问题就是接下来要遇到的问题


## mybatis 绑定带默认值的 data class
如果一个 data class 所有的属性都带了默认值，那么不加注解也是没问题的，但是如果你加了注解，并且又有一部分的属性是有默认值的，这个时候又会出现问题

一般问题长这样
```
Cause: java.lang.IndexOutOfBoundsException: Index 3 out-of-bounds for length 3] with root cause

java.lang.IndexOutOfBoundsException: Index 3 out-of-bounds for length 3
```

具体是3还是多少，取决于表的colum的多少，意思就是我给了3个阐述超过了你能容纳的最大长度了

这问题原因还是在于构造器，当出现了默认值之后，它理论上会生成多个构造器，这样结合之前看到的MyBatis的代码，就知道 MyBatis 这个时候就会傻了，因为它直觉的认为不应该产生多个构造器这种情况，它对它并没有使用做 for 循环去判断真正需要使用哪个，甚至没有简单的去对参数数量去做判断。

于是几种方法来解决这个问题

1. 使用给data class 的每个字段都增加默认值的方法，让它产生一个空构造器，这个是一劳永逸的解法，适用于任何场景。
1. 使用 @JvmOverloads 注解，这个注解会对默认值产生更多的重载方法，这个其实解决得需要一点运气，通过下一节来详细解释


## mybatis 处理带默认值的函数

如果在Dao中这么写
```kotlin
fun getUserById(userId: Int=0): UserDo
```

在XML中这么写

```xml
<select id="getUserById" resultType="UserDo">
    SELECT * FROM `user` WHERE #{userId}
</select>
```

结果就会报告找不到userId这个变量了，原因很简单，这个函数的函数名被Kotlin给改掉了，它编译之后生成的代码为：

```java
UserDo getUserById(int var1);

@Nullable
public static UserDo getUserById$default(UserDao var0, int var1, int var2, Object var3) {
    if (var3 != null) {
      throw new UnsupportedOperationException("Super calls with default arguments not supported in this target, function: getUserById");
    } else {
      if ((var2 & 1) != 0) {
          var1 = 0;
      }

      return var0.getUserById(var1);
    }
}
```
变量名被改成了var1了自然就识别不出来啦，这么写就没问题了

```kotlin 
fun getUserById(@Param("userId") userId: Int=0): UserDo
```

## Kotlin + Spring编程的一般问题解决之道

很多时候都会遇到一些kotlin产生的兼容问题，它们可能很难发现，也可能埋藏得很深。简单的Debug的方式当然是看它如何生成的Java源码——毕竟我们写Java的时候其实是不会遇到这么多事情的

所以Debug的方法其实也挺简单的，IDEA自带了就有。

选择 Tools - Kotlin - Show Kotlin ByteCode
这样 IDE 的右侧会出现 Kotlin 的编译之后的字节码，当然这个我们是很难看懂的，没关系，上面有个按钮，Decompile，反编译这些字节码生成Java源代码，这下我们就看得懂了。

再回到上一个问题的遗留。

反编译一个简单的包含三个属性的Data class之后，它生成了3个构造函数每个的头顶上都是顶着 @JvmOverloads 的，说明是这个注解生成的对应的构造函数，分别是1~3个参数，在每个参数没有值的时候给出默认值（如果有提供默认值的话），然后这三个构造函数，脑袋顶上都顶着 @AutomapConstructor 这个注解，分别是：

```java
   @AutomapConstructor
   @JvmOverloads
   public BackendRoleUriDo(@Nullable Integer roleId, @NotNull String roleUri, @NotNull String roleName) {
      Intrinsics.checkParameterIsNotNull(roleUri, "roleUri");
      Intrinsics.checkParameterIsNotNull(roleName, "roleName");
      super();
      this.roleId = roleId;
      this.roleUri = roleUri;
      this.roleName = roleName;
   }

   // $FF: synthetic method
   @AutomapConstructor
   @JvmOverloads
   public BackendRoleUriDo(Integer var1, String var2, String var3, int var4, DefaultConstructorMarker var5) {
      if ((var4 & 1) != 0) {
         var1 = (Integer)null;
      }

      this(var1, var2, var3);
   }

   @AutomapConstructor
   @JvmOverloads
   public BackendRoleUriDo(@NotNull String roleUri, @NotNull String roleName) {
      this((Integer)null, roleUri, roleName, 1, (DefaultConstructorMarker)null);
   }
```

中间一个是核心的构造函数，第一个是全覆盖的构造函数，最后一个是带默认值的

如果去掉的时候呢？

```java
@AutomapConstructor
   public BackendRoleUriDo(@Nullable Integer roleId, @NotNull String roleUri, @NotNull String roleName) {
      Intrinsics.checkParameterIsNotNull(roleUri, "roleUri");
      Intrinsics.checkParameterIsNotNull(roleName, "roleName");
      super();
      this.roleId = roleId;
      this.roleUri = roleUri;
      this.roleName = roleName;
   }

   // $FF: synthetic method
   @AutomapConstructor
   public BackendRoleUriDo(Integer var1, String var2, String var3, int var4, DefaultConstructorMarker var5) {
      if ((var4 & 1) != 0) {
         var1 = (Integer)null;
      }

      this(var1, var2, var3);
   }
```

最下面那个提供了默认值的构造器没有了，继续回到 MyBatis 寻找带注释的构造器的那个代码

```java
  private Constructor<?> findAnnotatedConstructor(final Constructor<?>[] constructors) {
    for (final Constructor<?> constructor : constructors) {
      if (constructor.isAnnotationPresent(AutomapConstructor.class)) {
        return constructor;
      }
    }
    return null;
  }
```

可以看到，其实它只是在所有的装饰器中去寻找第一个带有注解的。只不过很不碰巧，这个自动生成的装饰器它总是排在第一个。但是当使用了 @JvmOverloads 这个注解之后呢？ 问题发生了一点变化，它的字节码里面默认的那个可能排到了第一个，就会莫名其妙的正确了

所以总结一下，当你需要使用 data class 来装载 MyBatis 的结果时：
1. 如果都是原始数据类型，那么不需要做任何操作，可以运行得很好
1. 如果有字段是需要 Type Mapper 来进行映射的，如果没有任何字段需要默认值，那么加上 @AutomapConstructor 就可以运行得很好了。如果有字段需要默认值，那么给所有的字段都加上默认值