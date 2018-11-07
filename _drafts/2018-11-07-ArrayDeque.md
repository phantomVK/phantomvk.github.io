---
layout:     post
title:      "Java源码系列(20) -- ArrayDeque"
date:       2018-11-07
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Java源码系列
---

JDK11

## 类签名

__Deque__ 接口的可变大小的数组实现。数组双端队列没有容量限制，会在需要的时候扩容。本实现线程不安全，如果没有外界的同步约束，就不能支持多线程并发访问。本双端队列不接收为null的元素。此类作为栈使用时，看起来要比 __Stack__ 快；作为队列使用要比 __LinkedList__ 快。

大多数 __ArrayDeque__ 才做消耗常量级时间，除了__remove(Object)__，__removeFirstOccurrence__，__removeLastOccurrence__，__contains__，__iterator__ 和 批量操作是线性时间消耗。

```java
/**
 * Resizable-array implementation of the {@link Deque} interface.  Array
 * deques have no capacity restrictions; they grow as necessary to support
 * usage.  They are not thread-safe; in the absence of external
 * synchronization, they do not support concurrent access by multiple threads.
 * Null elements are prohibited.  This class is likely to be faster than
 * {@link Stack} when used as a stack, and faster than {@link LinkedList}
 * when used as a queue.
 *
 * <p>Most {@code ArrayDeque} operations run in amortized constant time.
 * Exceptions include
 * {@link #remove(Object) remove},
 * {@link #removeFirstOccurrence removeFirstOccurrence},
 * {@link #removeLastOccurrence removeLastOccurrence},
 * {@link #contains contains},
 * {@link #iterator iterator.remove()},
 * and the bulk operations, all of which run in linear time.
 *
 * <p>The iterators returned by this class's {@link #iterator() iterator}
 * method are <em>fail-fast</em>: If the deque is modified at any time after
 * the iterator is created, in any way except through the iterator's own
 * {@code remove} method, the iterator will generally throw a {@link
 * ConcurrentModificationException}.  Thus, in the face of concurrent
 * modification, the iterator fails quickly and cleanly, rather than risking
 * arbitrary, non-deterministic behavior at an undetermined time in the
 * future.
 *
 * <p>Note that the fail-fast behavior of an iterator cannot be guaranteed
 * as it is, generally speaking, impossible to make any hard guarantees in the
 * presence of unsynchronized concurrent modification.  Fail-fast iterators
 * throw {@code ConcurrentModificationException} on a best-effort basis.
 * Therefore, it would be wrong to write a program that depended on this
 * exception for its correctness: <i>the fail-fast behavior of iterators
 * should be used only to detect bugs.</i>
 *
 * <p>This class and its iterator implement all of the
 * <em>optional</em> methods of the {@link Collection} and {@link
 * Iterator} interfaces.
 */
public class ArrayDeque<E> extends AbstractCollection<E>
                           implements Deque<E>, Cloneable, Serializable
```

## 数据成员

```java
/*
 * VMs excel at optimizing simple array loops where indices are
 * incrementing or decrementing over a valid slice, e.g.
 *
 * for (int i = start; i < end; i++) ... elements[i]
 *
 * Because in a circular array, elements are in general stored in
 * two disjoint such slices, we help the VM by writing unusual
 * nested loops for all traversals over the elements.  Having only
 * one hot inner loop body instead of two or three eases human
 * maintenance and encourages VM loop inlining into the caller.
 */

/**
 * The array in which the elements of the deque are stored.
 * All array cells not holding deque elements are always null.
 * The array always has at least one null slot (at tail).
 */
transient Object[] elements;

/**
 * The index of the element at the head of the deque (which is the
 * element that would be removed by remove() or pop()); or an
 * arbitrary number 0 <= head < elements.length equal to tail if
 * the deque is empty.
 */
transient int head;

/**
 * The index at which the next element would be added to the tail
 * of the deque (via addLast(E), add(E), or push(E));
 * elements[tail] is always null.
 */
transient int tail;
```

## 常量

```java
/**
 * The maximum size of array to allocate.
 * Some VMs reserve some header words in an array.
 * Attempts to allocate larger arrays may result in
 * OutOfMemoryError: Requested array size exceeds VM limit
 */
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

```java
/**
 * Increases the capacity of this deque by at least the given amount.
 *
 * @param needed the required minimum extra capacity; must be positive
 */
