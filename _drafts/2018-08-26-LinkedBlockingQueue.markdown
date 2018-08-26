---
layout:     post
title:      "Java源码系列(17) -- LinkedBlockingQueue"
date:       2018-08-26
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、类签名

从类名可知，LinkedBlockingQueue是基于链表实现的阻塞队列。

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable
```

类特点：

- 基于链表实现的阻塞队列，线程安全；
- 队头元素是存活时间最长的元素，队尾元素是存活时间最短的元素；
- 元素从队列尾进队，从队列头出队，符合FIFO；
- 链表实现的队列一般比基于数组实现有更高吞吐量，但比大多数并发应用的理想性能低；
- 默认队列最大长度为Integer.MAX_VALUE，新节点动态创建，已保存节点不会超过此值；
- 此类和其迭代器均实现了`Collection`和`Iterator`接口的可选方法；

这是`two lock queue`算法的变体。putLock守卫元素的put、offer操作，且与等待存入的条件关联。takeLock原理类似。putLock和takeLock都依赖的`count`变量，被维护为一个原子变量，yi以避免多数情况下需同时请求两个锁。为了最小化put时需获取takeLock，使用了层叠式通知。当put操作注意到至少一个take可以启动，就会通知获取者。如果有多个item在信号后进队，获取者将依次通知其他获取者。因此，有对称的取操作通知存操作。有些操作如remove和iterators会同时请求两个锁。

读取者和写入者间可见性如下提供：

当元素已进入队列，putLock已被获取，且count变量也更新。随后，读取者通过获取putLock或获取takeLock，得到入队元素的可见性，然后读取`n = count.get();`，令前n个元素变得可见。

为实现弱一致性迭代器，显然要从前导出队节点上保持所有节点的GC可达性。这会引起两个问题：

1. 允许一个异常的迭代器触发无限制的内存保留；
2. 如果节点在老年代存活，会引起老节点和新节点间的跨代连接，令分代垃圾回收难以进行，导致重复老年代回收(major collections)；

然而，只有未删除节点需要从已出队节点可达，GC不需要理解可达性的类型。我们使用一些小手段连接一个刚刚被出队的节点。

源码来自JDK10。

## 二、节点

单向链表节点类，查找元素需要从链表头开始依次遍历节点

```java
static class Node<E> {
    E item; // 节点数据

    Node<E> next; // 下一节点

