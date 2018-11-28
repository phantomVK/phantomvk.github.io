---
layout:     post
title:      "Java源码系列(18) -- RejectedExecutionHandlers及子类"
date:       2018-10-23
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、基类

#### RejectedExecutionHandler

所有拒绝策略都需要继承此类，用于处理被 __ThreadPoolExecutor__ 拒绝的任务

```java
public interface RejectedExecutionHandler {
    
    // ThreadPoolExecutor.execute拒绝执行任务后，此方法由ThreadPoolExecutor调用
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```

实现类在 __java.util.concurrent__ 包中，源码来自 __JDK11__

## 二、实现策略

Java预定义4种拒绝策略，定制其他策略也需实现接口 __RejectedExecutionHandler__ 的抽象方法

![RejectedExecutionHandlers](/img/java/RejectedExecutionHandlers.png)

#### 2.1 CallerRunsPolicy

当任务添加到线程池被拒绝时，只要线程池尚在运行，该任务就会在调用者所在线程上直接执行。虽然这种策略并没有丢弃任务，但是会影响调用者线程其他功能的执行。

```java
public static class CallerRunsPolicy implements RejectedExecutionHandler {
    // 默认构造方法
    public CallerRunsPolicy() { }

    // 被拒绝任务直接运行
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            r.run();
        }
    }
}
```

#### 2.2 AbortPolicy

当任务添加到线程池被拒绝时抛出 __RejectedExecutionException__ 异常。这是 __ThreadPoolExecutor__ 和 __ScheduledThreadPoolExecutor__ 的默认策略。

```java
public static class AbortPolicy implements RejectedExecutionHandler {
    // 默认构造方法
    public AbortPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
                                             " rejected from " +
                                             e.toString());
    }
}
```

#### 2.3 DiscardPolicy

当任务添加到线程池被拒绝时，该任务会被丢弃且不给任何提示。

```java
public static class DiscardPolicy implements RejectedExecutionHandler {
    // 默认构造方法
    public DiscardPolicy() { }

    // 拒绝策略收到任务r后什么都没做
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}
```

#### 2.4 DiscardOldestPolicy

当任务添加到线程池被拒绝时，线程池会抛弃最久尚未被处理的任务，并把刚刚被拒绝的任务加入到线程池中。

```java
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
    // 默认构造方法
    public DiscardOldestPolicy() { }

    // 只要线程池没有关闭，最旧的任务就会被丢弃，并提交任务r
    // 这个最旧的任务其实就是线程池下一个等待处理的任务
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            e.getQueue().poll();
            e.execute(r);
        }
    }
}
```