private void grow(int needed) {
    // overflow-conscious code
    final int oldCapacity = elements.length;
    int newCapacity;
    // Double capacity if small; else grow by 50%
    int jump = (oldCapacity < 64) ? (oldCapacity + 2) : (oldCapacity >> 1);
    if (jump < needed
        || (newCapacity = (oldCapacity + jump)) - MAX_ARRAY_SIZE > 0)
        newCapacity = newCapacity(needed, jump);
    final Object[] es = elements = Arrays.copyOf(elements, newCapacity);
    // Exceptionally, here tail == head needs to be disambiguated
    if (tail < head || (tail == head && es[head] != null)) {
        // wrap around; slide first leg forward to end of array
        int newSpace = newCapacity - oldCapacity;
        System.arraycopy(es, head,
                         es, head + newSpace,
                         oldCapacity - head);
        for (int i = head, to = (head += newSpace); i < to; i++)
            es[i] = null;
    }
}

/** Capacity calculation for edge conditions, especially overflow. */
private int newCapacity(int needed, int jump) {
    final int oldCapacity = elements.length, minCapacity;
    if ((minCapacity = oldCapacity + needed) - MAX_ARRAY_SIZE > 0) {
        if (minCapacity < 0)
            throw new IllegalStateException("Sorry, deque too big");
        return Integer.MAX_VALUE;
    }
    if (needed > jump)
        return minCapacity;
    return (oldCapacity + jump - MAX_ARRAY_SIZE < 0)
        ? oldCapacity + jump
        : MAX_ARRAY_SIZE;
}
```

## 构造方法

构造默认队列，初始容量为16

```java
public ArrayDeque() {
    elements = new Object[16];
}
```

通过指定容量值构造双端数组队列

```java
public ArrayDeque(int numElements) {
    // 若numElements大于1小于MAX_VALUE，大小为numElements+1
    elements =
        new Object[(numElements < 1) ? 1 :
                   (numElements == Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                   numElements + 1];
}
```

通过指定集合构造双端数组队列，初始队列大小为执行集合元素数量值

```java
public ArrayDeque(Collection<? extends E> c) {
    this(c.size());
    copyElements(c);
}
```

## 静态方法

```java
/**
 * Circularly increments i, mod modulus.
 * Precondition and postcondition: 0 <= i < modulus.
 */
static final int inc(int i, int modulus) {
    if (++i >= modulus) i = 0;
    return i;
}

/**
 * Circularly decrements i, mod modulus.
 * Precondition and postcondition: 0 <= i < modulus.
 */
static final int dec(int i, int modulus) {
    if (--i < 0) i = modulus - 1;
    return i;
}

/**
 * Circularly adds the given distance to index i, mod modulus.
 * Precondition: 0 <= i < modulus, 0 <= distance <= modulus.
 * @return index 0 <= i < modulus
 */
static final int inc(int i, int distance, int modulus) {
    if ((i += distance) - modulus >= 0) i -= modulus;
    return i;
}

/**
 * Subtracts j from i, mod modulus.
 * Index i must be logically ahead of index j.
 * Precondition: 0 <= i < modulus, 0 <= j < modulus.
 * @return the "circular distance" from j to i; corner case i == j
 * is disambiguated to "empty", returning 0.
 */
static final int sub(int i, int j, int modulus) {
    if ((i -= j) < 0) i += modulus;
    return i;
}

// 返回数组中索引值为i的元素
@SuppressWarnings("unchecked")
static final <E> E elementAt(Object[] es, int i) {
    return (E) es[i];
}

/**
 * A version of elementAt that checks for null elements.
 * This check doesn't catch all possible comodifications,
 * but does catch ones that corrupt traversal.
 */
static final <E> E nonNullElementAt(Object[] es, int i) {
    @SuppressWarnings("unchecked") E e = (E) es[i];
    if (e == null)
        throw new ConcurrentModificationException();
    return e;
}
```

## 成员方法

```java
// The main insertion and extraction methods are addFirst,
// addLast, pollFirst, pollLast. The other methods are defined in
// terms of these.

// 把执行元素插入到队列头部，若元素为空抛出NullPointerException
public void addFirst(E e) {
    if (e == null)
        throw new NullPointerException();
    final Object[] es = elements;
    es[head = dec(head, es.length)] = e;
    if (head == tail)
        grow(1);
}

// 把执行元素插入到队列尾部，若元素为空抛出NullPointerException
public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    final Object[] es = elements;
    es[tail] = e;
    if (head == (tail = inc(tail, es.length)))
        grow(1);
}

// 把指定集合的所有元素添加到队列的尾部
public boolean addAll(Collection<? extends E> c) {
    final int s, needed;
    if ((needed = (s = size()) + c.size() + 1 - elements.length) > 0)
        grow(needed);
    copyElements(c);
    return size() > s;
}
```

```java
private void copyElements(Collection<? extends E> c) {
    c.forEach(this::addLast);
}

