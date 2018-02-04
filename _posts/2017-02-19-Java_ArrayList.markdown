---
layout:     post
title:      "Java源码系列(1) -- ArrayList"
date:       2017-02-19
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---



## 一、类签名

源码版本为JDK8，支持`随机存储`、`克隆`、`序列化`

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

## 二、数据成员

```java
private static final long serialVersionUID = 8683452581122892189L;
private static final int DEFAULT_CAPACITY = 10; // 缺省值
```

构造函数方法参数为0的数组用这个空数组标识

```java
private static final Object[] EMPTY_ELEMENTDATA = {};
```

无参构造方法使用下面的空数组作为标识，以便第一次初始化数组的时候系统知道需要用缺省值作为数组长度

```java
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
```

ArrayList的总长度就是这个总长度，如果构建前等于`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`，那么第一个元素加入时构建序列长度为10

```java
transient Object[] elementData;
```

ArrayList的大小，指已加入元素的数量

```java
private int size;
```

## 三、构造方法

### 3.1 默认构造

无参构造方法默认构造大小是10，初始化数组延迟到加入第一个元素时才进行

```java
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

### 3.2 指定构造

构造时指定ArrayList的容量，则空间会立即创建。如果列表的长度较短且是可预知的，使用此构造方法能避免动态扩展而造成多余的空间浪费。

```java 
public ArrayList(int initialCapacity) {
    // 使用指定容量初始化会立即创建数组
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
    }
}
```

### 3.3 集合构造

通过一个集合构建ArrayList，顺序由集合迭代器依次指定顺序为准。创建的ArrayList长度和Collection的长度一致。

```java
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        this.elementData = EMPTY_ELEMENTDATA; // 空集合构建为空数组列表
    }
}
```

## 四、方法

### 4.1 裁剪

 把列表长度裁剪到实际占用长度，用于释放未占用的数组空间。如果数组保存元素为0就设为空数组，否则缩短数组长度到仅占用的长度。

```java
public void trimToSize() {
    modCount++;
    if (size < elementData.length) {
        elementData = (size == 0)
          ? EMPTY_ELEMENTDATA
          : Arrays.copyOf(elementData, size);
    }
}
```

### 4.2 增长

#### 4.2.1 ensureCapacity()

增加大小有两种类别：

* 使用默认构造方法构造指向`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`，长度是0。调用下列方法时`minCapacity`只有大于10才会执行数组扩增，把数组从0增到`minCapacity`。

* 如果数组长度不为0而假设为A，则`minCapacity > A`才有效。`minCapacity`的意思就是数组的最短长度，而不是增加长度的数值。总不能`minCapacity`比原数组还小吧。

```java
public void ensureCapacity(int minCapacity) {
    int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) ? 
            0 : DEFAULT_CAPACITY;

    if (minCapacity > minExpand) {
        ensureExplicitCapacity(minCapacity);
    }
}
```
#### 4.2.2 ensureExplicitCapacity()

列表为空，`elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA` -> `grow(10)`;

列表为空，`elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA` -> `return`;

```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // minCapacity必须比数组长度大才扩展空间
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

#### 4.2.3 ensureCapacityInternal()

这里控制默认构造空数组扩展最小值为10

```java
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        // 取DEFAULT_CAPACITY=10和minCapacity两者中的最大值
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}
```

#### 4.2.4 MAX_ARRAY_SIZE

数组最大申请空间，有的虚拟机实现会把对象头信息保存在数组中，尝试分配更大内存空间在这种情况下会造成OOM：请求数字大小超过VM的限制

```java
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

#### 4.2.5 grow()

就算`minCapacity`比数组长度大，也不一定会采用`minCapacity`的值。因为每次数组扩增不是在原数组上扩展，而是创建新的数组并拷贝旧数组的内容到新数组上。

若每次扩增只增加1个长度，那么每次扩容后废弃的旧数组内存空间将对GC造成极大的压力，尤其在连续添加新元素的场景下。

假设旧数组长度是16，根据`newCapacity = oldCapacity + (oldCapacity >> 1)`，`newCapacity`为16+8=24。如果自定义`minCapacity`小于24，则方法按照24的长度扩增。

```java
private void grow(int minCapacity) {
    // 有溢出检查
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    
    // 取计算值与minCapacity大者
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
        
    // 上溢检查
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // 选择minCapacity是因为其更接近Size
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

先下溢检查，然后进行上溢检查

```java
private static int hugeCapacity(int minCapacity) {
    // 下溢检查
    if (minCapacity < 0)
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

### 4.3 元素数量

返回保存元素数量

```java
public int size() {
    return size;
}
```

### 4.4 是否有元素
元素数量是否为0

```java
public boolean isEmpty() {
    return size == 0;
}
```

### 4.4 是否包含

找指定对象是否保存在列表中

```java
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}
```

### 4.5 元素查找

查找指定元素的序号。若元素是空对象，则找数组遇到第一个null的下标。其他情况，找到元素返回下标，找不到返回`-1`  

```java
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

