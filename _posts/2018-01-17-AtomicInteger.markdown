---
layout:     post
title:      "Java源码系列(7) -- AtomicInteger"
date:       2018-01-17
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、前言

AtomicInteger基于CAS(Compare and Swap，比较并修改)的操作，主要实现乐观锁的思想。

对于传统的悲观锁来说，会假设线程并发非常重，每次修改数据，一定先100%确保自己进入安全区，再安心修改目标值。进而出现线程在竞争锁的过程中消耗大量时间在等待锁、加锁、解锁等操作上。（注：锁还可能涉及锁自旋、公平锁等知识点，而非简单暴力竞争）

相比之下的乐观锁，会假设只有自己一个线程修改目标值，先比较修改前的值是不是和自己的预期的一致，一致就修改并返回，不一致就放弃这次修改（CAS），并发起下一次的尝试，不会把时间用在加锁和解锁上。

虽然乐观锁看起来比悲观锁好很多，不过乐观锁主要用在单个值（Int，Float，Double）的并发修改上，而不是悲观锁对一个对象甚至是一个代码块的操作。

其次，若乐观锁对目标值修改操作次数远多于读取操作，那么CAS实际也会大量失败抵消CAS的优点，并演变成成多次重试失效。

说到CAS，需要了解的是这个能力并非由操作系统或JVM提供，而是CPU原生支持的。如果一个CPU不能保证CAS能力，那这个CPU的完全没有数据修改安全可言。

说回`AtomicInteger`，这类不能看见底层的运作，因为主要调用了`Unsafe`实现CAS。以后会写关于`Unsafe`的文章，敬请期待。

## 二、类签名

由于集成了`Number`，所以任何能接受`Number`类型的形参都能使用`AtomicInteger`

```java
public class AtomicInteger extends Number implements java.io.Serializable
```

## 三、静态初始化

在静态初始化块里面获取`value`的内存地址，这时的`value`内存地址已经开辟，但是没有被实例初始化。而静态初始化块是类初始化最早调用的，静态初始化安全由JVM来保证。

```java
static {
    try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}
```

## 四、数据成员

volatile保证value值的有序性和可见性，不保证原子性。原子性一般由synchronized或Lock来提供支持。

```java
private volatile int value;

// setup to use Unsafe.compareAndSwapInt for updates
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;
```

## 五、构造方法

```java
// 用一个给定的整形值初始化一个AtomicInteger实例
public AtomicInteger(int initialValue) {
    value = initialValue;
}

// 初始化一个值为0的AtomicInteger实例
public AtomicInteger() {
}
```

## 六、成员方法

```java
// 获取当前的整形值，线程不安全
public final int get() {
    return value;
}

// 设置新的整形值，线程不安全
public final void set(int newValue) {
    value = newValue;
}

// 最终一定会把newValue设置成功
public final void lazySet(int newValue) {
    unsafe.putOrderedInt(this, valueOffset, newValue);
}

// 设置新的整形值，把返回上一个保存的值
public final int getAndSet(int newValue) {
    return unsafe.getAndSetInt(this, valueOffset, newValue);
}
```

如果待修改的值和期待值相同，那就把待修改的值设置为update的值
伪代码： `value == expect ? value = update; return isModified;`

```java
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}

/**
 * Atomically sets the value to the given updated value
 * if the current value {@code ==} the expected value.
 *
 * <p><a href="package-summary.html#weakCompareAndSet">May fail
 * spuriously and does not provide ordering guarantees</a>, so is
 * only rarely an appropriate alternative to {@code compareAndSet}.
 *
 * @param expect the expected value
 * @param update the new value
 * @return {@code true} if successful
 */
public final boolean weakCompareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

```java
// 先返回上一个值，然后再在原基础上自增1
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}

// 先返回上一个值，然后再在原基础上自减1
public final int getAndDecrement() {
    return unsafe.getAndAddInt(this, valueOffset, -1);
}

// 返回上一个值，并在原基础上加上指定值
// 伪代码： oldValue = value; value += delta; return oldValue; 
public final int getAndAdd(int delta) {
    return unsafe.getAndAddInt(this, valueOffset, delta);
}

// 先自增，然后返回自增后的值
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}

// 先自减，然后返回自减后的值
public final int decrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
}

// 先增加delta的值，然后返回增加后的值
public final int addAndGet(int delta) {
    return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
}
```

## 七、Java8 Lambda支持

`IntUnaryOperator -> This is a functional interface and can therefore be used as the assignment target for a lambda expression or method reference.`

```java
/**
 * Atomically updates the current value with the results of
 * applying the given function, returning the previous value. The
 * function should be side-effect-free, since it may be re-applied
 * when attempted updates fail due to contention among threads.
 *
 * @param updateFunction a side-effect-free function
 * @return the previous value
 * @since 1.8
 */
public final int getAndUpdate(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get();
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(prev, next));
    return prev;
}

/**
 * Atomically updates the current value with the results of
 * applying the given function, returning the updated value. The
 * function should be side-effect-free, since it may be re-applied
 * when attempted updates fail due to contention among threads.
 *
 * @param updateFunction a side-effect-free function
 * @return the updated value
 * @since 1.8
 */
public final int updateAndGet(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get();
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(prev, next));
    return next;
}

/**
 * Atomically updates the current value with the results of
 * applying the given function to the current and given values,
 * returning the previous value. The function should be
 * side-effect-free, since it may be re-applied when attempted
 * updates fail due to contention among threads.  The function
 * is applied with the current value as its first argument,
 * and the given update as the second argument.
 *
 * @param x the update value
 * @param accumulatorFunction a side-effect-free function of two arguments
 * @return the previous value
 * @since 1.8
 */
public final int getAndAccumulate(int x,
                                  IntBinaryOperator accumulatorFunction) {
    int prev, next;
    do {
        prev = get();
        next = accumulatorFunction.applyAsInt(prev, x);
    } while (!compareAndSet(prev, next));
    return prev;
}

/**
 * Atomically updates the current value with the results of
 * applying the given function to the current and given values,
 * returning the updated value. The function should be
 * side-effect-free, since it may be re-applied when attempted
 * updates fail due to contention among threads.  The function
 * is applied with the current value as its first argument,
 * and the given update as the second argument.
 *
 * @param x the update value
 * @param accumulatorFunction a side-effect-free function of two arguments
 * @return the updated value
 * @since 1.8
 */
public final int accumulateAndGet(int x,
                                  IntBinaryOperator accumulatorFunction) {
    int prev, next;
    do {
        prev = get();
        next = accumulatorFunction.applyAsInt(prev, x);
    } while (!compareAndSet(prev, next));
    return next;
}
```

## 八、参考链接

[https://docs.oracle.com/javase/8/docs/api/java/util/function/IntUnaryOperator.html](https://docs.oracle.com/javase/8/docs/api/java/util/function/IntUnaryOperator.html)

[https://docs.oracle.com/javase/8/docs/api/java/util/function/IntBinaryOperator.html](https://docs.oracle.com/javase/8/docs/api/java/util/function/IntBinaryOperator.html)

[https://en.wikipedia.org/wiki/Compare-and-swap](https://en.wikipedia.org/wiki/Compare-and-swap)

