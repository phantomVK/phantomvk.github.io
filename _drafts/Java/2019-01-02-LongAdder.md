---
layout:     post
title:         "LongAdder"
subtitle:   ""
date:        2019-01-01
author:    "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、类签名

JDK11

```java
/**
 * One or more variables that together maintain an initially zero
 * {@code long} sum.  When updates (method {@link #add}) are contended
 * across threads, the set of variables may grow dynamically to reduce
 * contention. Method {@link #sum} (or, equivalently, {@link
 * #longValue}) returns the current total combined across the
 * variables maintaining the sum.
 *
 * <p>This class is usually preferable to {@link AtomicLong} when
 * multiple threads update a common sum that is used for purposes such
 * as collecting statistics, not for fine-grained synchronization
 * control.  Under low update contention, the two classes have similar
 * characteristics. But under high contention, expected throughput of
 * this class is significantly higher, at the expense of higher space
 * consumption.
 *
 * <p>LongAdders can be used with a {@link
 * java.util.concurrent.ConcurrentHashMap} to maintain a scalable
 * frequency map (a form of histogram or multiset). For example, to
 * add a count to a {@code ConcurrentHashMap<String,LongAdder> freqs},
 * initializing if not already present, you can use {@code
 * freqs.computeIfAbsent(key, k -> new LongAdder()).increment();}
 *
 * <p>This class extends {@link Number}, but does <em>not</em> define
 * methods such as {@code equals}, {@code hashCode} and {@code
 * compareTo} because instances are expected to be mutated, and so are
 * not useful as collection keys.
 */
public class LongAdder extends Striped64 implements Serializable
```

## 二、构造方法

默认构造方法，类完成构造后初始总值为 __0__。

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
            longAccumulate(x, null, uncontended);
    }
}
```

以下两个方法通过调用__add()__ 方法实现数值递增和递减。递增传递的参数是 __1L__，递减传递的参数是 __-1L__。

```java
public void increment() {
    add(1L);
}

public void decrement() {
    add(-1L);
}
```

返回当前的总计数值。返回的值不是原子性的快照，就是一个基本类型的 __long__。在没有并发更新的情况下调用会返回准确结果。但是在计算总和时发生的并发更新，可能不会被算到总值内。

```java
public long sum() {
    Cell[] cs = cells;
    long sum = base;
    if (cs != null) {
        for (Cell c : cs)
            if (c != null)
                sum += c.value;
    }
    return sum;
}
```

重置维护总计值的变量值到 __0__。对比创建一个新的实例，通过此方法重置总计值后复用实例，是个更好的选择，但也仅在更新的时候没有在其他线程并发添加值。因为这个方法很活泼，只能在确认没有并发更新的时候使用。 

```java
public void reset() {
    Cell[] cs = cells;
    base = 0L;
    if (cs != null) {
        for (Cell c : cs)
            if (c != null)
                c.reset();
    }
}
```

方法获取总值后重置原始值为 __0__，并把保存的值作为结果返回，相当于 __sum()__ 和 __reset()__ 的合并调用。如果调用此方法的同时，还有其他线程在进行更新操作，此方法的 __返回值__ 和 __重置前存储的值__ 不保证是一致的。因为先返回值，值在其他线程继续修改，最后才被重置为 __0__。

```java
public long sumThenReset() {
    Cell[] cs = cells;
    long sum = getAndSetBase(0L);
    if (cs != null) {
        for (Cell c : cs) {
            if (c != null)
                sum += c.getAndSet(0L);
        }
    }
    return sum;
}
```
