---
layout:     post
title:      "Android源码系列(27) -- CircularArray"
date:       2023-11-11
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Android源码系列
---

## 一、介绍

Oracle官方HotSpot JVM没有提供环形数组的标准实现，但这并不妨碍Android标准库提供实现，对应包路径为 __androidx.collection__。


这个环形数组的头部、尾部均能增删元素，使用方式较为灵活。



## 二、源码

### 2.1 类签名

有别于Java标准库的容器类实现，__CircularArray__ 没有继承父类或抽象方法。这意味该类实例和Java容器抽象接口不相容，例如：__java.util.Collection\<E\>__。

此外，设置 __final__ 修饰符也表示该类不能被继承或方法重写。

```java
public final class CircularArray<E> {
    ....
}
```



由于类实现不包含锁控制，属于线程不安全容器类。在多线程并发场景，要自行评估线程安全问题。



### 2.2 成员变量

用于存储元素的数组，容量不足时可扩容。

```java
private E[] mElements;
```



头引用、尾引用初始值都是0。

```java
private int mHead;
private int mTail;
```



__mCapacityBitmask__ 为 __数组总长度-1__，在构造方法中赋值。

```java
private int mCapacityBitmask;
```



举个例子：假设数组长度为8(二进制为 __1000__)，则 __mCapacityBitmask__ 的值为7(二进制为 __0111__)。所以 __mCapacityBitmask__ 可以用作整形的低位掩码，刚好覆盖索引范围 __[0, 7]__。



### 2.3 构造方法

默认构造方法设置数组长度最小值为8

```java
public CircularArray() {
    this(8);
}
```



另一个构造方法，可以指定创建数组的最小长度，长度范围为 __1 ~ 2^29__。这个传入构造方法的最小值实参，会向上取最接近 __2^n__ 的值，以便满足环形数组位操作要求。


```java
public CircularArray(int minCapacity) {
    // 检查minCapacity值范围处于[1, 2 << 29]，否则抛出IllegalArgumentException
    if (minCapacity < 1) {
        throw new IllegalArgumentException("capacity must be >= 1");
    }

    if (minCapacity > (2 << 29)) {
        throw new IllegalArgumentException("capacity must be <= 2^30");
    }

    // 这里会向上规整为2的n次幂，不然无法满足mCapacityBitmask低位全1
    final int arrayCapacity;
    if (Integer.bitCount(minCapacity) != 1) {
        arrayCapacity = Integer.highestOneBit(minCapacity - 1) << 1;
    } else {
        // 整形二进制只含一个1，满足2^n这个条件，直接使用
        arrayCapacity = minCapacity;
    }

    // arraayCapacity - 1 为数组可存入元素的总量
    // 剩下一个空位留给尾引用，因为它指向的数组下标不存任何元素
    mCapacityBitmask = arrayCapacity - 1;
    // 通过计算获得的2^n结果构造数组
    mElements = (E[]) new Object[arrayCapacity];
}
```



### 2.4 doubleCapacity

此方法负责数组的扩容操作。简单说，容量值没有超过上限时，每次扩容数组长度比原来长一倍。


```java
private void doubleCapacity() {
    int n = mElements.length;
    // mHead到数组结束之间，存储r个有效元素
    int r = n - mHead;
    int newCapacity = n << 1;
    
    // newCapacity位移后，若整形最高位为1(成为负数)，不能用作数组容量值
    // 长度增长到最大阈值会出现此现象，也就不允许继续扩容
    if (newCapacity < 0) {
        throw new RuntimeException("Max array capacity exceeded");
    }
    
    // 用新容量值创建新数组
    Object[] a = new Object[newCapacity];
    
    // 按序复制旧数组mHead至mTail的元素，从新数组下标为0开始位置按序放入
    // arraycopy(Object src, int srcPos, Object dest, int destPos, mint length);
    System.arraycopy(mElements, mHead, a, 0, r);
    System.arraycopy(mElements, 0, a, r, mHead);
    
    // 新数组引用赋值给成员变量mElements
    mElements = (E[]) a;
    
    // 新数组从0开始存放元素，对应索引范围是[0, n-1]
    mHead = 0;
    mTail = n;

    // 更新位操作的掩码 mCapacityBitmask
    mCapacityBitmask = newCapacity - 1;
}
```

