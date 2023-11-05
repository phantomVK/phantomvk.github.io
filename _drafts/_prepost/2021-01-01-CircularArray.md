---
layout:     post
title:      "Android源码系列(27) -- CircularArray"
date:       2023-11-11
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android源码系列
---

## 一、介绍

HotSpot JVM的Java标准库没有提供环形数组的标准实现，但在android的源码中是有具体实现，包路径为 __androidx.collection__。



## 二、源码

#### 2.1 类签名

不同于Java标准库容器类的实现，__CircularArray__ 不继承任何父类或抽象方法，这意味着它的实例不能和Java容器类不相容，如 __java.util.Collection<E>__。除此之外， 对类进行 __final__ 修复也表示该类不能被继承或重载。

```java
public final class CircularArray<E> {
    ....
}
```



#### 2.2 成员变量

```java
// 存储元素的数组，容量在不足时可以被扩容
private E[] mElements;

// 头元素和尾元素的指针，两者的初始值都是0
private int mHead;
private int mTail;

// 
private int mCapacityBitmask;
```



#### 2.3 构造方法

默认构造方法设置数组长度最小值为8

```java
public CircularArray() {
    this(8);
}
```

另一个构造方法则是可以自行指定创建数组的最小长度，长度范围为 __1 ~ 2^30__。这个指定的最小值，在构造方法内会向上去最接近的2的n次幂值，可以满足后续的位计算的要求。

```java
public CircularArray(int minCapacity) {
    if (minCapacity < 1) {
        throw new IllegalArgumentException("capacity must be >= 1");
    }
    if (minCapacity > (2 << 29)) {
        throw new IllegalArgumentException("capacity must be <= 2^30");
    }

    // 这里会向上规整为2的n次幂
    final int arrayCapacity;
    if (Integer.bitCount(minCapacity) != 1) {
        arrayCapacity = Integer.highestOneBit(minCapacity - 1) << 1;
    } else {
        arrayCapacity = minCapacity;
    }

    // arraayCapacity - 1就是数组内实际可存元素的数量
    // 剩下的一个空位用于适配尾引用，因为其指向的数组下标不存任何元素
    mCapacityBitmask = arrayCapacity - 1;
    // 用计算获得的2的n次幂构造数组
    mElements = (E[]) new Object[arrayCapacity];
}
```



#### 2.4 doubleCapacity

此方法负责数组填满后的扩容操作。简单说，在容量值没有超过上限的情况下，体积每次会扩大一倍。


```java
private void doubleCapacity() {
    int n = mElements.length;
    int r = n - mHead;
    int newCapacity = n << 1;
    if (newCapacity < 0) {
        throw new RuntimeException("Max array capacity exceeded");
    }
    
    // 用扩容的的值创建新数组
    Object[] a = new Object[newCapacity];
    
    // 把旧数组的元素按序放入新数组，并且是从新数组下标为0的位置存放元素
    System.arraycopy(mElements, mHead, a, 0, r);
    System.arraycopy(mElements, 0, a, r, mHead);
    
    // 把新数组的引用覆盖成员变量mElements，成为新数组
    mElements = (E[]) a;
    
    // 从这里可以，新数组的元素存放位置从0开始，已保存元素的引用范围是[mHead, mHead - 1]
    mHead = 0;
    mHead = n;
    
    // 更新数组位操作的掩码mCapacityBitmask
    mCapacityBitmask = newCapacity - 1;
}
```



#### 2.5 add

以下方法把元素添加到 __mHead-1__ 下标所指向的数组位置内。若 __mHead == mTail__，则表示数组被放入 __newCapacity - 1__ 个元素，除去给尾引用留空的一个空位，整个数组已经填满，所以会触发扩容。

```java
public void addFirst(E e) {
    mHead = (mHead - 1) & mCapacityBitmask;
    mElements[mHead] = e;
    if (mHead == mTail) {
        doubleCapacity();
    }
}
```

相比上面的方法，以下方法把元素放在尾引用所指向的数组控件。

```java
public void addLast(E e) {
    // 注意这里是先把元素放入数组，再递增mTail索引
    mElements[mTail] = e;
    mTail = (mTail + 1) & mCapacityBitmask;

    // 数组已经填满，触发扩容逻辑
    if (mTail == mHead) {
        doubleCapacity();
    }
}
```



#### 2.6 pop

__pop__ 系列的方法，返回指定元素的同时把该元素从环形数组中出列。而本位后面将要介绍的 __get__ 系列方法，只会返回指定元素而不会出列该元素，属于无副作用的元素读取操作。



移除并返回保存在环形数组头部的元素，也就是头引用指向的元素