// 把指定元素插入到队列头部
public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}

// 把指定元素插入到队列头部
public boolean offerLast(E e) {
    addLast(e);
    return true;
}
```

```java
// 若找不到头元素就抛出NoSuchElementException
public E removeFirst() {
    E e = pollFirst();
    if (e == null)
        throw new NoSuchElementException();
    return e;
}

// 若找不到最后一个元素就抛出NoSuchElementException
public E removeLast() {
    E e = pollLast();
    if (e == null)
        throw new NoSuchElementException();
    return e;
}
```

```java
public E pollFirst() {
    final Object[] es;
    final int h;
    E e = elementAt(es = elements, h = head);
    if (e != null) {
        es[h] = null;
        head = inc(h, es.length);
    }
    return e;
}

public E pollLast() {
    final Object[] es;
    final int t;
    E e = elementAt(es = elements, t = dec(tail, es.length));
    if (e != null)
        es[tail = t] = null;
    return e;
}
```

```java
// 找不到元素抛出NoSuchElementException
public E getFirst() {
    E e = elementAt(elements, head);
    if (e == null)
        throw new NoSuchElementException();
    return e;
}

// 找不到元素抛出NoSuchElementException
public E getLast() {
    final Object[] es = elements;
    E e = elementAt(es, dec(tail, es.length));
    if (e == null)
        throw new NoSuchElementException();
    return e;
}
```

```java
public E peekFirst() {
    return elementAt(elements, head);
}

public E peekLast() {
    final Object[] es;
    return elementAt(es = elements, dec(tail, es.length));
}
```

移出第一个命中的指定元素。如果队列存在多个相同元素，每次调用方法仅移除一个元素。每次查找从的头部开始，逐个遍历元素寻找匹配项。

```java
/**
 * Removes the first occurrence of the specified element in this
 * deque (when traversing the deque from head to tail).
 * If the deque does not contain the element, it is unchanged.
 * More formally, removes the first element {@code e} such that
 * {@code o.equals(e)} (if such an element exists).
 * Returns {@code true} if this deque contained the specified element
 * (or equivalently, if this deque changed as a result of the call).
 *
 * @param o element to be removed from this deque, if present
 * @return {@code true} if the deque contained the specified element
 */
public boolean removeFirstOccurrence(Object o) {
    if (o != null) {
        final Object[] es = elements;
        for (int i = head, end = tail, to = (i <= end) ? end : es.length;
             ; i = 0, to = end) {
            for (; i < to; i++)
                if (o.equals(es[i])) {
                    delete(i);
                    return true;
                }
            if (to == end) break;
        }
    }
    return false;
}
```

移出最后一个命中的指定元素。如果队列存在多个相同元素，每次调用方法仅移除一个元素。

```java
/**
 * Removes the last occurrence of the specified element in this
 * deque (when traversing the deque from head to tail).
 * If the deque does not contain the element, it is unchanged.
 * More formally, removes the last element {@code e} such that
 * {@code o.equals(e)} (if such an element exists).
 * Returns {@code true} if this deque contained the specified element
 * (or equivalently, if this deque changed as a result of the call).
 *
 * @param o element to be removed from this deque, if present
 * @return {@code true} if the deque contained the specified element
 */
public boolean removeLastOccurrence(Object o) {
    if (o != null) {
        final Object[] es = elements;
        for (int i = tail, end = head, to = (i >= end) ? end : 0;
             ; i = es.length, to = end) {
            for (i--; i > to - 1; i--)
                if (o.equals(es[i])) {
                    delete(i);
                    return true;
                }
            if (to == end) break;
        }
    }
    return false;
}
```

#### 队列方法

```java
// 把指定元素插入到队列尾部
public boolean add(E e) {
    addLast(e);
    return true;
}

// 把指定元素插入到队列尾部
public boolean offer(E e) {
    return offerLast(e);
}

// 获取并移除队列头元素，若队列没有元素则抛出NoSuchElementException
public E remove() {
    return removeFirst();
}

// 获取并移除队列头元素，如果元素不存在返回null
public E poll() {
    return pollFirst();
}

// 仅获取元素队列头元素，但不从队列中不会移除。如果队列为空，则此方法会抛出异常
public E element() {
    return getFirst();
}

// 仅获取元素队列头元素，但不从队列中不会移除。如果队列为空，则此方法返回null
public E peek() {
    return peekFirst();
}
```

#### 栈方法

```java
// 向栈中压入元素，即向本队列头部插入元素。若指定元素为空抛出NullPointerException
public void push(E e) {
    addFirst(e);
}