查指定元素在列表中最后一次出现的索引值

```java
public int lastIndexOf(Object o) {
    if (o == null) {
        for (int i = size-1; i >= 0; i--)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = size-1; i >= 0; i--)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

### 4.6 浅克隆

正常来说不会出现`CloneNotSupportedException`，因为本身实现了`Cloneable`接口

```java
public Object clone() {
    try {
        ArrayList<?> v = (ArrayList<?>) super.clone();
        v.elementData = Arrays.copyOf(elementData, size);
        v.modCount = 0;
        return v;
    } catch (CloneNotSupportedException e) {
        throw new InternalError(e); 
    }
}
```

### 4.7 返回数组

按照列表原顺序返回一个新数组，新数组和原ArrayList互相独立

```java
public Object[] toArray() {
    return Arrays.copyOf(elementData, size);
}
```

用自行传入的数组保存列表的元素，类型与传入相同。传入数组多余空位置为null，否则会创建一个新数组。因此，当传入的数组长度不足，那么返回的数组和传入的数组不是同一个对象。

```java
@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        // Make a new array of a's runtime type, but my contents:
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```

### 4.8 返回指定下标元素

返回指定位置的元素，无下标检查

```java
@SuppressWarnings("unchecked")
E elementData(int index) {
    return (E) elementData[index];
}
```

返回指定位置的元素，带下标检查

```java
public E get(int index) {
    rangeCheck(index);

    return elementData(index);
}
```

用新的对象替换指定位置的对象，并返回原位置旧对象

```java
public E set(int index, E element) {
    rangeCheck(index);

    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
}
```

### 4.9 加入

增加元素

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // 增加的修改操作
    elementData[size++] = e;
    return true;
}
```

在指定位置增加新的元素，原位置以及其后元素组成的子序列向后移动

```java
public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;
    size++;
}
```

把集合的保存的元素追加在ArrayList的尾部

```java
public boolean addAll(Collection<? extends E> c) {
    // 把Collection变换为数组类型
    Object[] a = c.toArray();
    // 统计a的元素数量，用于计算需要扩展容量的大小
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);
    // 把集合的元素逐一拷贝到列表的尾部
    System.arraycopy(a, 0, elementData, size, numNew);
    // 更新列表的元素数量size
    size += numNew;
    return numNew != 0;
}
```

在指定位置插入若干个保存在集合的元素，ArrayList原位置元素后移

```java
public boolean addAll(int index, Collection<? extends E> c) {
    rangeCheckForAdd(index);

    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);
    int numMoved = size - index;
    if (numMoved > 0)
        System.arraycopy(elementData, index, elementData, index + numNew,
                         numMoved);

    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    return numNew != 0;
}
```

### 4.10 移除、清空

移除指定位置的元素，随后元素组成的子序列依次向前移动一个位置

```java
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // 释放索引以便GC

    return oldValue;
}
```

如果指定对象存在列表中，移除第一次遇到的那个

```java
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
```

私有方法，没有边界检查，移除的元素不会作为结果返回

```java
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // 释放索引以便GC
}
```

移除列表中所有元素，成为一个空列表，size大小置0。 移出对象被GC，而ArrayList本身占用数组空间不会释放

```java
public void clear() {
    modCount++;

    // 释放被引用的对象，以便VM进行GC。注：请自行参考Java GC Roots工作方式
    for (int i = 0; i < size; i++)
        elementData[i] = null;

    size = 0;
}
```

移除指定范围包含的元素，并修改size

```java
protected void removeRange(int fromIndex, int toIndex) {
    modCount++;
    int numMoved = size - toIndex;
    System.arraycopy(elementData, toIndex, elementData, fromIndex,
                     numMoved);

    int newSize = size - (toIndex-fromIndex);
    for (int i = newSize; i < size; i++) {
        elementData[i] = null;
    }
    size = newSize;
}
```

