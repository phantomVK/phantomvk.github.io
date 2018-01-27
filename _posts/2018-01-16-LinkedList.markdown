---
layout:     post
title:      "Java源码系列(6) -- LinkedList"
date:       2018-01-16
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、介绍

Java常用的List实现有[ArrayList](http://phantomvk.coding.me/2017/02/19/Java_ArrayList/)和LinkedList。ArrayList通过数组实现，LinkedList通过链表实现。由于Java中没有指针的概念，所以通过一个对象保存下一对象引用的方式实现链表。

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

## 二、数据成员

LinkedList中保存元素的总数量，每次增删元素都会修改此值。

```java
transient int size = 0;
```

```java
// 第一个节点的指针
/**
 * Invariant: (first == null && last == null) ||
 *            (first.prev == null && first.item != null)
 */
transient Node<E> first;

// 最后一个节点的指针
/**
 * Invariant: (first == null && last == null) ||
 *            (last.next == null && last.item != null)
 */
transient Node<E> last;
```

## 三、构造方法

用指定集合构建列表，元素保存的顺序由集合的迭代器输出决定

```java
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c); // 若c为空抛空指针异常
}
```

## 四、成员方法

### 4.1 头插法、尾插法、选择插入

把元素作为第一个节点进入列表,，俗称头插法：

 - 插入完成前: 头指针 -> f -> ....
 - 插入完成后: 头指针 -> newNode -> f -> ....

```java
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```

把元素作为最后一个节点加入，俗称尾插法：

 - 插入完成前：... -> l <- 尾指针
 - 插入完成后：... -> l -> newNode <- 尾指针

```java
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

把元素添加到列表的首位

```java
public void addFirst(E e) {
    linkFirst(e);
}
```

把元素添加到列表的末尾

```java
public void addLast(E e) {
    linkLast(e);
}
```

在一个非空节点之前插入元素：

 - 插入完成前：pred -> succ -> ...
 - 插入完成后：pred -> newNode -> succ -> ...

```java
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

### 4.2 解除链接

解链接第一个非空的节点，私有方法

```java
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // 帮助GC
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
```

解链接最后一个非空节点，私有方法

```java
private E unlinkLast(Node<E> l) {
    // assert l == last && l != null;
    final E element = l.item;
    final Node<E> prev = l.prev;
    l.item = null;
    l.prev = null; // 帮助GC
    last = prev;
    if (prev == null)
        first = null;
    else
        prev.next = null;
    size--;
    modCount++;
    return element;
}
```

解连接非空节点

```java
E unlink(Node<E> x) {
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}
```

### 4.3 获取头元素和尾元素

获取列表的第一个元素

```java
public E getFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return f.item;
}
```

获取列表的最后一个元素

```java
public E getLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return l.item;
}
```

### 4.4 移除头元素和尾元素

如果仅有一个元素，那么头指针和尾指针指向的是同一个元素。头指针为空或尾指针为空，有且只有在列表为空时成立，并且是同时成立。

```java
// 移除并返回列表的第一个元素
public E removeFirst() {
    final Node<E> f = first;
    // 列表是空的，没有元素可以移除，抛出异常
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}

// 移除并返回列表的最后一个元素
public E removeLast() {
    final Node<E> l = last;
    // 列表是空的，没有元素可以移除，抛出异常
    if (l == null)
        throw new NoSuchElementException();
    return unlinkLast(l);
}
```

### 4.5 增删查改

```java
// 当列表中至少存在一个该对象时返回true，否则返回false
public boolean contains(Object o) {
    return indexOf(o) != -1;
}

// 返回列表元素的数量
public int size() {
    return size;
}

// 把指定元素附加到列表最后，和addLast()一样，尾插法
public boolean add(E e) {
    linkLast(e);
    return true;
}

// 从列表中移除第一个与对象o相同且存在的元素。如果列表不存在该元素，列表数据不会修改。
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}

// 把集合中所有元素通过尾插法插入到列表中，插入的顺序由集合的迭代器决定。
// 如果列表正在加入集合的元素，同时集合本身元素也在变化，那么导致的最终
// 结果是不可预知的。
public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);
}

// 把集合中所有元素通过尾插法插入到列表中，插入的顺序由集合的迭代器决定。
// 如果列表正在加入集合的元素，同时集合本身元素也在变化，那么导致的最终
// 结果是不可预知的。
public boolean addAll(int index, Collection<? extends E> c) {
    checkPositionIndex(index);

    Object[] a = c.toArray();
    int numNew = a.length;
    if (numNew == 0)
        return false;

    Node<E> pred, succ;
    if (index == size) {
        succ = null;
        pred = last;
    } else {
        succ = node(index);
        pred = succ.prev;
    }

    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        pred = newNode;
    }

    if (succ == null) {
        last = pred;
    } else {
        pred.next = succ;
        succ.prev = pred;
    }

    size += numNew;
    modCount++;
    return true;
}

// 移除列表中所有元素，方法执行结束后列表为空
public void clear() {
    for (Node<E> x = first; x != null; ) {
        Node<E> next = x.next;
        x.item = null;
        x.next = null;
        x.prev = null;
        x = next;
    }
    first = last = null;
    size = 0;
    modCount++;
}

```

## 五、指定下标操作

```java
// 返回指定下标的元素
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}

// 使用特定元素替换列表中指定元素
public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}

// 在列表的指定位置插入指定元素，并把原位置元素后移
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element);
    else
        linkBefore(element, node(index));
}

// 在列表中移除指定位置的元素，被移除的元素作为结果返回。
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}

// 检查参数index是否为一个存在元素的索引值
private boolean isElementIndex(int index) {
    return index >= 0 && index < size;
}

// 检查参数index是否是一个迭代器的有效索引值或是一个添加操作
private boolean isPositionIndex(int index) {
    return index >= 0 && index <= size;
}
```

```java
// 返回指定索引位置的非空节点
Node<E> node(int index) {
    // 检查索引值，如果小于列表元素数量的一半，就从头部顺序遍历
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else { // 否则就从尾部倒序遍历
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

```java
// 查找遇到的第一个与指定元素相同的节点的下标
// 列表不包含该元素则返回-1
public int indexOf(Object o) {
    int index = 0;
    // 如果o为空，就查找第一个遇到item为null的节点的下标
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null)
                return index;
            index++;
        }
    } else {
        // 否则会对标每个节点下的item是否与指定对象一致
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item))
                return index;
            index++;
        }
    }
    return -1;
}