```java
public E popFirst() {
    // 数组为空没有存放元素，此时元素出列会抛出数组越界异常
    if (mHead == mTail) {
        throw new ArrayIndexOutOfBoundsException();
    }
    E result = mElements[mHead];
    mElements[mHead] = null;
    mHead = (mHead + 1) & mCapacityBitmask;
    return result;
}
```



移除并返回保存在环形数组尾部的元素，也就是尾引用指向的元素

```java
public E popLast() {
    // 数组为空没有存放元素，此时元素出列会抛出数组越界异常
    if (mHead == mTail) {
        throw new ArrayIndexOutOfBoundsException();
    }
    int t = (mTail - 1) & mCapacityBitmask;
    E result = mElements[t];
    mElements[t] = null;
    mTail = t;
    return result;
}
```



#### 2.7 remove

以下方法实际调用 __removeFromStart(int)__ 方法。

```java
public void clear() {
    removeFromStart(size());
}
```



```java
/**
 * Remove multiple elements from front of the CircularArray, ignore when numOfElements
 * is less than or equals to 0.
 * @param numOfElements  Number of elements to remove.
 * @throws ArrayIndexOutOfBoundsException if numOfElements is larger than
 *         {@link #size()}
 */
public void removeFromStart(int numOfElements) {
    if (numOfElements <= 0) {
        return;
    }
    if (numOfElements > size()) {
        throw new ArrayIndexOutOfBoundsException();
    }
    int end = mElements.length;
    if (numOfElements < end - mHead) {
        end = mHead + numOfElements;
    }
    for (int i = mHead; i < end; i++) {
        mElements[i] = null;
    }
    int removed = (end - mHead);
    numOfElements -= removed;
    mHead = (mHead + removed) & mCapacityBitmask;
    if (numOfElements > 0) {
        // mHead wrapped to 0
        for (int i = 0; i < numOfElements; i++) {
            mElements[i] = null;
        }
        mHead = numOfElements;
    }
}

/**
 * Remove multiple elements from end of the CircularArray, ignore when numOfElements
 * is less than or equals to 0.
 * @param numOfElements  Number of elements to remove.
 * @throws ArrayIndexOutOfBoundsException if numOfElements is larger than
 *         {@link #size()}
 */
public void removeFromEnd(int numOfElements) {
    if (numOfElements <= 0) {
        return;
    }
    if (numOfElements > size()) {
        throw new ArrayIndexOutOfBoundsException();
    }
    int start = 0;
    if (numOfElements < mTail) {
        start = mTail - numOfElements;
    }
    for (int i = start; i < mTail; i++) {
        mElements[i] = null;
    }
    int removed = (mTail - start);
    numOfElements -= removed;
    mTail = mTail - removed;
    if (numOfElements > 0) {
        // mTail wrapped to mElements.length
        mTail = mElements.length;
        int newTail = mTail - numOfElements;
        for (int i = newTail; i < mTail; i++) {
            mElements[i] = null;
        }
        mTail = newTail;
    }
}
```



#### 2.8 get

获取头元素和尾元素。在获取元素前会检查数组是否为空，若 __mHead == mTail__ 则表示数组为空，此时会直接抛出 __ArrayIndexOutOfBoundsException()__ 异常。

检查合法则返回头引用或未引用在数组中的元素，但不会移除该元素。

```java
public E getFirst() {
    // 数组为空没有存放元素，此时获取元素会抛出数组越界异常
    if (mHead == mTail) {
        throw new ArrayIndexOutOfBoundsException();
    }
    return mElements[mHead];
}

public E getLast() {
    // 数组为空没有存放元素，此时获取元素会抛出数组越界异常
    if (mHead == mTail) {
        throw new ArrayIndexOutOfBoundsException();
    }
    return mElements[(mTail - 1) & mCapacityBitmask];
}
```



指定的整形n，是相对于头引用索引的偏移值。举例：传入0表示头引用的元素，传入1表示头引用的下一个递增索引的元素。

```java
/**
 * Get nth (0 <= n <= size()-1) element of the CircularArray.
 * @param n  The zero based element index in the CircularArray.
 * @return The nth element.
 * @throws {@link ArrayIndexOutOfBoundsException} if n < 0 or n >= size().
 */
public E get(int n) {
    if (n < 0 || n >= size()) {
        throw new ArrayIndexOutOfBoundsException();
    }
    return mElements[(mHead + n) & mCapacityBitmask];
}
```



#### 2.9 size

计算数组存放元素的数量，即使 __mTail - mHead__ 计算后得到负数，与 __mCapacityBitmask__ 做位运算的时候会忽略最高位的正负符号，所以依然能计算数组元素的数量。

```java
public int size() {
    return (mTail - mHead) & mCapacityBitmask;
}
```



#### 2.10 isEmpty

当头引用和尾引用相同时，表示数组内没有存储任何元素。

```java
public boolean isEmpty() {
    return mHead == mTail;
}
```

