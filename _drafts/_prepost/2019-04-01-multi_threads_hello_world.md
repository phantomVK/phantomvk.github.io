---
layout:     post
title:      "5个线程先打印Hello再打印world"
date:       2019-04-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Java
---

#### 问题

5个线程内部打印 __Hello__ 和 __world__：要求方法令5个线程先连续打印全部 __Hello__，再连续打印全部 __world__。

#### 实现

使用 __CyclicBarrier__ 解决问题。所有线程遇到 __CyclicBarrier__ 的 __await()__ 时会等待 __CyclicBarrier__ 的放行通知。而可放行的具体就是到达  __CyclicBarrier__ 的线程数目。

题目中指明5个线程合作，就可以先让5个线程打印 __Hello__。

```java
public class CyclicBarrierAnswer {

    private static final int THREADS_COUNT = 5;

    public static void main(String[] args) {
        CyclicBarrier barrier = new CyclicBarrier(THREADS_COUNT, System.out::println);
        ExecutorService service = Executors.newFixedThreadPool(THREADS_COUNT);

        for (int i = 0; i < THREADS_COUNT; i++) {
            service.execute(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + ": Hello");
                    barrier.await();
                    System.out.println(Thread.currentThread().getName() + ": world");
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
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