// 查找遇到的最后一个与指定元素相同的节点的下标
// 列表不包含该元素则返回-1
public int lastIndexOf(Object o) {
    int index = size;
    if (o == null) {
        for (Node<E> x = last; x != null; x = x.prev) {
            index--;
            if (x.item == null)
                return index;
        }
    } else {
        // 否则会对标每个节点下的item是否与指定对象一致
        for (Node<E> x = last; x != null; x = x.prev) {
            index--;
            if (o.equals(x.item))
                return index;
        }
    }
    return -1;
}

// 获取头指针指向的节点的item，该操作不会移除节点
public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}

// 获取头指针指向的节点的item，该操作不会移除节点
public E element() {
    return getFirst();
}

// 获取头指针指向的节点的item，该操作会移除节点
public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}

// 取出并移除列表的头结点
public E remove() {
    return removeFirst();
}

// 把元素作为尾节点添加到列表中
public boolean offer(E e) {
    return add(e);
}

// 把元素作为列表头结点加入
public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}

// 把元素作为列表的尾节点加入
public boolean offerLast(E e) {
    addLast(e);
    return true;
}

// 获取但不移除列表的第一个节点元素，列表为空返回null
public E peekFirst() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
 }

// 获取但不移除列表的第一个节点元素，列表为空返回null
public E peekLast() {
    final Node<E> l = last;
    return (l == null) ? null : l.item;
}

// 获取并移除列表的第一个节点元素，列表为空返回null
public E pollFirst() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}

// 获取并移除列表的最后一个节点元素，列表为空返回null
public E pollLast() {
    final Node<E> l = last;
    return (l == null) ? null : unlinkLast(l);
}

// 把一个元素压到一个相当于栈的列表里面。换句话说，就是在列表首位插入元素
public void push(E e) {
    addFirst(e);
}

// 把一个元素从一个相当于栈的列表中弹出。换句话说，就是移除并返回列表的第一个元素
// 这个功能相当于removeFirst()
public E pop() {
    return removeFirst();
}

// 移除第一个包含指定实例的节点，成功返回true，否者返回false，且列表不做改变
// 此方法顺序遍历
public boolean removeFirstOccurrence(Object o) {
    return remove(o);
}

