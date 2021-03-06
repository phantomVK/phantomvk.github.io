---
layout:     post
title:      "Java源码系列(23) -- ArrayDeque"
date:       2019-05-14
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、类签名

__ArrayDeque__ 是实现 __Deque__ 接口且容量可变的双端队列数组。数组实现的双端队列没有容量限制，需要更多空间时再进行扩容。

此类线程不安全，如果没有外部同步约束，就不支持多线程并发。值得注意的是，本双端队列不接受空对象，作为栈使用时比 __Stack__ 快，作为队列使用时比 __LinkedList__ 快。

大多数 __ArrayDeque__ 方法执行消耗常量时间，除了 __remove(Object)__、 __removeFirstOccurrence__，__removeLastOccurrence__、__contains__、__iterator__ 和批量操作是线性时间消耗的。

```java
public class ArrayDeque<E> extends AbstractCollection<E>
                           implements Deque<E>, Cloneable, Serializable
```

其次，虚拟机擅长基于有效切片中索引的递增、递减操作，对简单数组循环进行优化。例如：

```java
for (int i = start; i < end; i++) ... elements[i]
```

源码来自 JDK11

## 二、数据成员

保存双端数组队列变量。当数组的 __cells__ 没有持有双端队列元素时为空。数组存在至少一个空位，作为队列的尾部

```java
transient Object[] elements;
```

头元素在数组中的索引值，下标值对应元素由remove()或pop()方法移除。若队列没有元素，head为 __[0, elements.length)__ 间任意值，与尾引用值相同

```java
transient int head;
```

下一个元素存入数组尾部的索引值，所以 __elements[tail]__ 一直为空

```java
transient int tail;
```

## 三、常量

可申请数组的最大容量值。有些虚拟机实现会在数组中保留 __header words__。所以尝试分配更大数组空间会导致 __OutOfMemoryError__。使用此值避免了这种问题。