下标检查

```java
private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

私有方法，检查下表是否超过下标

```java
private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

移除交集部分元素

```java
public boolean removeAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, false);
}
```

移除未包含在集合的元素，即保留交集部分

```java
public boolean retainAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, true);
}
```

```java
private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;
    int r = 0, w = 0;
    boolean modified = false;
    try {
        for (; r < size; r++)
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    } finally {
        // Preserve behavioral compatibility with AbstractCollection,
        // even if c.contains() throws.
        if (r != size) {
            System.arraycopy(elementData, r,
                             elementData, w,
                             size - r);
            w += size - r;
        }
        if (w != size) {
            // 释放索引以便GC
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;
            size = w;
            modified = true;
        }
    }
    return modified;
}
```


### 4.11 序列化

```java
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException{
    // Write out element count, and any hidden stuff
    int expectedModCount = modCount;
    s.defaultWriteObject();

    // Write out size as capacity for behavioural compatibility with clone()
    s.writeInt(size);

    // 按照适当顺序写入所有元素
    for (int i=0; i<size; i++) {
        s.writeObject(elementData[i]);
    }

    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}

private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    elementData = EMPTY_ELEMENTDATA;

    // 读取数组已占用大小和其他隐藏的隐藏
    s.defaultReadObject();

    // 读取数组总长度
    s.readInt(); // 忽略

    if (size > 0) {
        // 很像clone()，由size决定数组的大小
        ensureCapacityInternal(size);

        Object[] a = elementData;
        // 按照适当顺序读出所有元素
        for (int i=0; i<size; i++) {
            a[i] = s.readObject();
        }
    }
}
```

### 4.12 迭代器

```java
public ListIterator<E> listIterator(int index) {
    if (index < 0 || index > size)
        throw new IndexOutOfBoundsException("Index: "+index);
    return new ListItr(index);
}

public ListIterator<E> listIterator() {
    return new ListItr(0);
}

public Iterator<E> iterator() {
    return new Itr();
}

/**
 * 优化版AbstractList.Itr
 */
private class Itr implements Iterator<E> {
    int cursor;       // 下一个待返回元素序号
    int lastRet = -1; // 已经返回的最后一个元素的序号；-1表示并没有元素返回过
    int expectedModCount = modCount;

    public boolean hasNext() {
        return cursor != size;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        checkForComodification();
        int i = cursor;
        if (i >= size)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i + 1;
        return (E) elementData[lastRet = i];
    }

    public void remove() {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            ArrayList.this.remove(lastRet);
            cursor = lastRet;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

    @Override
    @SuppressWarnings("unchecked")
    public void forEachRemaining(Consumer<? super E> consumer) {
        Objects.requireNonNull(consumer);
        final int size = ArrayList.this.size;
        int i = cursor;
        if (i >= size) {
            return;
        }
        final Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length) {
            throw new ConcurrentModificationException();
        }
        while (i != size && modCount == expectedModCount) {
            consumer.accept((E) elementData[i++]);
        }
        // update once at end of iteration to reduce heap write traffic
        cursor = i;
        lastRet = i - 1;
        checkForComodification();
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}

//优化版的AbstractList.ListItr
private class ListItr extends Itr implements ListIterator<E> {
    ListItr(int index) {
        super();
        cursor = index;
    }

    public boolean hasPrevious() {
        return cursor != 0;
    }

    public int nextIndex() {
        return cursor;
    }

    public int previousIndex() {
        return cursor - 1;
    }

    @SuppressWarnings("unchecked")
    public E previous() {
        checkForComodification();
        int i = cursor - 1;
        if (i < 0)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i;
        return (E) elementData[lastRet = i];
    }

    public void set(E e) {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            ArrayList.this.set(lastRet, e);
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

    public void add(E e) {
        checkForComodification();

        try {
            int i = cursor;
            ArrayList.this.add(i, e);
            cursor = i + 1;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
}
```

### 4.13 子序列