// 移除最后一个包含指定实例的节点，成功返回true，否者返回false，且列表不做改变
// 此方法倒序遍历
public boolean removeLastOccurrence(Object o) {
    if (o == null) {
        for (Node<E> x = last; x != null; x = x.prev) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = last; x != null; x = x.prev) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```

## 六、列表迭代器

```java
/**
 * Returns a list-iterator of the elements in this list (in proper
 * sequence), starting at the specified position in the list.
 * Obeys the general contract of {@code List.listIterator(int)}.<p>
 *
 * The list-iterator is <i>fail-fast</i>: if the list is structurally
 * modified at any time after the Iterator is created, in any way except
 * through the list-iterator's own {@code remove} or {@code add}
 * methods, the list-iterator will throw a
 * {@code ConcurrentModificationException}.  Thus, in the face of
 * concurrent modification, the iterator fails quickly and cleanly, rather
 * than risking arbitrary, non-deterministic behavior at an undetermined
 * time in the future.
 *
 * @param index index of the first element to be returned from the
 *              list-iterator (by a call to {@code next})
 * @return a ListIterator of the elements in this list (in proper
 *         sequence), starting at the specified position in the list
 * @throws IndexOutOfBoundsException {@inheritDoc}
 * @see List#listIterator(int)
 */
public ListIterator<E> listIterator(int index) {
    checkPositionIndex(index);
    return new ListItr(index);
}

private class ListItr implements ListIterator<E> {
    private Node<E> lastReturned;
    private Node<E> next;
    private int nextIndex;
    private int expectedModCount = modCount;

    ListItr(int index) {
        // assert isPositionIndex(index);
        next = (index == size) ? null : node(index);
        nextIndex = index;
    }

    public boolean hasNext() {
        return nextIndex < size;
    }

    public E next() {
        checkForComodification();
        if (!hasNext())
            throw new NoSuchElementException();

        lastReturned = next;
        next = next.next;
        nextIndex++;
        return lastReturned.item;
    }

    public boolean hasPrevious() {
        return nextIndex > 0;
    }

    public E previous() {
        checkForComodification();
        if (!hasPrevious())
            throw new NoSuchElementException();

        lastReturned = next = (next == null) ? last : next.prev;
        nextIndex--;
        return lastReturned.item;
    }

    public int nextIndex() {
        return nextIndex;
    }

    public int previousIndex() {
        return nextIndex - 1;
    }

    public void remove() {
        checkForComodification();
        if (lastReturned == null)
            throw new IllegalStateException();

        Node<E> lastNext = lastReturned.next;
        unlink(lastReturned);
        if (next == lastReturned)
            next = lastNext;
        else
            nextIndex--;
        lastReturned = null;
        expectedModCount++;
    }

    public void set(E e) {
        if (lastReturned == null)
            throw new IllegalStateException();
        checkForComodification();
        lastReturned.item = e;
    }

    public void add(E e) {
        checkForComodification();
        lastReturned = null;
        if (next == null)
            linkLast(e);
        else
            linkBefore(e, next);
        nextIndex++;
        expectedModCount++;
    }

    public void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (modCount == expectedModCount && nextIndex < size) {
            action.accept(next.item);
            lastReturned = next;
            next = next.next;
            nextIndex++;
        }
        checkForComodification();
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

## 七、链表节点模型

这是双向链表的模型，包含前一个节点的指针，下一个节点的指针，和该节点持有的实例。

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

## 八、倒序迭代器

```java
public Iterator<E> descendingIterator() {
    return new DescendingIterator();
}

/**
 * Adapter to provide descending iterators via ListItr.previous
 */
private class DescendingIterator implements Iterator<E> {
    private final ListItr itr = new ListItr(size());
    public boolean hasNext() {
        return itr.hasPrevious();
    }
    public E next() {
        return itr.previous();
    }
    public void remove() {
        itr.remove();
    }
}
```

## 九、克隆相关

```java
@SuppressWarnings("unchecked")
private LinkedList<E> superClone() {
    try {
        return (LinkedList<E>) super.clone();
    } catch (CloneNotSupportedException e) {
        throw new InternalError(e);
    }
}

// 返回列表的浅拷贝结果，列表内所有元素没有被克隆
public Object clone() {
    LinkedList<E> clone = superClone();

    // Put clone into "virgin" state
    clone.first = clone.last = null;
    clone.size = 0;
    clone.modCount = 0;

    // Initialize clone with our elements
    for (Node<E> x = first; x != null; x = x.next)
        clone.add(x.item);

    return clone;
}
```

## 十、列表转数组

返回一个包含所有元素的数组，元素顺序从链表第一位到最后一位。如果列表是空列表，会安全返回一个元素数量为0的有效数组对象。

由于返回的数组与原链表无关，所以对数组的修改不会影响原链表。

```java
public Object[] toArray() {
    // 首先构造一个与链表中有效元素数量一致的数组
    Object[] result = new Object[size];
    int i = 0;
    // 顺序遍历链表，把元素按照对应下表放入数组中
    for (Node<E> x = first; x != null; x = x.next)
        result[i++] = x.item;
    return result;
}
```

返回一个包含所有链表元素的数组，数组元素保存顺序与链表顺序一致。数组的返回类型由数组声明的类型为准，而不是链表节点保存类型为准。

如果数组空间足够保存所有链表元素，则正常返回传入数组，否则会创建一个与数组类型一致，容量与链表长度一致的新数组。

如果传入数组的容量大于链表的长度，则当最后一个链表节点存入数组位置
下一个数组空间会被置为null。这样有助于计算数组实际包含的元素数。

```java
@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        // 通过反射构造一个新的，有足够空间的新数组
        a = (T[])java.lang.reflect.Array.newInstance(
                            a.getClass().getComponentType(), size);
    int i = 0;
    Object[] result = a;
    for (Node<E> x = first; x != null; x = x.next)
        result[i++] = x.item;
    // 还有剩余的数组空间，则把第一个遇到的空闲空间置为null
    if (a.length > size)
        a[size] = null;

    return a;
}
```

