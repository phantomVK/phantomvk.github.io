---
layout:     post
title:      "Java源码系列(20) -- Runnable & Callable"
date:       2018-11-07
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Java源码系列
---

## Runnable

__Runnable__ 接口由需被线程执行的类继承实现，实现类需实现接口中无参数的方法 __run__。

此接口为那些希望在激活时执行代码的对象提供公共协议。例如 __Thread__ 实现 __Runnable__ 接口，当 __Thread__ 激活之后表示线程已经启动且尚未停止。

```java
@FunctionalInterface
public interface Runnable {
    // 当对象实现Runnable接口并用于创建线程
    // 在线程启动时，会引起run方法在独立执行的线程中执行
    public abstract void run();
}
```

其次，__Runnable__ 也表明实现类在不是 __Thread__ 的子类的情况下，也能变得活跃。实现 __Runnable__ 接口的类不需继承 __Thread__ ，可把本实例传递给 __Thread__ 实例作为运行目标。在多数情况下，如果不需要重写 __Thread__ 方法，应尽量使用 __Runnable__。

## Callable

这是返回运行结果值或抛出异常的任务。实现者需定义一个没有参数，且名为 __call__ 的方法。

```java
@FunctionalInterface
public interface Callable<V> {
    // 计算获得结果，或在无法运行时抛出异常
    // V为计算后结果的类型
    V call() throws Exception;
}
```

__Callable__ 和 __Runnable__ 接口有点类似，均设计为让类实例运行在其他线程上。但是，__Runnable__ 不会返回结果，且不能抛出受检异常。

__Executors__ 类包含一些工具方法，能把其他普通类型转换为 __Callable__。例如：把 __Runnable__ 转换为 __Callable__
