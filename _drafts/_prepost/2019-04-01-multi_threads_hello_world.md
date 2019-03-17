---
layout:     post
title:      "5个线程先打印Hello在打印world"
date:       2019-04-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Java
---

#### 问题

5个线程内部打印 __Hello__ 和 __world__，要求实现方法令5个线程先打印出全部 __Hello__ 再打印全部 __word__。

#### 实现

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
