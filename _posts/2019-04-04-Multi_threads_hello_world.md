---
layout:     post
title:      "5个线程先打印Hello再打印world"
date:       2019-04-04
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Java
---

#### 问题

5个线程打印 __Hello__ 和 __world__：要求5个线程先连续打印全部 __Hello__，再连续打印全部 __world__。

#### 实现

题目中指明5个线程合作，那么可以先让5个线程打印 __Hello__。线程打印完毕后就在 __CyclicBarrier__ 实例上等待，直到 __CyclicBarrier__ 累计线程数到达指定值，所有线程都会同时放行。放行后的线程继续打印 __world__ 即可完成要求。

```java
public class CyclicBarrierAnswer {
    private static final int THREAD_COUNT = 5; // CyclicBarrier计数和线程池数量
    public static void main(String[] args) {
        CyclicBarrier barrier = new CyclicBarrier(THREAD_COUNT, null);
        ExecutorService service = Executors.newFixedThreadPool(THREAD_COUNT);

        for (int i = 0; i < THREAD_COUNT; i++) {
            service.execute(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + ": Hello");
                    // 线程打印内容后都在这里等待
                    barrier.await();
                    // 累计达目标线程数放行所有线程
                    System.out.println(Thread.currentThread().getName() + ": world");
                } catch (InterruptedException | BrokenBarrierException ignored) {
                }
            });
        }
    }
}
```

#### 运行结果

```
pool-1-thread-2: Hello
pool-1-thread-3: Hello
pool-1-thread-4: Hello
pool-1-thread-5: Hello
pool-1-thread-1: Hello

pool-1-thread-1: world
pool-1-thread-4: world
pool-1-thread-5: world
pool-1-thread-3: world
pool-1-thread-2: world
```