数组元素复制时，首先从旧数组的 __mHead__ 开始拷贝元素到新数组。

![doubleCapacity_from_mHead](/img/android/CircularArray/doubleCapacity_from_mHead.png)


环形数组的的 __mTail__ 索引可能小于 __mHead__，所以要回到旧数组头部，从旧数组下标为 __0__ 的位置继续复制元素直至遇到 __mTail__。

![doubleCapacity_from_mHead](/img/android/CircularArray/doubleCapacity_to_mTail.png)


所有元素复制完成后，用 __a__ 数组引用覆写 __mElement__ 变量。



### 2.5 add

#### 2.5.1 addFirst

以下方法把元素添加到 __mHead-1__ 下标所指向的数组位置。

若遇到 __mHead == mTail__，表示数组已放入 __newCapacity - 1__ 个元素，算上被尾引用占用的空位，整个数组已经填满，因此触发扩容操作扩大数组长度。

```java
public void addFirst(E e) {
    // 头引用下标递减，指向前一个数组空位置
    mHead = (mHead - 1) & mCapacityBitmask;

    // 在空位置存入新元素
    mElements[mHead] = e;

    // 头引用和尾引用相遇表示数组已满，触发扩容逻辑
    if (mHead == mTail) {
        doubleCapacity();
    }
}
```

先计算 __mHead__ 的新索引，然后在新索引位置放入新元素。

![addFirst](/img/android/CircularArray/addFirst.png)

#### 2.5.2 addLast

相比 __addFirst__ 方法，__addLast__ 把元素存入当前 __mTail__ 指向数组的空位，并更新 __mTail__ 下标值指向下一个空位。

```java
public void addLast(E e) {
    // 这里先把元素放入数组
    mElements[mTail] = e;

    // 递增mTail索引，此时mTail指向一个空位置
    mTail = (mTail + 1) & mCapacityBitmask;

    // 数组已经填满，触发扩容逻辑
    if (mTail == mHead) {
        doubleCapacity();
    }
}
```

若 __mTail__ 原本指向数组最后一个位置，插入新元素后 __mTail__ 会指向数组头部，也就是 __index == 0__ 的位置。

![addLast](/img/android/CircularArray/addLast.png)



### 2.6 pop

__pop__ 系列方法返回结果，同时从环形数组出列该元素。而下文将要介绍的 __get__ 系列方法只返回指定元素结果、不会出列该元素，是无副作用的元素读取操作。


#### 2.6.1 popFirst

返回并移除保存在环形数组头部的元素，也就是头引用指向的元素。

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

#### 2.6.2 popLast

移除并返回保存在环形数组尾部的元素，也就是尾引用所指下标前一个位置的元素。

```java
public E popLast() {
    // 数组为空没有存放元素，此时出列元素触发数组越界异常
    if (mHead == mTail) {
        throw new ArrayIndexOutOfBoundsException();
    }

    // mTail索引本身指向空位置，所以 mTail - 1 才是有效的尾元素
    int t = (mTail - 1) & mCapacityBitmask;
    // 临时记录该元素，并把数组下标的位置置空
    E result = mElements[t];    
    mElements[t] = null;

    // 更新mTail索引
    mTail = t;

    // 返回结果
    return result;
}
```



### 2.7 remove

#### 2.7.1 clear

清空环形数组，实际调用 __removeFromStart(int)__ 方法，看下文的解释。

```java
public void clear() {
    removeFromStart(size());
}
```


#### 2.7.2 removeFromStart

从数组头索引 __mHead__ 开始，向数组尾部方向移除 __numOfElements__ 个元素。

```java
public void removeFromStart(int numOfElements) {
    // 检查numOfElements取值是否合法
    if (numOfElements <= 0) {
        return;
    }
    // 检查数组越界
    if (numOfElements > size()) {
        throw new ArrayIndexOutOfBoundsException();
    }
    
    int end = mElements.length;

    // [mHead, arrayCapacity)间元素数比numOfElements多，直接移除对应数量元素
    if (numOfElements < end - mHead) {
        end = mHead + numOfElements;
    }

    // 元素移除范围[mHead, mHead + numOfElements)
    for (int i = mHead; i < end; i++) {
        mElements[i] = null;
    }

    // 计算当前已经移除多少个元素
    int removed = (end - mHead);
    // 从总数减去已移除元素数量
    numOfElements -= removed;

    // 从mHead开始移除removed个元素，要增加mHead的索引下标
    mHead = (mHead + removed) & mCapacityBitmask;

    // 没有移除足量元素，回到数组头继续移除[0, numOfElements-1]之间元素
    if (numOfElements > 0) {
        for (int i = 0; i < numOfElements; i++) {
            mElements[i] = null;
        }

        // 修正mHead为数组存储首个元素的下标，此时 mHead <= mTail
        mHead = numOfElements;
    }
}
```

