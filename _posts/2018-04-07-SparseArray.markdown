---
layout:     post
title:      "Android源码系列(10) -- SparseArray"
date:       2018-04-07
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Android源码系列
---

### 一、前言

SparseArrays<E>由Android原生提供的稀疏数组，用于代替HashMap的容器类。准确说是在一部分场景中能代替HashMap<Integer, Object>，提供从int映射到Object<E>的能力，优点是具有高效的内存利用率。

```java
public class SparseArray<E> implements Cloneable
```

SparseArrays使用基本类型int作为键，不像HashMap<Integer, Object>的键需把int装箱为Integer，避免了装箱、拆箱的性能损失。并使用内存利用率更高的数组而不是链表存放value，同时避免链表依赖的Entry。

用时间换空间的策略令SparseArrays不像HashMap那样占用大量内存，但在存取操作上需耗费相对更多时间。

从类注释能了解到：元素保存在数组中，通过二分法查找键，再用键的index找对应索引的值，由此可推测时间复杂度为O(log(N))。同有几百个key-value查找性能只有HashMap一半。由于key保存在mKeys数组，value保存在mValues数组，任何一次增删键值对都有可能重建两个数组。

不过，SparseArrays做了一定优化，如移除一个键值对时只会把mValues对应的Object标记为`DELETED`，等下一次同key插入新value时直接替换，且失效空间在数组扩容或回收空间时才处理。

总结主要应用场景：

- 类型为<int, Object>，若key是Integer建议直接用HashMap；
- 存储键值对量较少，避免出现查询带来的性能问题；
- 对存取时间不太敏感，但内存可用条件苛刻的设备；
- 不在Java标准库，仅在Android系统中提供；
- 支持按照key升序输出value；
- 非线程安全，或自行保证；

### 二、数据成员

```java
// 用于标记键对应Object已被删除的标志，起占位作用
private static final Object DELETED = new Object();
// 是否开启失效值处理的标志位，用于规整数组
private boolean mGarbage = false;

// 保存键的整形数组
private int[] mKeys;
// 保存值的数组，索引与键数组对应
private Object[] mValues;
// 数组容量
private int mSize;
```

### 三、构造方法

创建一个没有初始元素且初始化大小为10的实例

```java
public SparseArray() {
    this(10);
}
```
指定初始化容量的构造方法。当初始容量设置为0时，mKeys数组和mValues数组各使用一个轻量级空数组初始化，否则按照指定容量进行初始化。
```java
public SparseArray(int initialCapacity) {
    if (initialCapacity == 0) {
        mKeys = EmptyArray.INT;
        mValues = EmptyArray.OBJECT;
    } else {
        mValues = ArrayUtils.newUnpaddedObjectArray(initialCapacity);
        mKeys = new int[mValues.length]; // key类型为int[]
    }
    // 没有存放任何键值对，所以mSize为0
    mSize = 0;
}
```
### 四、查询
```java
// 获取指定key的value，否则返回null
public E get(int key) {
    return get(key, null);
}

// 获取指定key的value，命失返回指定对象
@SuppressWarnings("unchecked")
public E get(int key, E valueIfKeyNotFound) {
    // 在mKeys的mSize有效范围内二分查找key的数组下标i
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i < 0 || mValues[i] == DELETED) {
        // 没有对应的值或值已被移除，返回valueIfKeyNotFound
        return valueIfKeyNotFound;
    } else {
        // 显式类型转换并返回
        return (E) mValues[i];
    }
}
```
### 五、移除
```java
// 移除key对应value
public void delete(int key) {
    // 在mKeys的mSize有效范围内二分查找key的数组下标i
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);
    // i>=0表示key存在
    if (i >= 0) {
        // 检查key对应value是否已被删除
        if (mValues[i] != DELETED) {
            mValues[i] = DELETED;
            mGarbage = true;
        }
    }
}

// 移除并返回指定key对应value，若不存在返回null
public E removeReturnOld(int key) {
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    if (i >= 0) {
        if (mValues[i] != DELETED) {
            final E old = (E) mValues[i];
            mValues[i] = DELETED;
            mGarbage = true;
            return old;
        }
    }
    return null;
}

// delete(key)的别名方法
public void remove(int key) {
    delete(key);
}

// 移除指定下标的value
public void removeAt(int index) {
    if (mValues[index] != DELETED) {
        mValues[index] = DELETED;
        mGarbage = true;
    }
}

// 移除指定索引开始的连续数个值
public void removeAtRange(int index, int size) {
    final int end = Math.min(mSize, index + size);
    for (int i = index; i < end; i++) {
        removeAt(i);
    }
}

// 清除集合
public void clear() {
    int n = mSize;
    Object[] values = mValues;
    // 所有mValue置null
    for (int i = 0; i < n; i++) {
        values[i] = null;
    }
    // 数组容量置0
    mSize = 0;
    mGarbage = false;
}
```
### 六、插入
```java
// 在指定key位置放入值，如果原位置已经存在vlaue，则直接替换
public void put(int key, E value) {
    // 在mKeys的mSize有效范围内二分查找key的数组下标i
    int i = ContainerHelpers.binarySearch(mKeys, mSize, key);

    // 直接用新value直接替换旧value，不管是否已被置为DELETE
    if (i >= 0) {
        mValues[i] = value;
    } else {
        //返回index是负数表明key不存在。对返i取反得到插入的位置i
        // 例：i = ~i -> 2 = ~(-3)
        i = ~i;

        // i没有越界，且i的value已被删除，则直接重用此空间
        if (i < mSize && mValues[i] == DELETED) {
            mKeys[i] = key;
            mValues[i] = value;
            return;
        }

        if (mGarbage && mSize >= mKeys.length) {
            gc();

            // 由于gc()可能会规整数组，需重新查找key可以插入的下标i
            i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);
        }

        //插入key，可触发mKeys数组扩容
        mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
        //插入value，可触发mValues数组扩容
        mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);
        mSize++;
    }
}

public void append(int key, E value) {
    if (mSize != 0 && key <= mKeys[mSize - 1]) {
        put(key, value);
        return;
    }

    if (mGarbage && mSize >= mKeys.length) {
        gc();
    }

    mKeys = GrowingArrayUtils.append(mKeys, mSize, key);
    mValues = GrowingArrayUtils.append(mValues, mSize, value);
    mSize++;
}
```
### 七、其他