// 从栈中弹出元素，即从本队列头部移除并返回元素。若队列为空抛出NoSuchElementException
public E pop() {
    return removeFirst();
}

/**
 * Removes the element at the specified position in the elements array.
 * This can result in forward or backwards motion of array elements.
 * We optimize for least element motion.
 *
 * <p>This method is called delete rather than remove to emphasize
 * that its semantics differ from those of {@link List#remove(int)}.
 *
 * @return true if elements near tail moved backwards
 */
// 从元素数组中移除指定索引值的元素。
boolean delete(int i) {
    final Object[] es = elements;
    final int capacity = es.length;
    final int h, t;
    // number of elements before to-be-deleted elt
    final int front = sub(i, h = head, capacity);
    // number of elements after to-be-deleted elt
    final int back = sub(t = tail, i, capacity) - 1;
    if (front < back) {
        // move front elements forwards
        if (h <= i) {
            System.arraycopy(es, h, es, h + 1, front);
        } else { // Wrap around
            System.arraycopy(es, 0, es, 1, i);
            es[0] = es[capacity - 1];
            System.arraycopy(es, h, es, h + 1, front - (i + 1));
        }
        es[h] = null;
        head = inc(h, capacity);
        return false;
    } else {
        // move back elements backwards
        tail = dec(t, capacity);
        if (i <= tail) {
            System.arraycopy(es, i + 1, es, i, back);
        } else { // Wrap around
            System.arraycopy(es, i + 1, es, i, capacity - (i + 1));
            es[capacity - 1] = es[0];
            System.arraycopy(es, 1, es, 0, t - 1);
        }
        es[tail] = null;
        return true;
    }
}
```

#### 集合方法

```java
// 返回双端队列包含元素的数量
public int size() {
    return sub(tail, head, elements.length);
}

// 若双端队列不含任何元素返回true
public boolean isEmpty() {
    return head == tail;
}
```

```java
// A tiny bit set implementation

private static long[] nBits(int n) {
    return new long[((n - 1) >> 6) + 1];
}
private static void setBit(long[] bits, int i) {
    bits[i >> 6] |= 1L << i;
}
private static boolean isClear(long[] bits, int i) {
    return (bits[i >> 6] & (1L << i)) == 0;
}

/**
 * Returns {@code true} if this deque contains the specified element.
 * More formally, returns {@code true} if and only if this deque contains
 * at least one element {@code e} such that {@code o.equals(e)}.
 *
 * @param o object to be checked for containment in this deque
 * @return {@code true} if this deque contains the specified element
 */
// 如果队列包含指定元素，则返回true。一般来说，队列可能存在多个相同的元素
// 所以本方法返回true是表示队列至少存在一个与指定元素相等的元素
public boolean contains(Object o) {
    if (o != null) {
        final Object[] es = elements;
        for (int i = head, end = tail, to = (i <= end) ? end : es.length;
             ; i = 0, to = end) {
            for (; i < to; i++)
                if (o.equals(es[i]))
                    return true;
            if (to == end) break;
        }
    }
    return false;
}

/**
 * Removes a single instance of the specified element from this deque.
 * If the deque does not contain the element, it is unchanged.
 * More formally, removes the first element {@code e} such that
 * {@code o.equals(e)} (if such an element exists).
 * Returns {@code true} if this deque contained the specified element
 * (or equivalently, if this deque changed as a result of the call).
 *
 * <p>This method is equivalent to {@link #removeFirstOccurrence(Object)}.
 *
 * @param o element to be removed from this deque, if present
 * @return {@code true} if this deque contained the specified element
 */
// 从队列中移除指定元素。如果队列不含该元素，则队列不会改变。
// 一般来说，队列可能会含有多和相同的元素
public boolean remove(Object o) {
    return removeFirstOccurrence(o);
}

// 从移除队列中所有元素
public void clear() {
    circularClear(elements, head, tail);
    head = tail = 0;
}

/**
 * Nulls out slots starting at array index i, upto index end.
 * Condition i == end means "empty" - nothing to do.
 */
private static void circularClear(Object[] es, int i, int end) {
    // assert 0 <= i && i < es.length;
    // assert 0 <= end && end < es.length;
    for (int to = (i <= end) ? end : es.length;
         ; i = 0, to = end) {
        for (; i < to; i++) es[i] = null;
        if (to == end) break;
    }
}
```

#### toArray

返回包含双端队列所有元素的数组，元素顺序和双端队列元素顺序一致。

```java
/**
 * Returns an array containing all of the elements in this deque
 * in proper sequence (from first to last element).
 *
 * <p>The returned array will be "safe" in that no references to it are
 * maintained by this deque.  (In other words, this method must allocate
 * a new array).  The caller is thus free to modify the returned array.
 *
 * <p>This method acts as bridge between array-based and collection-based
 * APIs.
 *
 * @return an array containing all of the elements in this deque
 */
