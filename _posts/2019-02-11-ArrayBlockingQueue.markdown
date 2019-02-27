---
layout:     post
title:      "Java源码系列(22) -- ArrayBlockingQueue"
date:       2019-02-11
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、类签名

__ArrayBlockingQueue__ 为有界数组。队列元素顺序为先进先出。队头指针元素在队列内保存时间最长，队尾指针元素在队列内保存时间最短。新元素插入到队尾，而遍历操作则从队头获取元素。

 ```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable
 ```

这是传统的有界缓冲，元素保存在固定大小的数组中，由生产者插入元素、消费者取出元素。一旦创建成功，数组容量不能再修改。向已满的队列存入元素会导致操作阻塞，同样，从空队列中取出元素也会引起阻塞。

本队列支持可选的公平策略，以便安排正在等待的生产者和消费者。该公平特性默认是不保证的。然而，创建实例时把 __fairness__ 设为 __true__ ，能保证线程按照FIFO的方式获取。公平特性会降低吞吐量，但也能降低可变性并避免线程饥饿。

源码版本：JDK11

## 二、成员变量

```java
// 保存元素的数组
final Object[] items;

// 下次可执行take、poll、peek或remove操作的位置索引
int takeIndex;

// 下次可执行put、offer或add操作的位置索引
int putIndex;

// 队列已保存元素数量
int count;

final ReentrantLock lock;

/** Condition for waiting takes */
private final Condition notEmpty;

/** Condition for waiting puts */
private final Condition notFull;
```

## 三、构造方法

构造方法内初始化保存元素的环形数组，同时也初始化同步锁。形参 __fair__ 控制队列是否支持公平锁，一般建议使用非公平锁。因为公平锁为了平衡读写线程的优先级，需要牺牲吞吐量。

```java
// 支持自定义公平锁
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}

// 使用指定集合初始化队列，支持自定义公平锁
public ArrayBlockingQueue(int capacity, boolean fair,
                          Collection<? extends E> c) {
    // 先调用上述构造方法进行初始化
    this(capacity, fair);

    // 获取锁，处理集合c的元素
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        final Object[] items = this.items;
        int i = 0;
        try {
            // 把指定集合的元素放入队列
            for (E e : c)
                items[i++] = Objects.requireNonNull(e);
        } catch (ArrayIndexOutOfBoundsException ex) {
            // 如果c的元素数量超过capacity，抛出ArrayIndexOutOfBoundsException
            throw new IllegalArgumentException();
        }
        // 更新已保存元素数量
        count = i;
        // 更新存入指针的索引
        putIndex = (i == capacity) ? 0 : i;
    } finally {
        // 操作完成，解锁
        lock.unlock();
    }
}
```

## 四、实例方法

#### 4.1 增

```java
// 在指定索引插入元素，方法在调用前需要先获取锁
private void enqueue(E e) {
    final Object[] items = this.items;
    // 索引插入元素
    items[putIndex] = e;
    // 计算下一个插入元素的有效索引
    if (++putIndex == items.length) putIndex = 0;
    // 已保存元素递增
    count++;
    notEmpty.signal();
}

// 如果没有超过队列容量，则指定元素直接插入到队列的尾部
// 此方法插入失败直接抛出异常，所以建议使用方法offer(E e)
public boolean add(E e) {
    return super.add(e);
}

// 如果没有超过队列容量，则指定元素直接插入到队列的尾部
public boolean offer(E e) {
    // 插入元素不能为空
    Objects.requireNonNull(e);
    final ReentrantLock lock = this.lock;
    // 先获取锁
    lock.lock();
    try {
        if (count == items.length)
            return false;
        else {
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}

// 插入指定元素到队列的尾部，如果队列空间已满则阻塞等待
public void put(E e) throws InterruptedException {
    Objects.requireNonNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        // 循环一直等待队列出现空间
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
        lock.unlock();
    }
}

// 插入指定元素到队列的尾部，如果队列空间已满则阻塞等待，可自定义超时时间
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    Objects.requireNonNull(e);
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        // 队列没有空间则继续等待
        while (count == items.length) {
            if (nanos <= 0L)
                // 元素没有插入，返回false
                return false;
            // 计算剩余超时时间
            nanos = notFull.awaitNanos(nanos);
        }
        // 队列出现空余且操作没有超时，元素存入队列
        enqueue(e);
        // 成功插入元素，返回true
        return true;
    } finally {
        lock.unlock();
    }
}
```