    Node(E x) { item = x; } // 构造方法，存入节点的内容
}
```

## 三、数据成员

队列容量，默认为Integer.MAX_VALUE

```java
private final int capacity;
```

已存元素总数

```java
private final AtomicInteger count = new AtomicInteger();
```

链表头节点

```java
transient Node<E> head;
```

链表尾节点

```java
private transient Node<E> last;
```

## 四、锁成员

此锁被take，poll等操作持有

```java
private final ReentrantLock takeLock = new ReentrantLock();
```

takes的等待队列

```java
private final Condition notEmpty = takeLock.newCondition();
```

此锁被put，offer等持有

```java
private final ReentrantLock putLock = new ReentrantLock();
```

puts的等待队列

```java
private final Condition notFull = putLock.newCondition();
```

## 五、锁操作

唤醒notEmpty队列上正在等待获取元素的线程，此方法由put/offer调用。

```java
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}
```

唤醒notFull队列上正在等待存入元素的线程，此方法由take/poll调用。

```java
private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        notFull.signal();
    } finally {
        putLock.unlock();
    }
}
```

## 六、进队出队

把节点插入到链表尾

```java
private void enqueue(Node<E> node) {
    // assert putLock.isHeldByCurrentThread();
    // assert last.next == null;
    last = last.next = node;
}
```

队头元素出队

```java
private E dequeue() {
    // assert takeLock.isHeldByCurrentThread();
    // assert head.item == null;
    Node<E> h = head;
    Node<E> first = h.next;
    h.next = h; // 帮助GC
    head = first;
    E x = first.item;
    first.item = null;
    return x;
}
```

## 七、读写锁操作

上锁禁止存取操作

```java
void fullyLock() {
    putLock.lock();
    takeLock.lock();
}
```

解锁允许存取操作

```java
void fullyUnlock() {
    takeLock.unlock();
    putLock.unlock();
}
```

## 八、构造方法


默认构造方法，最大容量为Integer.MAX_VALUE

```java
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}
```

构造方法，初始化头结点引用和为节点引用，使用指定capacity

```java
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    // 自定义队列capacity
    this.capacity = capacity;
    // 头指针和尾指针指向同一个空节点
    last = head = new Node<E>(null);
}
```

通过集合实例初始化类，最大容量为Integer.MAX_VALUE

```java
public LinkedBlockingQueue(Collection<? extends E> c) {
    this(Integer.MAX_VALUE); // capacity设置为Integer.MAX_VALUE
    final ReentrantLock putLock = this.putLock;
    putLock.lock(); // 没有竞争，但对可见性来说有必要
    try {
        int n = 0;
        // 依次遍历集合c
        for (E e : c) {
            if (e == null)
                // 集合元素为空抛出NullPointerException
                throw new NullPointerException();
            if (n == capacity)
                throw new IllegalStateException("Queue full");
            // 在集合c中取元素，用元素创建节点存入队列
            enqueue(new Node<E>(e));
            // 添加元素递增
            ++n;
        }
        // 最后把总添加元素更新至原子值count
        count.set(n);
    } finally {
        putLock.unlock();
    }
}
```

## 九、成员方法

获取队已存元素数量

```java
public int size() {
    return count.get();
}
```

剩余可用队列容量

```java
public int remainingCapacity() {
    return capacity - count.get();
}
```

插入到队列尾部，直到成功插入才返回成功

```java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    // Note: convention in all put/take/etc is to preset local var
    // holding count negative to indicate failure unless set.
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    // 此锁可以被中断
    putLock.lockInterruptibly();
    try {
        while (count.get() == capacity) {
            notFull.await();
        }
        // 元素插入到队列尾部
        enqueue(node);
        // 已保存元素数量递增
        c = count.getAndIncrement();
        // 检查队列是否已满
        if (c + 1 < capacity)
            // 通知其他线程继续插入元素
            notFull.signal();
    } finally {
        // 解除锁
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
}
```

在队列尾部插入指定元素，如果在指定超时时间内插入成功返回true，否则返回false

```java
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    // e为空抛出NullPointerException
    if (e == null) throw new NullPointerException();
    // 根据超时值和时间单位转换为纳秒时长
    long nanos = unit.toNanos(timeout);
    int c = -1;
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        while (count.get() == capacity) {
            if (nanos <= 0L)
                return false;
            nanos = notFull.awaitNanos(nanos); // 时间倒计时
        }
        enqueue(new Node<E>(e));
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return true;
}
```

把指定元素插入到队尾，如果插入成功返回true，队列已满插入失败返回false。当使用有容量限制的队列时，这个方法是比add方法更好，因为add元素添加失败会抛出异常。存入元素e为空时抛出NullPointerException。

```java
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    final AtomicInteger count = this.count;
    if (count.get() == capacity)
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        if (count.get() < capacity) {
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return c >= 0;
}
```

获取元素

```java
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {
            notEmpty.await();
        }
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```

在指定等待超时时间内获取元素，到达超时时间没有则返回null

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E x = null;
    int c = -1;
    long nanos = unit.toNanos(timeout);
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {
            if (nanos <= 0L)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```

获取元素

```java
public E poll() {
    // 获取元素数量
    final AtomicInteger count = this.count;
    // 队列中没有元素则返回null
    if (count.get() == 0)
        return null;
    // 队列中有元素，开始以下逻辑
    E x = null;
    int c = -1;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        if (count.get() > 0) {
            // 元素从对头出队
            x = dequeue();
            // 队列元素数量递减
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        }
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```

返回队列头节点包含的数据，此操作不会改变队列节点的数量或顺序

```java
public E peek() {
    // 队列没有节点直接返回null
    if (count.get() == 0)
        return null;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        // 返回对头元素的内容，否则返回null
        return (count.get() > 0) ? head.next.item : null;
    } finally {
        takeLock.unlock();
    }
}
```

把节点p从列表中解除链接，pred是p的上一个节点

```java
void unlink(Node<E> p, Node<E> pred) {
    // assert putLock.isHeldByCurrentThread();
    // assert takeLock.isHeldByCurrentThread();
    // p.next is not changed, to allow iterators that are
    // traversing p to maintain their weak-consistency guarantee.
    p.item = null; // 节点p的内容置空
    pred.next = p.next; // 操作p前后节点，令p解除链接
    if (last == p) // 如果p是尾节点，则调整尾指针的指向
        last = pred;
    if (count.getAndDecrement() == capacity)
        notFull.signal();
}
```

如果队列中存在指定元素，则把该元素从队列中移除。即使队列中存在多个相同元素，此方法只会移除其中一个。移除成功返回true，否则返回false。

