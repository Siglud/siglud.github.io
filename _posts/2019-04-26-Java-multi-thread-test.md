---
layout: post
title:  "在Java中测试一些并发条件"
date:   2019-04-26 17:37:00 +0800
author: Siglud
categories:
  - Java
  - Kotlin
tags:
  - Java
comment: true
share: true
---
在业务中因为使用了一些锁，需要测试一下锁的有效性，保证不会出纰漏，因此会专门对这个Service做一些并发测试，做起来也挺简单的，但是还是记录一下：

总的来说，是使用CyclicBarrier来人为的制造临界的竞态条件，相当于“各就各位，预备，抢”这个流程了

```kotlin
@Test
@DisplayName("多线程重载测试")
fun testMultiThreadReload() {
    whenever(aasKeywordDao.listKeyword(null, null)).thenReturn(
            keyWordList + listOf(
                    AasKeywordDo(6, "测试", now, ""),
                    AasKeywordDo(7, "keyword", now, "")
            ))
    val pool = Executors.newCachedThreadPool()
    val barrier = CyclicBarrier(4)
    val tester = (0 .. 3).map { threadNum ->
        pool.submit( Callable {
            logger.info("test start in $threadNum")
            barrier.await()
            wordTrie.rebuildFromDb()
        })
    }.count {
        it.get().get() == true
    }
    Assertions.assertEquals(1, tester)
    Assertions.assertEquals(null, wordTrie.checkExists("a"))
}
```
其实写得也很糙，如果测试环境线程切换压力很大的话这个时候虽然是同时发出也还是有可能有线程被调度到后面去执行，如果轮到它执行的等待时间甚至长于了我给定的两个值的建Trie的过程，测试还是会不通过（我的代码是遇到拿不到锁冲突的情况直接不做的），当然，测试不通过不代表代码不对，只要不是正确的拿到了锁避免了两个线程同时开始建Trie就足够了。

那个很奇怪的it.get().get()是因为返回的值是一个AsyncResult。

