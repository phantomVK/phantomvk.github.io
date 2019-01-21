---
layout:     post
title:      "ArrayBlockingQueue"
subtitle:   ""
date:       2018-01-01
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - tags
---

JDK11

## 类签名

__BlockingQueue__ 是利用数组实现的有界数组。队列元素按照先进先出的规则管理。队列头部的元素是保存在队列事件最长的元素。队列尾部的元素是保存在队列中时间最短的元素。新元素插入到队列的尾部，二队列遍历操作从队列的头部获取元素。

这是传统的游街缓冲，元素保存在固定大小的数组中，由生产者插入元素并由消费者取出元素。一旦元素创建成功，元素内容不能被修改。尝试向一个已满的队列放入元素会引起阻塞，同样从空队列中多去元素也会出现阻塞。

```java
/**
 * A bounded {@linkplain BlockingQueue blocking queue} backed by an
 * array.  This queue orders elements FIFO (first-in-first-out).  The
 * <em>head</em> of the queue is that element that has been on the
 * queue the longest time.  The <em>tail</em> of the queue is that
 * element that has been on the queue the shortest time. New elements
 * are inserted at the tail of the queue, and the queue retrieval
 * operations obtain elements at the head of the queue.
 *
 * <p>This is a classic &quot;bounded buffer&quot;, in which a
 * fixed-sized array holds elements inserted by producers and
 * extracted by consumers.  Once created, the capacity cannot be
 * changed.  Attempts to {@code put} an element into a full queue
 * will result in the operation blocking; attempts to {@code take} an
 * element from an empty queue will similarly block.
 */
```

此类支持可选的公平策略安排正在等待的生产者和消费者。然而，把 __fairness__ 设为 __true__ 的保证线程通过先进先出的方式获取。公平特性一般会减低吞吐量，但能降低可变性和避免线程饥饿。

 ```java
/**
 * <p>This class supports an optional fairness policy for ordering
 * waiting producer and consumer threads.  By default, this ordering
 * is not guaranteed. However, a queue constructed with fairness set
 * to {@code true} grants threads access in FIFO order. Fairness
 * generally decreases throughput but reduces variability and avoids
 * starvation.
 */
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable
 ```

## 成员变量

```java
// 队列元素
final Object[] items;

// 下次可执行take、poll、peek或remove操作的位置索引
int takeIndex;

// 下次可执行put、offer或add操作的位置索引
int putIndex;

// 队列以保存元素数量
int count;
```

## 锁

```java
final ReentrantLock lock;

/** Condition for waiting takes */
private final Condition notEmpty;

/** Condition for waiting puts */
private final Condition notFull;
```

```java
/**
 * Increments i, mod modulus.
 * Precondition and postcondition: 0 <= i < modulus.
 */
static final int inc(int i, int modulus) {
    if (++i >= modulus) i = 0;
    return i;
}

/**
 * Decrements i, mod modulus.
 * Precondition and postcondition: 0 <= i < modulus.
 */
static final int dec(int i, int modulus) {
    if (--i < 0) i = modulus - 1;
    return i;
}
```

```java
// 根据指定索引获取数组中的变量值
@SuppressWarnings("unchecked")
final E itemAt(int i) {
    return (E) items[i];
}

/**
 * Returns element at array index i.
 * This is a slight abuse of generics, accepted by javac.
 */
@SuppressWarnings("unchecked")
static <E> E itemAt(Object[] items, int i) {
    return (E) items[i];
}

/**
 * Inserts element at current put position, advances, and signals.
 * Call only when holding lock.
 */
private void enqueue(E e) {
    // assert lock.isHeldByCurrentThread();
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items;
    items[putIndex] = e;
    if (++putIndex == items.length) putIndex = 0;
    count++;
    notEmpty.signal();
}

/**
 * Extracts element at current take position, advances, and signals.
 * Call only when holding lock.
 */
private E dequeue() {
    // assert lock.isHeldByCurrentThread();
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E e = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length) takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal();
    return e;
}

/**
 * Deletes item at array index removeIndex.
 * Utility for remove(Object) and iterator.remove.
 * Call only when holding lock.
 */
void removeAt(final int removeIndex) {
    // assert lock.isHeldByCurrentThread();
    // assert lock.getHoldCount() == 1;
    // assert items[removeIndex] != null;
    // assert removeIndex >= 0 && removeIndex < items.length;
    final Object[] items = this.items;
    if (removeIndex == takeIndex) {
        // removing front item; just advance
        items[takeIndex] = null;
        if (++takeIndex == items.length) takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
    } else {
        // an "interior" remove

        // slide over all others up through putIndex.
        for (int i = removeIndex, putIndex = this.putIndex;;) {
            int pred = i;
            if (++i == items.length) i = 0;
            if (i == putIndex) {
                items[pred] = null;
                this.putIndex = pred;
                break;
            }
            items[pred] = items[i];
        }
        count--;
        if (itrs != null)
            itrs.removedAt(removeIndex);
    }
    notFull.signal();
}
```

## 构造方法

