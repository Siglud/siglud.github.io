---
layout: post
title:  "Java Reactor编程初步"
date:   2018-05-16 20:49:00 +0800
author: Siglud
categories:
  - Kotlin
tags:
  - Kotlin
  - Reactor
comment: true
share: true
---

## Nothing Happens Until You subscribe()
大体上确实是如此，如果不订阅，任何流都不会主动的启动

```kotlin
fun main(args: Array<String>) {
    val baseFolder = Paths.get("Z:\\Download\\source")
    val streamSource = baseFolder.toFile().walk().asSequence()
        .filter{ 
            println("do filter")
            !it.isDirectory 
        }
        .toFlux()
}
```

是没有输出的。

但是这句话也并非绝对

```kotlin
fun main(args: Array<String>) {
    val baseFolder = Paths.get("Z:\\Download\\source")
    val streamSource = baseFolder.toFile().walk().asSequence()
        .filter{ 
            println("do filter"); 
            !it.isDirectory 
        }
        .toFlux()
        .blockLast()
}
```

这样就有输出了

因为blockLast()是几个转换异步为同步的方法之一



## I'm using Flux. And doing operations in parallel. But what I have observed is I am losing some events while doing parallel.
是的，最初我也遇到这个问题了，这个代码运行时是没有问题的

```kotlin
fun main(args: Array<String>) {
    val baseFolder = Paths.get("Z:\\Download\\source")
    val streamSource = baseFolder.toFile().walk().asSequence().filter{ !it.isDirectory }.toFlux()
    streamSource
        .zipWith(Flux.range(0, Int.MAX_VALUE))
        .map { println("No.${it.t2} : ${it.t1}") }
        .subscribe()
}
```

但是改成这样问题就来了

```kotlin
fun main(args: Array<String>) {
    val baseFolder = Paths.get("Z:\\Download\\source")
    val streamSource = baseFolder.toFile().walk().asSequence().filter{ !it.isDirectory }.toFlux()
    streamSource
            .zipWith(Flux.range(0, Int.MAX_VALUE))
            .parallel()
            .runOn(Schedulers.parallel())
            .map { inputTuple ->
                Files.newReader(inputTuple.t1, Charsets.UTF_8).use {
                    BufferedReader(it).use {
                        val firstLine = it.readLine()
                        println("No.${inputTuple.t2} : ${inputTuple.t1} + $firstLine")
                        firstLine
                    }
                }
            }
            .subscribe()
}
```

这个时候，这个代码就会表现得相当奇怪，有时候会输出一两条，有时候可能一条也不输出，有时候又可能输出更多。

原因就在于runOn的这个Schedulers.parallel()，这个是一个内部封装的线程池，它创建线程池的方法是：

```java
static final Supplier<Scheduler> PARALLEL_SUPPLIER = () -> newParallel(PARALLEL,
    Runtime.getRuntime()
            .availableProcessors(),
    true);
```

可以看到它把daemon设置为true了，这就意味着，当主线程退出只剩它一个工作线程的时候它就会让整个进程退出，而不会卡住进程。于是，当主线程完成生产任务之后决定退出，后继在工作的线程**哪怕有工作没做完，它也不做直接就退出掉了**。

当然，自己创建一个线程池，把daemon设置为false也不合理，这样工作线程会阻碍主线程正常退出，这明显也是不正确的。所以正确的做法应该是将异步的执行在结尾的时候转为同步

```kotlin
fun main(args: Array<String>) {
    val baseFolder = Paths.get("Z:\\Download\\source")
    val streamSource = baseFolder.toFile().walk().asSequence().filter{ !it.isDirectory }.toFlux()
    streamSource
            .zipWith(Flux.range(0, Int.MAX_VALUE))
            .parallel()
            .runOn(Schedulers.parallel())
            .map { inputTuple ->
                Files.newReader(inputTuple.t1, Charsets.UTF_8).use {
                    BufferedReader(it).use {
                        val firstLine = it.readLine()
                        println("No.${inputTuple.t2} : ${inputTuple.t1} + $firstLine")
                        firstLine
                    }
                }
            }.toFlux().blockLast()
}
```

这样就可以了。当然，实际应用中，像这样脚本式的跑一跑就结束了的情况很少，一般主线程是不会退出的，所以放心的使用上面的方法就可以了



## Don't forget add complete function when you use Flux#create
这也是我之前犯下的错误之一，

```kotlin
Flux.create<Int>{ sink ->
        (0 .. 100).forEach {
            sink.next(it)
        }
        // sink.complete()
    }
    .map { it + 1 }
    .toFlux().blockLast()
```

如果不加complete，那么当你blockLast()的时候反倒无法正常退出主进程了



## ThreadLocal is useless in reactor
这也算是我的理解错误之一，确实reactor推荐使用Context来装载一些伴随变量，但是这并不意味着无论什么时候都必须要用Contex

其实Contex只是用来解决在整个RX链式处理的每个环节中，负责处理的Thread一直变化的问题而构建的一个方法，如果只是在一个单的map或者flatMap里面去做这个工作，配合parallel其实是没有问题的

```kotlin
fun main(args: Array<String>) {
    val baseFolder = Paths.get("Z:\\Download\\source")
    val random = Random()
    val threadLocalTest = ThreadLocal.withInitial { random.nextInt() }

    val streamSource = baseFolder.toFile().walk().asSequence().filter { !it.isDirectory }.toFlux()
    streamSource
            .zipWith(Flux.range(0, Int.MAX_VALUE))
            .parallel()
            .runOn(Schedulers.parallel())
            .doOnNext {
                println("No.${it.t2} : ${it.t1} + ${threadLocalTest.get()} @ ${Thread.currentThread().name}")
            }
            .toFlux()
            .blockLast()
}
```
观察就会发现，其实对应的thread和随机出来的值还是非常的稳定的

但是，再多加几行，就又不稳定了

```kotlin
fun main(args: Array<String>) {
    val baseFolder = Paths.get("Z:\\Download\\source")
    val random = Random()
    val threadLocalTest = ThreadLocal.withInitial { random.nextInt() }

    val streamSource = baseFolder.toFile().walk().asSequence().filter { !it.isDirectory }.toFlux()
    streamSource
            .zipWith(Flux.range(0, Int.MAX_VALUE))
            .parallel()
            .runOn(Schedulers.parallel())
            .map {
                val randomNumber = threadLocalTest.get()
                println("No.${it.t2} : ${it.t1} + $randomNumber @ ${Thread.currentThread().name}")
                Pair(it.t2, randomNumber)
            }
            .toFlux()
            .map {
                println("${it.second == threadLocalTest.get()}")
            }
            .blockLast()
}
```
因为在整个流水线上每次处理同一个任务的Thread并不能保证是同一个

但是也会注意到，如果去掉中间的那个.toFlux()，那么输出比较的结果大部分都会是ture，这是因为reactor做了优化，对于连续的map调用，它会把类似的调用叠加在一起，以减少反复迭代的次数

如同对着一个list连续forEach两遍来处理，还不如在一个对list的迭代中就完成它来的高效，reactor就是这样把连续的map合成了一个去操作，当然了，这样的优化也并不会是万能的

为了加快整体的速度，在写代码的时候还是应当尽量少些中间操作，把尽可能多的操作合并起来写到一起