```java
// 返回指定序号的子序列
public List<E> subList(int fromIndex, int toIndex) {
    subListRangeCheck(fromIndex, toIndex, size);
    return new SubList(this, 0, fromIndex, toIndex);
}

static void subListRangeCheck(int fromIndex, int toIndex, int size) {
    if (fromIndex < 0)
        throw new IndexOutOfBoundsException("fromIndex = " + fromIndex);
    if (toIndex > size)
        throw new IndexOutOfBoundsException("toIndex = " + toIndex);
    if (fromIndex > toIndex)
        throw new IllegalArgumentException("fromIndex(" + fromIndex + ") > toIndex(" + toIndex + ")");
}

private class SubList extends AbstractList<E> implements RandomAccess {
    private final AbstractList<E> parent;
    private final int parentOffset;
    private final int offset;
    int size;

    SubList(AbstractList<E> parent,
            int offset, int fromIndex, int toIndex) {
        this.parent = parent;
        this.parentOffset = fromIndex;
        this.offset = offset + fromIndex;
        this.size = toIndex - fromIndex;
        this.modCount = ArrayList.this.modCount;
    }

    public E set(int index, E e) {
        rangeCheck(index);
        checkForComodification();
        E oldValue = ArrayList.this.elementData(offset + index);
        ArrayList.this.elementData[offset + index] = e;
        return oldValue;
    }

    public E get(int index) {
        rangeCheck(index);
        checkForComodification();
        return ArrayList.this.elementData(offset + index);
    }

    public int size() {
        checkForComodification();
        return this.size;
    }

    public void add(int index, E e) {
        rangeCheckForAdd(index);
        checkForComodification();
        parent.add(parentOffset + index, e);
        this.modCount = parent.modCount;
        this.size++;
    }

    public E remove(int index) {
        rangeCheck(index);
        checkForComodification();
        E result = parent.remove(parentOffset + index);
        this.modCount = parent.modCount;
        this.size--;
        return result;
    }

    protected void removeRange(int fromIndex, int toIndex) {
        checkForComodification();
        parent.removeRange(parentOffset + fromIndex,
                           parentOffset + toIndex);
        this.modCount = parent.modCount;
        this.size -= toIndex - fromIndex;
    }

    public boolean addAll(Collection<? extends E> c) {
        return addAll(this.size, c);
    }

    public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);
        int cSize = c.size();
        if (cSize==0)
            return false;

        checkForComodification();
        parent.addAll(parentOffset + index, c);
        this.modCount = parent.modCount;
        this.size += cSize;
        return true;
    }

    public Iterator<E> iterator() {
        return listIterator();
    }

    public ListIterator<E> listIterator(final int index) {
        checkForComodification();
        rangeCheckForAdd(index);
        final int offset = this.offset;

        return new ListIterator<E>() {
            int cursor = index;
            int lastRet = -1;
            int expectedModCount = ArrayList.this.modCount;

            public boolean hasNext() {
                return cursor != SubList.this.size;
            }

            @SuppressWarnings("unchecked")
            public E next() {
                checkForComodification();
                int i = cursor;
                if (i >= SubList.this.size)
                    throw new NoSuchElementException();
                Object[] elementData = ArrayList.this.elementData;
                if (offset + i >= elementData.length)
                    throw new ConcurrentModificationException();
                cursor = i + 1;
                return (E) elementData[offset + (lastRet = i)];
            }

            public boolean hasPrevious() {
                return cursor != 0;
            }

            @SuppressWarnings("unchecked")
            public E previous() {
                checkForComodification();
                int i = cursor - 1;
                if (i < 0)
                    throw new NoSuchElementException();
                Object[] elementData = ArrayList.this.elementData;
                if (offset + i >= elementData.length)
                    throw new ConcurrentModificationException();
                cursor = i;
                return (E) elementData[offset + (lastRet = i)];
            }

            @SuppressWarnings("unchecked")
            public void forEachRemaining(Consumer<? super E> consumer) {
                Objects.requireNonNull(consumer);
                final int size = SubList.this.size;
                int i = cursor;
                if (i >= size) {
                    return;
                }
                final Object[] elementData = ArrayList.this.elementData;
                if (offset + i >= elementData.length) {
                    throw new ConcurrentModificationException();
                }
                while (i != size && modCount == expectedModCount) {
                    consumer.accept((E) elementData[offset + (i++)]);
                }
                // update once at end of iteration to reduce heap write traffic
                lastRet = cursor = i;
                checkForComodification();
            }

            public int nextIndex() {
                return cursor;
            }

            public int previousIndex() {
                return cursor - 1;
            }

            public void remove() {
                if (lastRet < 0)
                    throw new IllegalStateException();
                checkForComodification();

                try {
                    SubList.this.remove(lastRet);
                    cursor = lastRet;
                    lastRet = -1;
                    expectedModCount = ArrayList.this.modCount;
                } catch (IndexOutOfBoundsException ex) {
                    throw new ConcurrentModificationException();
                }
            }

            public void set(E e) {
                if (lastRet < 0)
                    throw new IllegalStateException();
                checkForComodification();

                try {
                    ArrayList.this.set(offset + lastRet, e);
                } catch (IndexOutOfBoundsException ex) {
                    throw new ConcurrentModificationException();
                }
            }

            public void add(E e) {
                checkForComodification();

                try {
                    int i = cursor;
                    SubList.this.add(i, e);
                    cursor = i + 1;
                    lastRet = -1;
                    expectedModCount = ArrayList.this.modCount;
                } catch (IndexOutOfBoundsException ex) {
                    throw new ConcurrentModificationException();
                }
            }

            final void checkForComodification() {
                if (expectedModCount != ArrayList.this.modCount)
                    throw new ConcurrentModificationException();
            }
        };
    }

    public List<E> subList(int fromIndex, int toIndex) {
        subListRangeCheck(fromIndex, toIndex, size);
        return new SubList(this, offset, fromIndex, toIndex);
    }

    private void rangeCheck(int index) {
        if (index < 0 || index >= this.size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    private void rangeCheckForAdd(int index) {
        if (index < 0 || index > this.size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    private String outOfBoundsMsg(int index) {
        return "Index: "+index+", Size: "+this.size;
    }

    private void checkForComodification() {
        if (ArrayList.this.modCount != this.modCount)
            throw new ConcurrentModificationException();
    }

    public Spliterator<E> spliterator() {
        checkForComodification();
        return new ArrayListSpliterator<E>(ArrayList.this, offset,
                                           offset + this.size, this.modCount);
    }
}
```