```java
public boolean remove(Object o) {
    // 元素为null直接返回false
    if (o == null) return false;
    fullyLock();
    try {
        for (Node<E> pred = head, p = pred.next;
             p != null;
             pred = p, p = p.next) {
            // 在链表上逐个查找元素是否匹配
            if (o.equals(p.item)) {
                // 找到匹配元素则把该元素解除链接
                unlink(p, pred);
                // 元素返回true
                return true;
            }
        }
        // 没有移除元素
        return false;
    } finally {
        fullyUnlock();
    }
}
```

检查是否包含指定元素

```java
public boolean contains(Object o) {
    // 元素为null直接返回false
    if (o == null) return false;
    fullyLock();
    try {
        // 遍历队列逐个查找元素
        for (Node<E> p = head.next; p != null; p = p.next)
            // 找到匹配元素
            if (o.equals(p.item))
                return true;
        // 找不到匹配元素
        return false;
    } finally {
        fullyUnlock();
    }
}
```

返回一个数组，此数组包含队列的所有元素，且数组元素的顺序和队列的元素的顺序一致。每次返回的数组为不同对象，修改数组是安全的。

```java
public Object[] toArray() {
    fullyLock();
    try {
        // 获取队列中元素个数
        int size = count.get();
        // 通过元素数量构造数组
        Object[] a = new Object[size];
        int k = 0;
        // 遍历队列，一次拷贝元素引用到数组对应索引
        for (Node<E> p = head.next; p != null; p = p.next)
            a[k++] = p.item;
        // 返回数组
        return a;
    } finally {
        fullyUnlock();
    }
}
```

返回一个数组，此数组包含队列的所有元素，且数组元素的顺序和队列的元素的顺序一致。如果传入数组空间足够，返回的数组就是传入的数组。否则方法内存创建类型相同新数组，数组长度和被队列长度相等。

```java
/**
 * Returns an array containing all of the elements in this queue, in
 * proper sequence; the runtime type of the returned array is that of
 * the specified array.  If the queue fits in the specified array, it
 * is returned therein.  Otherwise, a new array is allocated with the
 * runtime type of the specified array and the size of this queue.
 *
 * <p>If this queue fits in the specified array with room to spare
 * (i.e., the array has more elements than this queue), the element in
 * the array immediately following the end of the queue is set to
 * {@code null}.
 *
 * <p>Like the {@link #toArray()} method, this method acts as bridge between
 * array-based and collection-based APIs.  Further, this method allows
 * precise control over the runtime type of the output array, and may,
 * under certain circumstances, be used to save allocation costs.
 *
 * <p>Suppose {@code x} is a queue known to contain only strings.
 * The following code can be used to dump the queue into a newly
 * allocated array of {@code String}:
 *
 * <pre> {@code String[] y = x.toArray(new String[0]);}</pre>
 *
 * Note that {@code toArray(new Object[0])} is identical in function to
 * {@code toArray()}.
 *
 * @param a the array into which the elements of the queue are to
 *          be stored, if it is big enough; otherwise, a new array of the
 *          same runtime type is allocated for this purpose
 * @return an array containing all of the elements in this queue
 * @throws ArrayStoreException if the runtime type of the specified array
 *         is not a supertype of the runtime type of every element in
 *         this queue
 * @throws NullPointerException if the specified array is null
 */
@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a) {
    fullyLock();
    try {
        int size = count.get();
        if (a.length < size)
            a = (T[])java.lang.reflect.Array.newInstance
                (a.getClass().getComponentType(), size);

        int k = 0;
        for (Node<E> p = head.next; p != null; p = p.next)
            a[k++] = (T)p.item;
        if (a.length > k)
            a[k] = null;
        return a;
    } finally {
        fullyUnlock();
    }
}
```

移除队列中所有元素，移除完成后队列元素为空

```java
public void clear() {
    // 上锁
    fullyLock();
    try {
        for (Node<E> p, h = head; (p = h.next) != null; h = p) {
            h.next = h;
            p.item = null; // 置空节点的数据
        }
        head = last; // 由于队列元素全清空了，所以头指针和为指针引用相同
        // assert head.item == null && head.next == null;
        if (count.getAndSet(0) == capacity)
            notFull.signal();
    } finally {
        fullyUnlock();
    }
}
```

```java
public int drainTo(Collection<? super E> c) {
    return drainTo(c, Integer.MAX_VALUE);
}
```