```java
/**
 * Creates an {@code ArrayBlockingQueue} with the given (fixed)
 * capacity and the specified access policy.
 *
 * @param capacity the capacity of this queue
 * @param fair if {@code true} then queue accesses for threads blocked
 *        on insertion or removal, are processed in FIFO order;
 *        if {@code false} the access order is unspecified.
 * @throws IllegalArgumentException if {@code capacity < 1}
 */
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}

/**
 * Creates an {@code ArrayBlockingQueue} with the given (fixed)
 * capacity, the specified access policy and initially containing the
 * elements of the given collection,
 * added in traversal order of the collection's iterator.
 *
 * @param capacity the capacity of this queue
 * @param fair if {@code true} then queue accesses for threads blocked
 *        on insertion or removal, are processed in FIFO order;
 *        if {@code false} the access order is unspecified.
 * @param c the collection of elements to initially contain
 * @throws IllegalArgumentException if {@code capacity} is less than
 *         {@code c.size()}, or less than 1.
 * @throws NullPointerException if the specified collection or any
 *         of its elements are null
 */
public ArrayBlockingQueue(int capacity, boolean fair,
                          Collection<? extends E> c) {
    this(capacity, fair);

    final ReentrantLock lock = this.lock;
    lock.lock(); // Lock only for visibility, not mutual exclusion
    try {
        final Object[] items = this.items;
        int i = 0;
        try {
            for (E e : c)
                items[i++] = Objects.requireNonNull(e);
        } catch (ArrayIndexOutOfBoundsException ex) {
            throw new IllegalArgumentException();
        }
        count = i;
        putIndex = (i == capacity) ? 0 : i;
    } finally {
        lock.unlock();
    }
}
```

## 实例方法

```java
/**
 * Inserts the specified element at the tail of this queue if it is
 * possible to do so immediately without exceeding the queue's capacity,
 * returning {@code true} upon success and throwing an
 * {@code IllegalStateException} if this queue is full.
 *
 * @param e the element to add
 * @return {@code true} (as specified by {@link Collection#add})
 * @throws IllegalStateException if this queue is full
 * @throws NullPointerException if the specified element is null
 */
public boolean add(E e) {
    return super.add(e);
}

/**
 * Inserts the specified element at the tail of this queue if it is
 * possible to do so immediately without exceeding the queue's capacity,
 * returning {@code true} upon success and {@code false} if this queue
 * is full.  This method is generally preferable to method {@link #add},
 * which can fail to insert an element only by throwing an exception.
 *
 * @throws NullPointerException if the specified element is null
 */
public boolean offer(E e) {
    Objects.requireNonNull(e);
    final ReentrantLock lock = this.lock;
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

/**
 * Inserts the specified element at the tail of this queue, waiting
 * for space to become available if the queue is full.
 *
 * @throws InterruptedException {@inheritDoc}
 * @throws NullPointerException {@inheritDoc}
 */
// 插入指定元素到队列的尾部，如果队列空间已满则阻塞等待
public void put(E e) throws InterruptedException {
    Objects.requireNonNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
        lock.unlock();
    }
}

/**
 * Inserts the specified element at the tail of this queue, waiting
 * up to the specified wait time for space to become available if
 * the queue is full.
 *
 * @throws InterruptedException {@inheritDoc}
 * @throws NullPointerException {@inheritDoc}
 */
// 插入指定元素到队列的尾部，如果队列空间已满则阻塞等待，可自定义超时时间
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    Objects.requireNonNull(e);
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length) {
            if (nanos <= 0L)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
        enqueue(e);
        return true;
    } finally {
        lock.unlock();
    }
}
```

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}

public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
}

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
        return itemAt(takeIndex); // null when queue is empty
    } finally {
        lock.unlock();
    }
}
```

```java
/**
 * Removes a single instance of the specified element from this queue,
 * if it is present.  More formally, removes an element {@code e} such
 * that {@code o.equals(e)}, if this queue contains one or more such
 * elements.
 * Returns {@code true} if this queue contained the specified element
 * (or equivalently, if this queue changed as a result of the call).
 *
 * <p>Removal of interior elements in circular array based queues
 * is an intrinsically slow and disruptive operation, so should
 * be undertaken only in exceptional circumstances, ideally
 * only when the queue is known not to be accessible by other
 * threads.
 *
 * @param o element to be removed from this queue, if present
 * @return {@code true} if this queue changed as a result of the call
 */
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
                    if (o.equals(items[i])) {
                        removeAt(i);
                        return true;
                    }
                if (to == end) break;
            }
        }
        return false;
    } finally {
        lock.unlock();
    }
}

/**
 * Returns {@code true} if this queue contains the specified element.
 * More formally, returns {@code true} if and only if this queue contains
 * at least one element {@code e} such that {@code o.equals(e)}.
 *
 * @param o object to be checked for containment in this queue
 * @return {@code true} if this queue contains the specified element
 */
public boolean contains(Object o) {
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
                    if (o.equals(items[i]))
                        return true;
                if (to == end) break;
            }
        }
        return false;
    } finally {
        lock.unlock();
    }
}
```

```java
/**
 * Atomically removes all of the elements from this queue.
 * The queue will be empty after this call returns.
 */
public void clear() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        int k;
        if ((k = count) > 0) {
            circularClear(items, takeIndex, putIndex);
            takeIndex = putIndex;
            count = 0;
            if (itrs != null)
                itrs.queueIsEmpty();
            for (; k > 0 && lock.hasWaiters(notFull); k--)
                notFull.signal();
        }
    } finally {
        lock.unlock();
    }
}

/**
 * Nulls out slots starting at array index i, upto index end.
 * Condition i == end means "full" - the entire array is cleared.
 */
private static void circularClear(Object[] items, int i, int end) {
    // assert 0 <= i && i < items.length;
    // assert 0 <= end && end < items.length;
    for (int to = (i < end) ? end : items.length;
         ; i = 0, to = end) {
        for (; i < to; i++) items[i] = null;
        if (to == end) break;
    }
}
```