```java
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

## 四、扩容方法

增加至少 __needed__ 个数组空间，必须为正数。方法计算新容量值时，已经完成整形值向上溢出的处理

```java
private void grow(int needed) {
    // 获取原数组容量值
    final int oldCapacity = elements.length;
    
    // 可 newCapacity = oldCapacity + jump 
    // 或 newCapacity = oldCapacity + needed
    // 或 newCapacity = MAX_ARRAY_SIZE
    // 或 newCapacity = Integer.MAX_VALUE
    int newCapacity;

    // 若原容量值小于64，jump为原值加2，否则jump为原值一半
    int jump = (oldCapacity < 64) ? (oldCapacity + 2) : (oldCapacity >> 1);

    // 计算jump是否比理想扩容值needed小
    if (jump < needed
        || (newCapacity = (oldCapacity + jump)) - MAX_ARRAY_SIZE > 0)
        newCapacity = newCapacity(needed, jump);

    // 根据newCapacity创建新数组，并把原数组元素拷贝到新数组
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
```

为边缘条件进行容量计算，尤其是向上溢出的情况。

```java
private int newCapacity(int needed, int jump) {
    final int oldCapacity = elements.length, minCapacity;

    if ((minCapacity = oldCapacity + needed) - MAX_ARRAY_SIZE > 0) {
        // 最大容量值溢出
        if (minCapacity < 0)
            throw new IllegalStateException("Sorry, deque too big");
        
        // 设置为最大值
        return Integer.MAX_VALUE;
    }
    
    if (needed > jump)
        return minCapacity;

    // needed <= jump
    return (oldCapacity + jump - MAX_ARRAY_SIZE < 0)
        ? oldCapacity + jump
        : MAX_ARRAY_SIZE;
}
```

## 五、构造方法

构造默认队列，初始容量为16

```java
public ArrayDeque() {
    elements = new Object[16];
}
```

通过指定容量值构造双端数组队列

```java
public ArrayDeque(int numElements) {
    // 若numElements大于1小于MAX_VALUE，大小为numElements+1，多出来的一个空间给尾指针
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

## 六、静态方法

```java
// 循环递增i，实现对i取模的能力。先决条件和事后条件为：0 <= i < modulus
static final int inc(int i, int modulus) {
    if (++i >= modulus) i = 0;
    return i;
}

// 循环递减i，实现对i取模的能力。先决条件和事后条件为：0 <= i < modulus
static final int dec(int i, int modulus) {
    if (--i < 0) i = modulus - 1;
    return i;
}

// 循环增加指定距离值到i，实现对i取模的能力
// 先决条件: 0 <= i < modulus, 0 <= distance <= modulus
// 返回值：index 0 <= i < modulus
static final int inc(int i, int distance, int modulus) {
    if ((i += distance) - modulus >= 0) i -= modulus;
    return i;
}

// 从i减去j，并对i取模的能力
// 索引i必须在逻辑上在索引j之前
// 先决条件: 0 <= i < modulus, 0 <= j < modulus； 
// 返回值：j到i之间的环形距离；
static final int sub(int i, int j, int modulus) {
    if ((i -= j) < 0) i += modulus;
    return i;
}

// 返回数组中索引值为i的元素
@SuppressWarnings("unchecked")
static final <E> E elementAt(Object[] es, int i) {
    return (E) es[i];
}
```

## 七、成员方法

元素主要的插入、获取方法是 __addFirst__、__addLast__、 __pollFirst__、 __pollLast__，其他方法都在此基础上实现

#### 7.1 add

把执行元素插入到队列头部，若元素为空抛出NullPointerException

```java
public void addFirst(E e) {
    if (e == null)
        throw new NullPointerException();
    final Object[] es = elements;
    es[head = dec(head, es.length)] = e;
    if (head == tail)
        grow(1);
}
```

把执行元素插入到队列尾部，若元素为空抛出NullPointerException

```java
public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    final Object[] es = elements;
    es[tail] = e;
    if (head == (tail = inc(tail, es.length)))
        grow(1);
}
```

把指定集合的所有元素添加到队列的尾部

```java
public boolean addAll(Collection<? extends E> c) {
    final int s, needed;
    // 需要扩容大小 = 已用空间 + 新元素数量 + 1 - 数组长度
    if ((needed = (s = size()) + c.size() + 1 - elements.length) > 0)
        grow(needed);
    copyElements(c);
    return size() > s;
}
```

#### 7.2 copyElements

把集合C的元素添加到本队列尾部

```java
private void copyElements(Collection<? extends E> c) {
    c.forEach(this::addLast);
}
```

#### 7.3 offer

把指定元素插入到队列头部

```java
public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}
```

把指定元素插入到队列头部

```java
public boolean offerLast(E e) {
    addLast(e);
    return true;
}
```

#### 7.4 remove

若找不到头元素就抛出NoSuchElementException

```java
public E removeFirst() {
    E e = pollFirst();
    if (e == null)
        throw new NoSuchElementException();
    return e;
}
```

若找不到最后一个元素就抛出NoSuchElementException

```java
public E removeLast() {
    E e = pollLast();
    if (e == null)
        throw new NoSuchElementException();
    return e;
}
```

#### 7.5 poll

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

#### 7.6 get

找不到元素抛出NoSuchElementException

```java
public E getFirst() {
    // 通过head索引值获取元素
    E e = elementAt(elements, head);
    if (e == null)
        throw new NoSuchElementException();
    return e;
}
```

找不到元素抛出NoSuchElementException

```java
public E getLast() {
    final Object[] es = elements;
    E e = elementAt(es, dec(tail, es.length));
    if (e == null)
        throw new NoSuchElementException();
    return e;
}
```

#### 7.7 peek

```java
public E peekFirst() {
    return elementAt(elements, head);
}

public E peekLast() {
    final Object[] es;
    return elementAt(es = elements, dec(tail, es.length));
}
```

#### 7.8 firstOccurrence

移出第一个命中的指定元素。如果队列存在多个相同元素，每次调用方法仅移除一个。

每次查找从头部开始，逐个遍历元素寻找匹配项。元素命中并移除成功返回 __true__，元素为null或不包含该元素返回 __false__。

```java
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

移出最后一个命中的指定元素。如果队列存在多个相同元素，每次调用方法仅移除一个。元素命中并移除成功返回 __true__，元素为null或不包含该元素返回 __false__。

```java
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

#### 7.9 队列方法

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

// 仅获取元素队列头元素，但不从队列中移除。如果队列为空，此方法会抛出异常
public E element() {
    return getFirst();
}

// 仅获取元素队列头元素，但不从队列中移除。如果队列为空，此方法返回null
public E peek() {
    return peekFirst();
}
```

#### 7.10 栈方法

```java
// 向栈中压入元素，即向本队列头部插入元素。若指定元素为空抛出NullPointerException
public void push(E e) {
    addFirst(e);
}

// 从栈中弹出元素，即从本队列头部移除并返回元素。若队列为空抛出NoSuchElementException
public E pop() {
    return removeFirst();
}

// 从元素数组中移除指定索引的元素。
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

#### 7.11 集合方法

返回双端队列包含元素的数量

```java
public int size() {
    return sub(tail, head, elements.length);
}
```

若双端队列不含任何元素返回true。在头引用和尾引用指向同一个对象的时候能表示双端队列为空。

```java
public boolean isEmpty() {
    return head == tail;
}
```

#### 7.12 位操作

```java
private static long[] nBits(int n) {
    return new long[((n - 1) >> 6) + 1];
}

private static void setBit(long[] bits, int i) {
    bits[i >> 6] |= 1L << i;
}

private static boolean isClear(long[] bits, int i) {
    return (bits[i >> 6] & (1L << i)) == 0;
}
```

#### 7.13 contains

如果队列包含指定元素返回true。一般来说，队列可能存在多个相同的元素。所以本方法返回true表示队列至少存在一个元素与指定元素相等。

```java
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
```

#### 7.14 remove

从队列中移除指定单个元素。如果队列不含该元素，则队列不会改变。一般来说，队列可能会含有多和相同的元素，每次仅移除其中一个。

```java
public boolean remove(Object o) {
    return removeFirstOccurrence(o);
}
```

#### 7.14 clear

从移除队列中所有元素

```java
public void clear() {
    circularClear(elements, head, tail);
    head = tail = 0;
}
```

调用以下方法，空间逐个置空

```java
// Nulls out slots starting at array index i, upto index end.
// Condition i == end means "empty" - nothing to do.
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

#### 7.15 toArray

返回包含双端队列所有元素的数组，元素顺序和双端队列元素顺序一致。

```java
public Object[] toArray() {
    return toArray(Object[].class);
}

private <T> T[] toArray(Class<T[]> klazz) {
    // 获取元素数组
    final Object[] es = elements;
    final T[] a;
    // 分别获取头引用和未引用
    final int head = this.head, tail = this.tail, end;
    if ((end = tail + ((head <= tail) ? 0 : es.length)) >= 0) {
        // Uses null extension feature of copyOfRange
        a = Arrays.copyOfRange(es, head, end, klazz);
    } else {
        // 整形上溢
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

数组元素的运行时类型不匹配双端队列元素的运行时类型时，抛出 __ArrayStoreException__；
数组为空抛出 __NullPointerException__；

```java
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
