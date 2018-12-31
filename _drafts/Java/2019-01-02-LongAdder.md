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

返回当前的总计数值。返回的值不是原子性的快照，就是一个基本类型的 __long__。

```java
/**
 * Returns the current sum.  The returned value is <em>NOT</em> an
 * atomic snapshot; invocation in the absence of concurrent
 * updates returns an accurate result, but concurrent updates that
 * occur while the sum is being calculated might not be
 * incorporated.
 *
 * @return the sum
 */
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

```java
/**
 * Resets variables maintaining the sum to zero.  This method may
 * be a useful alternative to creating a new adder, but is only
 * effective if there are no concurrent updates.  Because this
 * method is intrinsically racy, it should only be used when it is
 * known that no threads are concurrently updating.
 */
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

```java
/**
 * Equivalent in effect to {@link #sum} followed by {@link
 * #reset}. This method may apply for example during quiescent
 * points between multithreaded computations.  If there are
 * updates concurrent with this method, the returned value is
 * <em>not</em> guaranteed to be the final value occurring before
 * the reset.
 *
 * @return the sum
 */
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