```java
@Override
public void forEach(Consumer<? super E> action) {
    Objects.requireNonNull(action);
    final int expectedModCount = modCount;
    @SuppressWarnings("unchecked")
    final E[] elementData = (E[]) this.elementData;
    final int size = this.size;
    for (int i=0; modCount == expectedModCount && i < size; i++) {
        action.accept(elementData[i]);
    }
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}
```

```java
/**
 * Creates a <em><a href="Spliterator.html#binding">late-binding</a></em>
 * and <em>fail-fast</em> {@link Spliterator} over the elements in this
 * list.
 *
 * <p>The {@code Spliterator} reports {@link Spliterator#SIZED},
 * {@link Spliterator#SUBSIZED}, and {@link Spliterator#ORDERED}.
 * Overriding implementations should document the reporting of additional
 * characteristic values.
 *
 * @return a {@code Spliterator} over the elements in this list
 * @since 1.8
 */
@Override
public Spliterator<E> spliterator() {
    return new ArrayListSpliterator<>(this, 0, -1, 0);
}

/** Index-based split-by-two, lazily initialized Spliterator */
static final class ArrayListSpliterator<E> implements Spliterator<E> {

    /*
     * If ArrayLists were immutable, or structurally immutable (no
     * adds, removes, etc), we could implement their spliterators
     * with Arrays.spliterator. Instead we detect as much
     * interference during traversal as practical without
     * sacrificing much performance. We rely primarily on
     * modCounts. These are not guaranteed to detect concurrency
     * violations, and are sometimes overly conservative about
     * within-thread interference, but detect enough problems to
     * be worthwhile in practice. To carry this out, we (1) lazily
     * initialize fence and expectedModCount until the latest
     * point that we need to commit to the state we are checking
     * against; thus improving precision.  (This doesn't apply to
     * SubLists, that create spliterators with current non-lazy
     * values).  (2) We perform only a single
     * ConcurrentModificationException check at the end of forEach
     * (the most performance-sensitive method). When using forEach
     * (as opposed to iterators), we can normally only detect
     * interference after actions, not before. Further
     * CME-triggering checks apply to all other possible
     * violations of assumptions for example null or too-small
     * elementData array given its size(), that could only have
     * occurred due to interference.  This allows the inner loop
     * of forEach to run without any further checks, and
     * simplifies lambda-resolution. While this does entail a
     * number of checks, note that in the common case of
     * list.stream().forEach(a), no checks or other computation
     * occur anywhere other than inside forEach itself.  The other
     * less-often-used methods cannot take advantage of most of
     * these streamlinings.
     */

    private final ArrayList<E> list;
    private int index; // current index, modified on advance/split
    private int fence; // -1 until used; then one past last index
    private int expectedModCount; // initialized when fence set

    /** Create new spliterator covering the given  range */
    ArrayListSpliterator(ArrayList<E> list, int origin, int fence,
                         int expectedModCount) {
        this.list = list; // OK if null unless traversed
        this.index = origin;
        this.fence = fence;
        this.expectedModCount = expectedModCount;
    }

    private int getFence() { // initialize fence to size on first use
        int hi; // (a specialized variant appears in method forEach)
        ArrayList<E> lst;
        if ((hi = fence) < 0) {
            if ((lst = list) == null)
                hi = fence = 0;
            else {
                expectedModCount = lst.modCount;
                hi = fence = lst.size;
            }
        }
        return hi;
    }

    public ArrayListSpliterator<E> trySplit() {
        int hi = getFence(), lo = index, mid = (lo + hi) >>> 1;
        return (lo >= mid) ? null : // divide range in half unless too small
            new ArrayListSpliterator<E>(list, lo, index = mid,
                                        expectedModCount);
    }

    public boolean tryAdvance(Consumer<? super E> action) {
        if (action == null)
            throw new NullPointerException();
        int hi = getFence(), i = index;
        if (i < hi) {
            index = i + 1;
            @SuppressWarnings("unchecked") E e = (E)list.elementData[i];
            action.accept(e);
            if (list.modCount != expectedModCount)
                throw new ConcurrentModificationException();
            return true;
        }
        return false;
    }

    public void forEachRemaining(Consumer<? super E> action) {
        int i, hi, mc; // hoist accesses and checks from loop
        ArrayList<E> lst; Object[] a;
        if (action == null)
            throw new NullPointerException();
        if ((lst = list) != null && (a = lst.elementData) != null) {
            if ((hi = fence) < 0) {
                mc = lst.modCount;
                hi = lst.size;
            }
            else
                mc = expectedModCount;
            if ((i = index) >= 0 && (index = hi) <= a.length) {
                for (; i < hi; ++i) {
                    @SuppressWarnings("unchecked") E e = (E) a[i];
                    action.accept(e);
                }
                if (lst.modCount == mc)
                    return;
            }
        }
        throw new ConcurrentModificationException();
    }

    public long estimateSize() {
        return (long) (getFence() - index);
    }

    public int characteristics() {
        return Spliterator.ORDERED | Spliterator.SIZED | Spliterator.SUBSIZED;
    }
}
```