public Object[] toArray() {
    return toArray(Object[].class);
}

private <T> T[] toArray(Class<T[]> klazz) {
    final Object[] es = elements;
    final T[] a;
    final int head = this.head, tail = this.tail, end;
    if ((end = tail + ((head <= tail) ? 0 : es.length)) >= 0) {
        // Uses null extension feature of copyOfRange
        a = Arrays.copyOfRange(es, head, end, klazz);
    } else {
        // integer overflow!
        a = Arrays.copyOfRange(es, 0, end - head, klazz);
        System.arraycopy(es, head, a, 0, es.length - head);
    }
    if (end != tail)
        System.arraycopy(es, 0, a, es.length - head, tail);
    return a;
}
```

传入目标数组，并把双端队列所有元素放入该数组，元素顺序和双端队列元素顺序一致。返回数组的类型和传入数组类型相同。如果传入数组大小不足容纳所有元素，方法会创建新数组，且新容量和所放入元素数量一致，然后放入所有元素并返回。所以，会出现传入数组和返回数组不是同一个对象的现象。

如果传入数组空间足够存入所有元素，该数组的下一个空间会被置为 __null__。

本方法可以实现队列转数组的功能：__String[] y = x.toArray(new String[0]);__。且值得注意的是，传入 __toArray(new Object[0])__ 和 传入 __toArray()__ 的效果完全相同。

指定数组元素的运行时类型不是双端队列元素的运行时类型的子类时抛出 __ArrayStoreException__；
指定数组为空抛出 __NullPointerException__；

```java
/**
 * Returns an array containing all of the elements in this deque in
 * proper sequence (from first to last element); the runtime type of the
 * returned array is that of the specified array.  If the deque fits in
 * the specified array, it is returned therein.  Otherwise, a new array
 * is allocated with the runtime type of the specified array and the
 * size of this deque.
 *
 * <p>If this deque fits in the specified array with room to spare
 * (i.e., the array has more elements than this deque), the element in
 * the array immediately following the end of the deque is set to
 * {@code null}.
 *
 * <p>Like the {@link #toArray()} method, this method acts as bridge between
 * array-based and collection-based APIs.  Further, this method allows
 * precise control over the runtime type of the output array, and may,
 * under certain circumstances, be used to save allocation costs.
 *
 * <p>Suppose {@code x} is a deque known to contain only strings.
 * The following code can be used to dump the deque into a newly
 * allocated array of {@code String}:
 *
 * <pre> {@code String[] y = x.toArray(new String[0]);}</pre>
 *
 * Note that {@code toArray(new Object[0])} is identical in function to
 * {@code toArray()}.
 *
 * @param a the array into which the elements of the deque are to
 *          be stored, if it is big enough; otherwise, a new array of the
 *          same runtime type is allocated for this purpose
 * @return an array containing all of the elements in this deque
 * @throws ArrayStoreException if the runtime type of the specified array
 *         is not a supertype of the runtime type of every element in
 *         this deque
 * @throws NullPointerException if the specified array is null
 */
@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a) {
    final int size;
    if ((size = size()) > a.length)
        return toArray((Class<T[]>) a.getClass());
    final Object[] es = elements;
    for (int i = head, j = 0, len = Math.min(size, es.length - i);
         ; i = 0, len = tail) {
        System.arraycopy(es, i, a, j, len);
        if ((j += len) == size) break;
    }
    if (size < a.length)
        a[size] = null;
    return a;
}
```

#### checkInvariants

```java
void checkInvariants() {
    // Use head and tail fields with empty slot at tail strategy.
    // head == tail disambiguates to "empty".
    try {
        int capacity = elements.length;
        // assert 0 <= head && head < capacity;
        // assert 0 <= tail && tail < capacity;
        // assert capacity > 0;
        // assert size() < capacity;
        // assert head == tail || elements[head] != null;
        // assert elements[tail] == null;
        // assert head == tail || elements[dec(tail, capacity)] != null;
    } catch (Throwable t) {
        System.err.printf("head=%d tail=%d capacity=%d%n",
                          head, tail, elements.length);
        System.err.printf("elements=%s%n",
                          Arrays.toString(elements));
        throw t;
    }
}
```