```java
public int drainTo(Collection<? super E> c, int maxElements) {
    Objects.requireNonNull(c);
    if (c == this)
        throw new IllegalArgumentException();
    if (maxElements <= 0)
        return 0;
    boolean signalNotFull = false;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        int n = Math.min(maxElements, count.get());
        // count.get provides visibility to first n Nodes
        Node<E> h = head;
        int i = 0;
        try {
            while (i < n) {
                Node<E> p = h.next;
                c.add(p.item);
                p.item = null;
                h.next = h;
                h = p;
                ++i;
            }
            return n;
        } finally {
            // Restore invariants even if c.add() threw
            if (i > 0) {
                // assert h.item == null;
                head = h;
                signalNotFull = (count.getAndAdd(-i) == capacity);
            }
        }
    } finally {
        takeLock.unlock();
        if (signalNotFull)
            signalNotFull();
    }
}
```

在任何元素没有完全上锁的情况下遍历。此种遍历必须处理一下两种情况：
 - 出队节点(p.next == p)
 - (可能多个)内部节点移除(p.item == null)

```java
/**
 * Used for any element traversal that is not entirely under lock.
 * Such traversals must handle both:
 * - dequeued nodes (p.next == p)
 * - (possibly multiple) interior removed nodes (p.item == null)
 */
Node<E> succ(Node<E> p) {
    if (p == (p = p.next))
        p = head.next;
    return p;
}
```

## 十、Itr

返回队列当前元素顺序的迭代器，元素顺序按照队列头到队列尾

```java
public Iterator<E> iterator() {
    return new Itr();
}
```

弱一致性迭代器。懒更新祖先域提供预计O(1)的remove()，最差情况下为O(n)，不管何时保存的祖先节点已被并发删除。

```java
/**
 * Weakly-consistent iterator.
 *
 * Lazily updated ancestor field provides expected O(1) remove(),
 * but still O(n) in the worst case, whenever the saved ancestor
 * is concurrently deleted.
 */
private class Itr implements Iterator<E> {
    private Node<E> next;           // Node holding nextItem
    private E nextItem;             // next item to hand out
    private Node<E> lastRet;
    private Node<E> ancestor;       // Helps unlink lastRet on remove()

    Itr() {
        fullyLock();
        try {
            if ((next = head.next) != null)
                nextItem = next.item;
        } finally {
            fullyUnlock();
        }
    }

    public boolean hasNext() {
        return next != null;
    }

    public E next() {
        Node<E> p;
        if ((p = next) == null)
            throw new NoSuchElementException();
        lastRet = p;
        E x = nextItem;
        fullyLock();
        try {
            E e = null;
            for (p = p.next; p != null && (e = p.item) == null; )
                p = succ(p);
            next = p;
            nextItem = e;
        } finally {
            fullyUnlock();
        }
        return x;
    }

    public void forEachRemaining(Consumer<? super E> action) {
        // A variant of forEachFrom
        Objects.requireNonNull(action);
        Node<E> p;
        if ((p = next) == null) return;
        lastRet = p;
        next = null;
        final int batchSize = 64;
        Object[] es = null;
        int n, len = 1;
        do {
            fullyLock();
            try {
                if (es == null) {
                    p = p.next;
                    for (Node<E> q = p; q != null; q = succ(q))
                        if (q.item != null && ++len == batchSize)
                            break;
                    es = new Object[len];
                    es[0] = nextItem;
                    nextItem = null;
                    n = 1;
                } else
                    n = 0;
                for (; p != null && n < len; p = succ(p))
                    if ((es[n] = p.item) != null) {
                        lastRet = p;
                        n++;
                    }
            } finally {
                fullyUnlock();
            }
            for (int i = 0; i < n; i++) {
                @SuppressWarnings("unchecked") E e = (E) es[i];
                action.accept(e);
            }
        } while (n > 0 && p != null);
    }

    public void remove() {
        Node<E> p = lastRet;
        if (p == null)
            throw new IllegalStateException();
        lastRet = null;
        fullyLock();
        try {
            if (p.item != null) {
                if (ancestor == null)
                    ancestor = head;
                ancestor = findPred(p, ancestor);
                unlink(p, ancestor);
            }
        } finally {
            fullyUnlock();
        }
    }
}
```

##  十一、LBQSpliterator

Spliterators.IteratorSpliterator的自定义变体