```java
@Override
public boolean removeIf(Predicate<? super E> filter) {
    Objects.requireNonNull(filter);
    // figure out which elements are to be removed
    // any exception thrown from the filter predicate at this stage
    // will leave the collection unmodified
    int removeCount = 0;
    final BitSet removeSet = new BitSet(size);
    final int expectedModCount = modCount;
    final int size = this.size;
    for (int i=0; modCount == expectedModCount && i < size; i++) {
        @SuppressWarnings("unchecked")
        final E element = (E) elementData[i];
        if (filter.test(element)) {
            removeSet.set(i);
            removeCount++;
        }
    }
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }

    // shift surviving elements left over the spaces left by removed elements
    final boolean anyToRemove = removeCount > 0;
    if (anyToRemove) {
        final int newSize = size - removeCount;
        for (int i=0, j=0; (i < size) && (j < newSize); i++, j++) {
            i = removeSet.nextClearBit(i);
            elementData[j] = elementData[i];
        }
        for (int k=newSize; k < size; k++) {
            elementData[k] = null;  // Let gc do its work
        }
        this.size = newSize;
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
        modCount++;
    }

    return anyToRemove;
}

@Override
@SuppressWarnings("unchecked")
public void replaceAll(UnaryOperator<E> operator) {
    Objects.requireNonNull(operator);
    final int expectedModCount = modCount;
    final int size = this.size;
    for (int i=0; modCount == expectedModCount && i < size; i++) {
        elementData[i] = operator.apply((E) elementData[i]);
    }
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
    modCount++;
}

@Override
@SuppressWarnings("unchecked")
public void sort(Comparator<? super E> c) {
    final int expectedModCount = modCount;
    Arrays.sort((E[]) elementData, 0, size, c);
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
    modCount++;
}
```