```java
// 返回当前保存的键值对数量
public int size() {
    if (mGarbage) {
        gc();
    }

    return mSize;
}

// index范围在[0, size()-1]之内，返回index的key
// index为0，返回mKeys最小key；
// index为size()-1，返回mKeys最大key
// 小于0或大于等于size()会出现未知结果
public int keyAt(int index) {
    if (mGarbage) {
        gc();
    }

    return mKeys[index];
}

// index范围在[0, size()-1]之内，返回index下标在mValue的value；
// index为0，返回mKeys最小key对应的value；
// index为size()-1，返回mKeys最大key对应的value；
// 小于0或大于等于size()会出现未知结果
@SuppressWarnings("unchecked")
public E valueAt(int index) {
    // 清理废弃value
    if (mGarbage) {
        gc();
    }

    return (E) mValues[index];
}

// 返回key在mKeys中对应的索引值，不存在则返回负数
public int indexOfKey(int key) {
    if (mGarbage) {
        gc();
    }

    return ContainerHelpers.binarySearch(mKeys, mSize, key);
}

// 返回指定value在mValues的索引值，不存在返回-1
public int indexOfValue(E value) {
    if (mGarbage) {
        gc();
    }

    // 遍历所有mValues的value，并查找是否命中指定value
    for (int i = 0; i < mSize; i++)
        if (mValues[i] == value)
            return i;
    // 命失指定value返回-1
    return -1;
}

// 清理已被标记为DELETE的value，并规整mKeys和mValues数组空间
private void gc() {
    int n = mSize;
    int o = 0;
    int[] keys = mKeys;
    Object[] values = mValues;

    // 规整数组空间，把后面有效的key-value向数组前面移动排列，不会引起数组大小变化
    for (int i = 0; i < n; i++) {
        Object val = values[i];

        if (val != DELETED) {
            if (i != o) {
                keys[o] = keys[i];
                values[o] = val;
                values[i] = null;
            }

            o++;
        }
    }

    // 已经清理数组空间，标记为false
    mGarbage = false;
    // 规整之后有效的键值对数量
    mSize = o;
}
```

### 八、修改

index范围在[0, size()-1]之内，修改index下标在mValue的value。

index为0，修改mKeys最小key对应的value。index为size()-1，修改mKeys最大key对应的value。

小于0或大于等于size()会出现未知结果

```java
public void setValueAt(int index, E value) {
    if (mGarbage) {
        gc();
    }

    mValues[index] = value;
}
```

### 九、ContainerHelpers类

```java
class ContainerHelpers {

    static int binarySearch(int[] array, int size, int value) {
        // low：低位
        int lo = 0;
        // high: 高位
        int hi = size - 1;
        
        // 二分法循环查找
        while (lo <= hi) {
            // 假设lo=0，hi=9，可知(lo + hi) >>> 1为4
            // 向左位移等同乘2，向右位移等同除2，助记：左乘右除
            final int mid = (lo + hi) >>> 1;
            final int midVal = array[mid];

            if (midVal < value) {
                lo = mid + 1;
            } else if (midVal > value) {
                hi = mid - 1;
            } else {
                return mid;  // value命中
            }
        }
        return ~lo;  // value不存在
    }

    static int binarySearch(long[] array, int size, long value) {
        int lo = 0;
        int hi = size - 1;

        while (lo <= hi) {
            final int mid = (lo + hi) >>> 1;
            final long midVal = array[mid];

            if (midVal < value) {
                lo = mid + 1;
            } else if (midVal > value) {
                hi = mid - 1;
            } else {
                return mid;  // value命中
            }
        }
        return ~lo;  // value不存在
    }
}
```

