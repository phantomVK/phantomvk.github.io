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

源码版本为JDK8，ArrayList实现了`随机存储`、`克隆`、`序列化`接口

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

## 二、数据成员

```java
private static final long serialVersionUID = 8683452581122892189L;
private static final int DEFAULT_CAPACITY = 10; // 缺省值
```

构造函数方法参数为0的数组用此空数组标识：

```java
private static final Object[] EMPTY_ELEMENTDATA = {};
```

无参构造方法使用此空数组作为标识，以便初始化数组时决定容量缺省值：

```java
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
```

ArrayList的总长度就是这个总长度，如果构建前等于`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`，第一个元素加入时构建序列长度初始化为10

```java
transient Object[] elementData;
```

ArrayList大小，指已加入元素的数量

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

构造时指定ArrayList的容量，则空间会立即创建。如果列表的长度较短且可预知，此构造方法能避免动态扩展造成空间浪费。

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

通过集合构建ArrayList，顺序由集合迭代器指定顺序为准。ArrayList长度和Collection长度一致。

```java
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        this.elementData = EMPTY_ELEMENTDATA; // 空集合构建为空数组列表，等同initialCapacity为0的情况
    }
}
```

## 四、方法

### 4.1 裁剪

把列表长度裁剪到实际占用长度，用于释放未占用空间。如果数组没有保存元素就设为空数组，否则缩短数组长度到占用长度。

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

使用默认构造方法指向`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`，调用下列方法时`minCapacity`大于10，数组才扩增到`minCapacity`。

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

```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // minCapacity必须比数组长度大才扩展空间
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

#### 4.2.3 ensureCapacityInternal()

私有方法，取DEFAULT_CAPACITY=10和minCapacity两者最大值

```java
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}
```

#### 4.2.4 MAX_ARRAY_SIZE

数组最大申请空间，有的虚拟机实现会把对象头信息保存在数组中，尝试分配更大内存空间在这种情况下会造成OOM：请求数字大小超过VM的限制。为了保护不同虚拟机实现的安全性，最大申请空间保留一部分空间。

```java
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

#### 4.2.5 grow()

就算`minCapacity`比数组长度大，也不一定会采用`minCapacity`的值。因为每次数组扩增不是在原数组上扩展，而是创建新数组并拷贝旧数组内容到新数组。若每次扩增只增加1个长度，尤其在连续添加新元素的场景下，扩容后废弃的旧数组对象将对GC造成极大压力。

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

### 4.4 列表为空
元素数量是否为0

```java
public boolean isEmpty() {
    return size == 0;
}
```

### 4.5 包含元素

找指定对象是否保存在列表中

```java
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}
```

### 4.6 元素查找

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

### 4.7 浅克隆

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

### 4.8 返回数组

按照列表原顺序返回一个新数组，新数组和原ArrayList互相独立

```java
public Object[] toArray() {
    return Arrays.copyOf(elementData, size);
}
```

用自行传入的数组保存列表的元素，类型与传入相同。传入数组多余空位置为null，否则创建新数组。当传入的数组长度不足，可知返回的数组和传入数组不是同一个对象。

```java
@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a) {
    // 传入的数组a不足以保存elementData，则需要新创建Array
    if (a.length < size)
        // Make a new array of a's runtime type, but my contents:
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());

    // 传入的数组a足够存放elementData
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```

### 4.9 返回指定下标元素

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

### 4.10 加入

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

### 4.11 移除、清空

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

如果指定对象存在列表中，移除第一次遇到的

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

移除列表中所有元素，size大小置0。 移出对象被GC，而ArrayList本身占用数组空间不会释放

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

私有方法，检查下标越界

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