#### 2.7.3 removeFromEnd

从数组尾索引 __mTail__，向数组头部方向移除元素。

```java
public void removeFromEnd(int numOfElements) {
    // 检查numOfElements取值是否合法
    if (numOfElements <= 0) {
        return;
    }
    // 检查数组越界
    if (numOfElements > size()) {
        throw new ArrayIndexOutOfBoundsException();
    }

    // 从尾索引逆向计算清除索引的范围
    int start = 0;
    if (numOfElements < mTail) {
        start = mTail - numOfElements;
    }

    // 清除[start, mTail - 1]间元素
    for (int i = start; i < mTail; i++) {
        mElements[i] = null;
    }

    // 计算当前已经移除多少个元素
    int removed = (mTail - start);
    // 从总数减去已移除元素数量
    numOfElements -= removed;

    // 从mTail往前移除removed个元素，对应递减mTail的索引下标
    mTail = mTail - removed;
    
    // 没有移除足量元素，跳到数组尾继续移除[newTail, mTail-1]间元素
    if (numOfElements > 0) {
        // 尾引用回到数组的尾部
        mTail = mElements.length;
        // 继续从尾引用向数组头部移除元素，计算mTail新索引
        int newTail = mTail - numOfElements;

        for (int i = newTail; i < mTail; i++) {
            mElements[i] = null;
        }

        // 修正mTail索引，此时 mHead <= mTail
        mTail = newTail;
    }
}
```



### 2.8 get

获取头元素和尾元素。在获取元素前会检查数组是否为空。__mHead == mTail__ 表示数组为空，此时会直接抛出 __ArrayIndexOutOfBoundsException()__ 异常。

检查合法则返回数组头、数组尾指向的结果，但不会移除该元素。

```java
public E getFirst() {
    // 数组为空没有存放元素，此时获取元素会出现数组越界异常
    if (mHead == mTail) {
        throw new ArrayIndexOutOfBoundsException();
    }
    return mElements[mHead];
}

public E getLast() {
    // 数组为空没有存放元素，此时获取元素会出现数组越界异常
    if (mHead == mTail) {
        throw new ArrayIndexOutOfBoundsException();
    }

    // mTail索引位置不存储元素，所以尾部有效元素索引是 mTail - 1
    return mElements[(mTail - 1) & mCapacityBitmask];
}
```



传入的整形形参 __n__ 是相对于头引用索引的偏移值。例如：传入0表示头引用元素，传入1表示头引用下一个递增索引的元素。

```java
public E get(int n) {
    // n 取值范围 [0, size() - 1]
    if (n < 0 || n >= size()) {
        throw new ArrayIndexOutOfBoundsException();
    }
    return mElements[(mHead + n) & mCapacityBitmask];
}
```



### 2.9 size

返回数组已保存元素数量。即使 __mTail - mHead__ 计算后得到负数，因为 __mCapacityBitmask__ 做位运算时会忽略整形最高位的正负符号，所以依然能得出数组元素数量的非负数。

```java
public int size() {
    return (mTail - mHead) & mCapacityBitmask;
}
```



### 2.10 isEmpty

头引用和尾引用相同时数组内不存储任何元素，也是实例创建后的初始状态。

```java
public boolean isEmpty() {
    return mHead == mTail;
}
```

数组为空时头引用、尾引用示意：

![isEmpty](/img/android/CircularArray/isEmpty.png)


有两种情况可令 __mHead__ 与 __mTail__ 相等：

1. 数组为空时，也就是上文的 __isEmpty()__ 为 __true__；
2. 数组含有 __arrayCapacity - 1__ 个元素并触发扩容操作，此时两个引用下标短暂相等。直到数组完成扩容，这两个引用会被修正且不再相等；


