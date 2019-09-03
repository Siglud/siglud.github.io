---
layout: post
title:  "Java 虚拟机 SIGBUS 问题"
date:   2018-07-10 22:51:00 +0800
author: Siglud
categories:
  - Java
tags:
  - Java
  - JVM
comment: true
share: true
---

## 起因

今天碰到一个奇怪的JVM Crash问题，一台运行在Docker中的JVM突然Crash掉了，然后重启之后也会不断的Crash，基本上变成了无法启动的局面

报告的错误代码如下：

```
#
# A fatal error has been detected by the Java Runtime Environment:
#
#  SIGBUS (0x7) at pc=0x00007f2fc581c190, pid=215, tid=400
#
# JRE version: Java(TM) SE Runtime Environment (10.0.1+10) (build 10.0.1+10)
# Java VM: Java HotSpot(TM) 64-Bit Server VM (10.0.1+10, mixed mode, tiered, compressed oops, g1 gc, linux-amd64)
# Problematic frame:
# v  ~StubRoutines::jlong_disjoint_arraycopy
#
# Core dump will be written. Default location: /usr/src/use/core.215
#
# If you would like to submit a bug report, please visit:
#   http://bugreport.java.com/bugreport/crash.jsp
-
```

## 调查
这段代码同样的Dockerfile部署了不下4个副本，但是只有一台机器上的Docker会出问题，而且基本上是100%的会复现这个错误。

考虑到Docker本身的环境隔离，可以认为和JVM本身的关系不大，那么只有可能是两个原因

1. 数据原因
2. 外部环境

其中数据原因可以认为是代码写的有问题导致的，而外部因素可能就千奇百怪了。于是，继续从日志中寻找答案

```
Current thread (0x00007f2ee4056800):  JavaThread "AtlasFileRead-14" daemon [_thread_in_Java, id=400, stack(0x00007f2ea1c18000,0x00007f2ea1d19000)]

Stack: [0x00007f2ea1c18000,0x00007f2ea1d19000],  sp=0x00007f2ea1d17590,  free space=1021k
Native frames: (J=compiled Java code, A=aot compiled Java code, j=interpreted, Vv=VM code, C=native code)
v  ~StubRoutines::jlong_disjoint_arraycopy
J 5068 c2 java.security.MessageDigestSpi.engineUpdate(Ljava/nio/ByteBuffer;)V java.base@10.0.1 (141 bytes) @ 0x00007f2fcd5472f1 [0x00007f2fcd547040+0x00000000000002b1]
j  java.security.MessageDigest$Delegate.engineUpdate(Ljava/nio/ByteBuffer;)V+5 java.base@10.0.1
j  java.security.MessageDigest.update(Ljava/nio/ByteBuffer;)V+14 java.base@10.0.1
j  com.google.common.hash.MessageDigestHashFunction$MessageDigestHasher.update(Ljava/nio/ByteBuffer;)V+9
j  com.google.common.hash.AbstractByteHasher.putBytes(Ljava/nio/ByteBuffer;)Lcom/google/common/hash/Hasher;+2
j  com.kanjian.atlas.utils.FileUtils$getFileMd5$1.get()Ljava/lang/String;+72
j  com.kanjian.atlas.utils.FileUtils$getFileMd5$1.get()Ljava/lang/Object;+1
j  java.util.concurrent.CompletableFuture$AsyncSupply.run()V+37 java.base@10.0.1
j  java.util.concurrent.ThreadPoolExecutor.runWorker(Ljava/util/concurrent/ThreadPoolExecutor$Worker;)V+92 java.base@10.0.1
j  java.util.concurrent.ThreadPoolExecutor$Worker.run()V+5 java.base@10.0.1
j  java.lang.Thread.run()V+11 java.base@10.0.1
v  ~StubRoutines::call_stub
V  [libjvm.so+0x8abc52]  JavaCalls::call_helper(JavaValue*, methodHandle const&, JavaCallArguments*, Thread*)+0x412
V  [libjvm.so+0x8aa170]  JavaCalls::call_virtual(JavaValue*, Handle, Klass*, Symbol*, Symbol*, Thread*)+0x1d0
V  [libjvm.so+0x95248a]  thread_entry(JavaThread*, Thread*)+0x8a
V  [libjvm.so+0xd940e8]  JavaThread::thread_main_inner()+0x108
V  [libjvm.so+0xd9426e]  JavaThread::run()+0x13e
V  [libjvm.so+0xc05342]  thread_native_entry(Thread*)+0xf2
C  [libpthread.so.0+0x7e25]  start_thread+0xc5


siginfo: si_signo: 7 (SIGBUS), si_code: 2 (BUS_ADRERR), si_addr: 0x00007f2e692bf000
```

这段是展示当前正在执行的Thread，最后的一行显示它拿到了一个系统信号，代码7，名字是SIGBUS,寻根纠源的去找对应的那一行代码，这是一段计算文件MD5值的代码

```kotlin
val thisFile = file.toFile()
val md5Maker = Hashing.md5().newHasher()
FileInputStream(thisFile).use {
    val ch = it.channel
    val byteBuffer =ch.map(FileChannel.MapMode.READ_ONLY, 0,thisFile.length())
    md5Maker.putBytes(byteBuffer)
    md5Maker.hash().toString().toUpperCase()
}
```

注意到了它使用了ch.map，这段是调用mmap函数把文件直接map到内存，本质上调用的是Linux内核的mmap命令，或许问题就在这里？

把mmap和SIGBUS放狗搜一下，很明显得出了一些答案

很明显SIGBUS是mmap过程中出现的一些异常而出现的反馈，而这个信号出现的因素可能会很多，例如内存不足无法完成mmap、写入的时候无法正常写入（外部磁盘损坏等因素）、读取文件的map的时候检查需要访问的地址偏移量超过了文件当前的大小等因素

于是去检查一下当前的内存使用量，

```
              total        used        free      shared  buff/cache   
            available
Mem:           7.6G        2.3G        276M        360M        5.1G        4.6G
Swap:          8.0G        1.0G        7.0G

```

果断的使用量基本上达到了90%以上，而且也已经吃掉了大半的虚拟内存。这估计就是mmap不成功的主要因素了。而且这其中大部分的内存被cache占有了，说明了这台机器一直在重IO的状态下运行着

## 解决

考虑到我的程序基本上虽然短时间会频繁的使用某些硬盘文件数据，但是其实从长期看，历史的文件其实没有那么必要留在Cache里面，于是可以用命令把buff清理一下

```bash
echo 1 > /proc/sys/vm/drop_caches
```

顿时感觉内存空了许多

```
              total        used        free      shared  buff/cache   
            available
Mem:           7.6G        2.3G        4.8G        347M        544M        4.7G
Swap:          8.0G        981M        7.0G
```

为了以绝后患，还是把代码修改一下，用流式的方式来读取比较稳妥，guava本身也提供了这种便捷的方式

```kotlin
Files.asByteSource(file.toFile()).hash(Hashing.md5()).toString().toUpperCase()
```

再次上线，问题不再复现，毛病解决