```java
private final class LBQSpliterator implements Spliterator<E> {
    static final int MAX_BATCH = 1 << 25;  // max batch array size;
    Node<E> current;    // current node; null until initialized
    int batch;          // batch size for splits
    boolean exhausted;  // true when no more nodes
    long est = size();  // size estimate

    LBQSpliterator() {}

    public long estimateSize() { return est; }

    public Spliterator<E> trySplit() {
        Node<E> h;
        if (!exhausted &&
            ((h = current) != null || (h = head.next) != null)
            && h.next != null) {
            int n = batch = Math.min(batch + 1, MAX_BATCH);
            Object[] a = new Object[n];
            int i = 0;
            Node<E> p = current;
            fullyLock();
            try {
                if (p != null || (p = head.next) != null)
                    for (; p != null && i < n; p = succ(p))
                        if ((a[i] = p.item) != null)
                            i++;
            } finally {
                fullyUnlock();
            }
            if ((current = p) == null) {
                est = 0L;
                exhausted = true;
            }
            else if ((est -= i) < 0L)
                est = 0L;
            if (i > 0)
                return Spliterators.spliterator
                    (a, 0, i, (Spliterator.ORDERED |
                               Spliterator.NONNULL |
                               Spliterator.CONCURRENT));
        }
        return null;
    }

    public boolean tryAdvance(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        if (!exhausted) {
            E e = null;
            fullyLock();
            try {
                Node<E> p;
                if ((p = current) != null || (p = head.next) != null)
                    do {
                        e = p.item;
                        p = succ(p);
                    } while (e == null && p != null);
                if ((current = p) == null)
                    exhausted = true;
            } finally {
                fullyUnlock();
            }
            if (e != null) {
                action.accept(e);
                return true;
            }
        }
        return false;
    }

    public void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        if (!exhausted) {
            exhausted = true;
            Node<E> p = current;
            current = null;
            forEachFrom(action, p);
        }
    }

    public int characteristics() {
        return (Spliterator.ORDERED |
                Spliterator.NONNULL |
                Spliterator.CONCURRENT);
    }
}
```

返回基于此队列元素的可分割迭代器。返回的可分割迭代器是弱一致性的。此迭代器可报告Spliterator#CONCURRENT、Spliterator#ORDERED、Spliterator#NONNULL。

```java
public Spliterator<E> spliterator() {
    return new LBQSpliterator();
}
```

## 十二、前导节点

返回活节点p的前导节点，以便解链接p

```java
Node<E> findPred(Node<E> p, Node<E> ancestor) {
    // assert p.item != null;
    if (ancestor.item == null)
        ancestor = head;
    // Fails with NPE if precondition not satisfied
    for (Node<E> q; (q = ancestor.next) != p; )
        ancestor = q;
    return ancestor;
}
```

## 十三、批量操作


移除所有集合c的元素，若集合c对象为空则抛出NullPointerException
```java
public boolean removeAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return bulkRemove(e -> c.contains(e));
}
```

仅保留所有集合c的元素，若集合c对象为空则抛出NullPointerException

```java
public boolean retainAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return bulkRemove(e -> !c.contains(e));
}
```


实现批量移除方法

```java
@SuppressWarnings("unchecked")
private boolean bulkRemove(Predicate<? super E> filter) {
    boolean removed = false;
    Node<E> p = null, ancestor = head;
    Node<E>[] nodes = null;
    int n, len = 0;
    do {
        // 1. Extract batch of up to 64 elements while holding the lock.
        long deathRow = 0;          // "bitset" of size 64
        fullyLock();
        try {
            if (nodes == null) {
                if (p == null) p = head.next;
                for (Node<E> q = p; q != null; q = succ(q))
                    if (q.item != null && ++len == 64)
                        break;
                nodes = (Node<E>[]) new Node<?>[len];
            }
            for (n = 0; p != null && n < len; p = succ(p))
                nodes[n++] = p;
        } finally {
            fullyUnlock();
        }

        // 2. Run the filter on the elements while lock is free.
        for (int i = 0; i < n; i++) {
            final E e;
            if ((e = nodes[i].item) != null && filter.test(e))
                deathRow |= 1L << i;
        }

        // 3. Remove any filtered elements while holding the lock.
        if (deathRow != 0) {
            fullyLock();
            try {
                for (int i = 0; i < n; i++) {
                    final Node<E> q;
                    if ((deathRow & (1L << i)) != 0L
                        && (q = nodes[i]).item != null) {
                        ancestor = findPred(q, ancestor);
                        unlink(q, ancestor);
                        removed = true;
                    }
                }
            } finally {
                fullyUnlock();
            }
        }
    } while (n > 0 && p != null);
    return removed;
}
```