#### 4.2 取

```java
// 从takeIndex索引获取元素，由take()方法调用
private E dequeue() {
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    // 获取索引的元素，即头引用的元素
    E e = (E) items[takeIndex];
    items[takeIndex] = null;
    // 索引后移
    if (++takeIndex == items.length) takeIndex = 0;
    // 已保存元素数量递减
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal();
    return e;
}

// 从队列取元素
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        // 循环一直等待队列非空
        while (count == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
}

// 从队列取元素
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}

// 从队列取元素，如果队列空间已空则阻塞等待，可自定义超时时间
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0) {
            if (nanos <= 0L)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        return dequeue();
    } finally {
        lock.unlock();
    }
}

public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return itemAt(takeIndex);
    } finally {
        lock.unlock();
    }
}
```

#### 4.3 查

因为是个环形数组，所以查找时需要注意开始索引和结束索引

```java
public boolean contains(Object o) {
    if (o == null) return false;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 快速失败，如果队列为空，不进行任何查找操作
        if (count > 0) {
            final Object[] items = this.items;
            for (int i = takeIndex, end = putIndex,
                     to = (i < end) ? end : items.length;
                 ; i = 0, to = end) {
                // 先遍历唤醒数组的后段，元素没有命中，再遍历数组前段
                for (; i < to; i++)
                    if (o.equals(items[i]))
                        return true;
                // 全部元素遍历完成，没有找到目标元素
                if (to == end) break;
            }
        }
        // 队列为空，或队列不存在目标元素
        return false;
    } finally {
        lock.unlock();
    }
}
```

#### 4.4 清除

```java
// 根据索引移出元素
void removeAt(final int removeIndex) {
    final Object[] items = this.items;
    if (removeIndex == takeIndex) {
        // 移除的元素是队列头元素，该元素置空，指针后移即可
        items[takeIndex] = null;
        if (++takeIndex == items.length) takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
    } else {
        // 移除的元素在环形队列内部
        for (int i = removeIndex, putIndex = this.putIndex;;) {
            int pred = i;
            if (++i == items.length) i = 0;
            // 移除中间的元素
            if (i == putIndex) {
                items[pred] = null;
                this.putIndex = pred;
                break;
            }
            // 后续元素需逐一向前移动一个位置
            items[pred] = items[i];
        }
        count--;
        if (itrs != null)
            itrs.removedAt(removeIndex);
    }
    notFull.signal();
}

// 从队列中移除指定元素
public boolean remove(Object o) {
    if (o == null) return false;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        if (count > 0) {
            final Object[] items = this.items;
            for (int i = takeIndex, end = putIndex,
                     to = (i < end) ? end : items.length;
                 ; i = 0, to = end) {
                for (; i < to; i++)
                    // 要先遍历队列才能找到指定元素
                    if (o.equals(items[i])) {
                        // 元素命中则移除命中目标
                        removeAt(i);
                        // 移除成功返回true
                        return true;
                    }
                // 队列已经完全遍历完毕，没有发现指定元素，退出遍历
                if (to == end) break;
            }
        }
        // 没有找到指定元素
        return false;
    } finally {
        lock.unlock();
    }
}

public void clear() {
    final ReentrantLock lock = this.lock;
    // 获取锁，开始清除所有元素
    lock.lock();
    try {
        int k;
        if ((k = count) > 0) {
            circularClear(items, takeIndex, putIndex);
            // 重置takeIndex和putIndex
            takeIndex = putIndex;
            // 已保存元素数量置0
            count = 0;
            // 清除迭代器
            if (itrs != null)
                itrs.queueIsEmpty();
            // 通知所有等待的Waiters
            for (; k > 0 && lock.hasWaiters(notFull); k--)
                notFull.signal();
        }
    } finally {
        lock.unlock();
    }
}

// 清除环形队列内所有元素
private static void circularClear(Object[] items, int i, int end) {
    for (int to = (i < end) ? end : items.length;
         ; i = 0, to = end) {
        for (; i < to; i++) items[i] = null;
        if (to == end) break;
    }
}
```

#### 4.5 取余计算

由于这个是环形队列，所以需要处理队列尾部和队列头部索引的关系

```java
// 处理队列尾部递增的情况
static final int inc(int i, int modulus) {
    if (++i >= modulus) i = 0;
    return i;
}

// 处理队列头部递减的情况
static final int dec(int i, int modulus) {
    if (--i < 0) i = modulus - 1;
    return i;
}
```


