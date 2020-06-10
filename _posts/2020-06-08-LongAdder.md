---
layout:     post
title:      "Java源码系列(25) -- LongAdder"
date:       2020-06-08
author:    "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、类签名

利用一个或多个变量共同维护一个初始值为 __0__ 的 **long** 总和。当调用 __add()__ 时出现线程竞争，这些变量集分别动态增加，减少对同一个锁的竞争。

方法 __sum()__ 或变量 __longValue__，返回当前用于维护总和变量集的总大小。

```java
public class LongAdder extends Striped64 implements Serializable
```

在多线程下更新总和值，例如进行统计数据的收集，而不是用于细粒度的同步控制，相比 __AtomicLong__，此类是更好的选择。

在较少竞争的情况下，此类和 __AtomicLong__ 特性基本相似。当在高竞争的情况下，本类的吞吐量明显更高，同时也消耗更多内存空间。

![LongAdder_UML](/img/java/LongAdder_UML.png)

__LongAdders__ 能和 __ConcurrentHashMap__ 一起使用去维护一个频繁伸缩的 __map__。例如，添加计数值到`ConcurrentHashMap<String, LongAdder> freqs`，键不存在时进行初始化，可通过`freqs.computeIfAbsent(key, k -> new LongAdder()).increment();`实现。

此类继承自 __Number__，但没有定义如 __equals__、__hashCode__、__compareTo__ 等方法，因为实例会发生变化，所以不能用作集合的键。源码版本 __JDK11__。

## 二、构造方法

默认构造方法，类完成构造后初始总和为 __0__。

```java
public LongAdder() {
}
```

## 三、成员方法

此方法能把指定参数值 __x__ 增加到目标值上。如果该参数传递负数，则意味着从总数上减去指定值。

```java
public void add(long x) {
    Cell[] cs; long b, v; int m; Cell c;
    if ((cs = cells) != null || !casBase(b = base, b + x)) {
        boolean uncontended = true;
        if (cs == null || (m = cs.length - 1) < 0 ||
            (c = cs[getProbe() & m]) == null ||
            !(uncontended = c.cas(v = c.value, v + x)))
            // 调用父类的Striped64()
            longAccumulate(x, null, uncontended);
    }
}
```

以下两个方法通过调用 __add()__ 方法实现数值递增和递减。递增传递的参数是 __1L__，递减传递的参数是 __-1L__。

```java
public void increment() {
    add(1L);
}

public void decrement() {
    add(-1L);
}
```

返回当前的总计数值。返回的值不是原子性的快照，就是一个基本类型的 __long__。在没有并发更新的情况下调用会返回准确结果。但是在计算总和时发生的并发更新，可能不会被算到总和内。

```java
public long sum() {
    // 从父类获取cells
    Cell[] cs = cells;
    long sum = base;
    if (cs != null) {
        for (Cell c : cs)
            if (c != null)
                // 从cell获取记录值并累加到sum
                sum += c.value;
    }
    // 返回sum的值
    return sum;
}
```

重置维护总计值的变量值到 __0__。对比创建一个新的实例，通过此方法重置总计值后复用实例，是个更好的选择，但也仅在更新的时候，没有其他线程并发添加值。

因为这个方法很活跃，只能在确认没有并发更新的时候使用。 

```java
public void reset() {
    // 从父类获取cells
    Cell[] cs = cells;
    base = 0L;
    if (cs != null) {
        // 逐个重置cell保存的值
        for (Cell c : cs)
            if (c != null)
                c.reset();
    }
}
```

方法获取总和后重置原始值为 __0__，并把保存的值作为结果返回，相当于 __sum()__ 和 __reset()__ 的合并调用。

如果调用此方法的同时，还有其他线程在进行更新操作，此方法的 __返回值__ 和 __重置前存储的值__ 不保证是一致的。因为先返回值，值在其他线程继续修改，最后才被重置为 __0__。

```java
public long sumThenReset() {
    // 从父类获取cells
    Cell[] cs = cells;
    // 获取现在的总计值
    long sum = getAndSetBase(0L);
    if (cs != null) {
        for (Cell c : cs) {
            if (c != null)
                // 获取总值并重置cell
                sum += c.getAndSet(0L);
        }
    }
    // 返回sum的值
    return sum;
}
```
