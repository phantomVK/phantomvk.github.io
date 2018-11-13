---
layout:     post
title:      "Java源码系列(14) -- CopyOnWriteArrayList"
date:       2018-08-09
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、类签名

ArrayList在多线程操作下是不安全的，为此应使用CopyOnWriteArrayList。通过CopyOnWrite(简称COW，写时复制)策略，所有读取共享同一个数组对象，修改时另拷贝出新数组，操作在新数组上完成后再用此新数组替换旧数组。

```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

由于修改时方法会自行拷贝得到新数组，所以在这段时间，内存中同时存在原数组对象和新数组对象。如果修改操作过于频繁，产生大量废弃对象将增加垃圾回收负担。

由此，可推理出此类适合在读多写少的场景下使用。通过读写分离，即使修改操作费时也不会阻塞读取，而读取的数组数据未必是最新的。还有修改操作是线程安全的，每次最多只有一个线程在进行修改，以此保证数据最终一致性。

此次源码来自JDK10，和之前版本有一定差别。

## 二、数据成员

通过网上阅读JDK8版本的CopyOnWriteArrayList源码，可了解以前约束同步使用的是ReentrantLock。而在JDK10中用synchronized (lock)方式，暂时不知道对性能有多大提升。

```java
final transient Object lock = new Object();
```

变量array只能通过getArray()、setArray()获取，见第四节。注意，array变量的类型是Object[]，而不是E[]。

```java
private transient volatile Object[] array;
```

## 三、构造方法

构造空列表
```java
public CopyOnWriteArrayList() {
    setArray(new Object[0]);
}
```

从集合c构建实例
```java
public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    if (c.getClass() == CopyOnWriteArrayList.class)
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        elements = c.toArray();
        // defend against c.toArray (incorrectly) not returning Object[]
        // (see e.g. https://bugs.openjdk.java.net/browse/JDK-6260652)
        if (elements.getClass() != Object[].class)
            // 类型不是Object[].class，重新拷贝
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    setArray(elements);
}
```

从数组构建实例
```java
public CopyOnWriteArrayList(E[] toCopyIn) {
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}
```

## 四、列表存取

获取数组，返回类型为Object[]。

```java
final Object[] getArray() {
    return array;
}
```

设置数组，形参类型为Object[]。

```java
final void setArray(Object[] a) {
    array = a;
}
```

## 五、基本操作

### 5.1、增加


```java
// 把新元素插入到列表尾
public boolean add(E e) {
    // 修改时获取锁以保证线程安全
    synchronized (lock) {
        // 获取原数组
        Object[] elements = getArray();
        // 获取原数组长度
        int len = elements.length;
        // 用原数组拷贝出新数组，长度增加1个单位
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 在新数组最后位置存入新元素e
        newElements[len] = e;
        // 此时同时存在新、旧两个数组，用新数组引用替换旧数组引用
        setArray(newElements);
        return true;
    }
}

// 把新元素插入到指定索引位置，指定索引原位置元素及后续元素都相对后移一个位置
public void add(int index, E element) {
    // 修改时获取锁以保证线程安全
    synchronized (lock) {
        // 获取原数组
        Object[] elements = getArray();
        // 获取原数组长度
        int len = elements.length;
        // 检查插入位置是否越界
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException(outOfBounds(index, len));
        Object[] newElements;
        // 有多少个元素需要后移
        int numMoved = len - index;
        // numMoved为0表示新元素插入到最后一个索引
        if (numMoved == 0)
            // 创建新数组
            newElements = Arrays.copyOf(elements, len + 1);
        else {
            // 创建新数组
            newElements = new Object[len + 1];
            // 把旧元素赋值到新数组[0, index)
            System.arraycopy(elements, 0, newElements, 0, index);
            // 把旧元素赋值到新数组[index+1, numMoved)
            System.arraycopy(elements, index, newElements, index + 1,
                             numMoved);
        }
        // 在新数组指定位置index插入新元素
        newElements[index] = element;
        // 此时同时存在新、旧两个数组，用新数组引用替换旧数组引用
        setArray(newElements);
    }
}

// 把集合c的所有元素添加到本数组中
public boolean addAll(Collection<? extends E> c) {
    Object[] cs = (c.getClass() == CopyOnWriteArrayList.class) ?
        ((CopyOnWriteArrayList<?>)c).getArray() : c.toArray();
    
    // 没有添加任何元素
    if (cs.length == 0)
        return false;
        
    // 修改时获取锁以保证线程安全
    synchronized (lock) {
        // 获取原数组
        Object[] elements = getArray();
        // 获取原数组长度
        int len = elements.length;
        if (len == 0 && cs.getClass() == Object[].class)
            setArray(cs);
        else {
            // 用原数组构建新数组，长度为原数组长度与集合c长度之和
            Object[] newElements = Arrays.copyOf(elements, len + cs.length);
            // 把集合c的元素拷贝到新数组中
            System.arraycopy(cs, 0, newElements, len, cs.length);
            // 新数组替换旧数组
            setArray(newElements);
        }
        return true; // 完成添加所有新元素
    }
}

// 在原数组index位置开始插入所有集合c的元素
public boolean addAll(int index, Collection<? extends E> c) {
    Object[] cs = c.toArray();
    // 修改时获取锁以保证线程安全
    synchronized (lock) {
        // 原数组
        Object[] elements = getArray();
        // 原数组长度
        int len = elements.length;
        // 检查元素插入索引
        if (index > len || index < 0)
            throw new IndexOutOfBoundsException(outOfBounds(index, len));
        if (cs.length == 0)
            return false;
        // 有多少个元素需要后移
        int numMoved = len - index;
        // 新数组
        Object[] newElements;
        if (numMoved == 0)
            // 用原数组构建新数组，新长度为原数组长度与集合c长度之和
            newElements = Arrays.copyOf(elements, len + cs.length);
        else {
            // 创建新数组
            newElements = new Object[len + cs.length];
            // 拷贝原数组数据到新数组，[0, index)
            System.arraycopy(elements, 0, newElements, 0, index);
            // 拷贝原数组数据到新数组，[index + cs.length, index + cs.length + numMoved)
            System.arraycopy(elements, index,
                             newElements, index + cs.length,
                             numMoved);
        }
        // 拷贝新元素到新数组，[index, index + cs.length)
        System.arraycopy(cs, 0, newElements, index, cs.length);
        // 新数组索引替换旧数组
        setArray(newElements);
        return true;
    }
}

// 若原列表不存在此元素，则存入
public boolean addIfAbsent(E e) {
    // 先拷贝出一份快照
    Object[] snapshot = getArray();
    // 检查快照是否包含此元素，不包含则存入此元素
    return indexOf(e, snapshot, 0, snapshot.length) >= 0 ? false :
        addIfAbsent(e, snapshot);
}

private boolean addIfAbsent(E e, Object[] snapshot) {
    // 修改时获取锁以保证线程安全
    synchronized (lock) {
        // 原数组
        Object[] current = getArray();
        // 获取数组长度
        int len = current.length;
        // 对比快照与原数组是否为同一对象
        if (snapshot != current) {
            // Optimize for lost race to another addXXX operation
            int common = Math.min(snapshot.length, len);
            
            // 依次遍历两者最小长度下的元素
            for (int i = 0; i < common; i++)
                // 检查快照的顺序元素是否和原数组的顺序元素一致
                // 同时检查原数组是否已包含元素e
                if (current[i] != snapshot[i]
                    && Objects.equals(e, current[i]))
                    return false; // 快照与原数组不一致，或原数组已包含元素e
            // 元素已经存在原数组中
            if (indexOf(e, current, common, len) >= 0)
                    return false;
        }
        // 创建新数组
        Object[] newElements = Arrays.copyOf(current, len + 1);
        // 存入新元素
        newElements[len] = e;
        // 新数组引用替换旧数组引用
        setArray(newElements);
        // 添加成功
        return true;
    }
}

// 集合c中不包含在原列表的元素添加到原列表尾部
public int addAllAbsent(Collection<? extends E> c) {
    // c为空抛出空指针异常，cs是c的拷贝
    Object[] cs = c.toArray();
    if (cs.length == 0)
        return 0;
    // 修改时获取锁以保证线程安全
    synchronized (lock) {
        // 获取原数组
        Object[] elements = getArray();
        // 获取原数组长度
        int len = elements.length;
        int added = 0;
        // 依次遍历集合c中的元素
        for (int i = 0; i < cs.length; ++i) {
            Object e = cs[i];
            // 如果集合c的元素e不在elements中
            if (indexOf(e, elements, 0, len) < 0 &&
                indexOf(e, cs, 0, added) < 0)
                cs[added++] = e; // 把元素e放在cs前面
        }
        if (added > 0) {
            // 根据原列表拷贝新数组
            Object[] newElements = Arrays.copyOf(elements, len + added);
            // 把cs中前added个元素尾插入到新数组
            System.arraycopy(cs, 0, newElements, len, added);
            // 新数组替换旧数组
            setArray(newElements);
        }
        return added;
    }
}
```

### 5.2、删除

```java
// 移除数组中指定索引的元素
public E remove(int index) {
    // 修改时获取锁以保证线程安全
    synchronized (lock) {
        // 获取原数组
        Object[] elements = getArray();
        // 获取原数组长度
        int len = elements.length;
        // 获取指定索引元素
        E oldValue = elementAt(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            // 创建新数组
            Object[] newElements = new Object[len - 1];
            // 把旧元素赋值到新数组[0, index)
            System.arraycopy(elements, 0, newElements, 0, index);
            // 把旧元素赋值到新数组[index, numMoved)
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            // 此时同时存在新、旧两个数组，用新数组引用替换旧数组引用
            setArray(newElements);
        }
        return oldValue; // 返回被移除的元素
    }
}

// 移除该元素
public boolean remove(Object o) {
    // 获取原数组
    Object[] snapshot = getArray();
    // 获取指定元素在数组中的索引
    int index = indexOf(o, snapshot, 0, snapshot.length);
    // 成功移除返回true，否则返回false
    return (index < 0) ? false : remove(o, snapshot, index);
}

// 根据快照移除元素
private boolean remove(Object o, Object[] snapshot, int index) {
    // 修改时获取锁以保证线程安全
    synchronized (lock) {
        // 获取原数组
        Object[] current = getArray();
        // 获取原数组长度
        int len = current.length;
        if (snapshot != current) findIndex: {
            int prefix = Math.min(index, len);
            for (int i = 0; i < prefix; i++) {
                if (current[i] != snapshot[i]
                    && Objects.equals(o, current[i])) {
                    index = i;
                    break findIndex;
                }
            }
            if (index >= len)
                return false;
            if (current[index] == o)
                break findIndex;
            index = indexOf(o, current, index, len);
            if (index < 0)
                return false;
        }
        // 除了被移除的元素，其余元素全部复制到新数组中
        Object[] newElements = new Object[len - 1];
        // 拷贝原数组[0, index)元素到新数组[0, index)
        System.arraycopy(current, 0, newElements, 0, index);
        // 拷贝原数组[index+1, len)元素到新数组[index, len-1)
        System.arraycopy(current, index + 1,
                         newElements, index,
                         len - index - 1);
        // 此时同时存在新、旧两个数组，用新数组引用替换旧数组引用
        setArray(newElements);
        return true;
    }
}

// 移除指定范围的元素[fromIndex, toIndex)
void removeRange(int fromIndex, int toIndex) {
    // 修改时获取锁以保证线程安全
    synchronized (lock) {
        // 原数组
        Object[] elements = getArray();
        // 获取数组长度
        int len = elements.length;
        
        // 越界检查
        if (fromIndex < 0 || toIndex > len || toIndex < fromIndex)
            throw new IndexOutOfBoundsException();
        // 计算移除元素后数组的长度
        int newlen = len - (toIndex - fromIndex);
        int numMoved = len - toIndex;
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, newlen));
        else {
            // 用新长度创建新数组
            Object[] newElements = new Object[newlen];
            // 旧数组[0, fromIndex)拷贝元素到新数组
            System.arraycopy(elements, 0, newElements, 0, fromIndex);
            // 旧数组[toIndex, toIndex+numMoved)拷贝元素到新数组
            System.arraycopy(elements, toIndex, newElements,
                             fromIndex, numMoved);
            // 用新数组引用替换旧数组
            setArray(newElements);
        }
    }
}

// 在原数组中移除所有与集合c有关的元素
public boolean removeAll(Collection<?> c) {
    // c不能为空
    Objects.requireNonNull(c);
    return bulkRemove(e -> c.contains(e));
}

// 被removeAll调用
boolean bulkRemove(Predicate<? super E> filter, int i, int end) {
    // assert Thread.holdsLock(lock);
    final Object[] es = getArray();
    // Optimize for initial run of survivors
    for (; i < end && !filter.test(elementAt(es, i)); i++)
        ;
    if (i < end) {
        final int beg = i;
        final long[] deathRow = nBits(end - beg);
        int deleted = 1;
        deathRow[0] = 1L;   // set bit 0
        for (i = beg + 1; i < end; i++)
            if (filter.test(elementAt(es, i))) {
                setBit(deathRow, i - beg);
                deleted++;
            }
        // Did filter reentrantly modify the list?
        if (es != getArray())
            throw new ConcurrentModificationException();
        final Object[] newElts = Arrays.copyOf(es, es.length - deleted);
        int w = beg;
        for (i = beg; i < end; i++)
            if (isClear(deathRow, i - beg))
                newElts[w++] = es[i];
        System.arraycopy(es, i, newElts, w, es.length - i);
        setArray(newElts);
        return true;
    } else {
        if (es != getArray())
            throw new ConcurrentModificationException();
        return false;
    }
}
```


### 5.3、查询

```java
// 获取列表元素的数量
public int size() {
    return getArray().length;
}

// 检查列表是否为空
public boolean isEmpty() {
    return size() == 0;
}

// 获取指定索引的元素，元素来自本数组getArray()
public E get(int index) {
    return elementAt(getArray(), index);
}

// 检查列表是否包含指定对象
public boolean contains(Object o) {
    Object[] elements = getArray();
    return indexOf(o, elements, 0, elements.length) >= 0;
}

// 在[0, fence)的范围内查找元素
private static int indexOf(Object o, Object[] elements,
                           int index, int fence) {
    if (o == null) {
        // 返回数组中第一个内容为null的索引
        for (int i = index; i < fence; i++)
            if (elements[i] == null)
                return i;
    } else {
        // 返回数组中第一个匹配内容的索引
        for (int i = index; i < fence; i++)
            if (o.equals(elements[i]))
                return i;
    }
    return -1;
}

// 在[0, index]的范围内倒序查找元素
private static int lastIndexOf(Object o, Object[] elements, int index) {
    if (o == null) {
        // 返回数组中最后一个内容为null的索引
        for (int i = index; i >= 0; i--)
            if (elements[i] == null)
                return i;
    } else {
        // 返回数组中最后一个匹配内容的索引
        for (int i = index; i >= 0; i--)
            if (o.equals(elements[i]))
                return i;
    }
    return -1;
}

// 获取列表中指定元素的索引值
public int indexOf(Object o) {
    Object[] elements = getArray();
    return indexOf(o, elements, 0, elements.length);
}

// 获取列表中指定元素在index之后的索引值
public int indexOf(E e, int index) {
    Object[] elements = getArray();
    return indexOf(e, elements, index, elements.length);
}

// 获取列表中指定元素最后出现的索引值
public int lastIndexOf(Object o) {
    Object[] elements = getArray();
    return lastIndexOf(o, elements, elements.length - 1);
}

// 获取列表中指定元素在索引之后，且最后出现的索引值
public int lastIndexOf(E e, int index) {
    Object[] elements = getArray();
    return lastIndexOf(e, elements, index);
}

// 检查原列表是否全部包含集合c元素
public boolean containsAll(Collection<?> c) {
    // 原数组
    Object[] elements = getArray();
    // 原数组长度
    int len = elements.length;
    // 逐个确认元素是否包含
    for (Object e : c) {
        if (indexOf(e, elements, 0, len) < 0)
            // 有一个元素不包含就返回false
            return false;
    }
    // 原数组包含所有集合c的元素
    return true;
}
```

### 5.4、修改

```java
// 把指定元素设置到指定索引位置
public E set(int index, E element) {
    // 修改时获取锁以保证线程安全
    synchronized (lock) {
        // 原数组
        Object[] elements = getArray();
        // 获取原数组指定索引位置下的元素
        E oldValue = elementAt(elements, index);
        
        // 旧元素和新元素是否相同
        if (oldValue != element) {
            int len = elements.length;
            // 拷贝出新数组
            Object[] newElements = Arrays.copyOf(elements, len);
            // 把新元素放入新数组
            newElements[index] = element;
            // 新数组引用替换旧数组引用
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics
            setArray(elements);
        }
        return oldValue; // 返回旧元素
    }
}

// 原数组仅保留所有集合c的元素，相当于把两者交集结果作为原数组的新结果
public boolean retainAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return bulkRemove(e -> !c.contains(e));
}
```

## 六、其他

```java
// 根据原数组浅拷贝一份独立新数组，长度一致并保持元素原有顺序，元素类型为Object
public Object[] toArray() {
    Object[] elements = getArray();
    return Arrays.copyOf(elements, elements.length);
}

// 把原数组元素放入数组a中
@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a) {
    // 原数组
    Object[] elements = getArray();
    // 原数组长度
    int len = elements.length;
    // 数组a长度不足以存放所有数组elements的元素
    if (a.length < len)
        // 创建新的数组，类型和数组a一致
        return (T[]) Arrays.copyOf(elements, len, a.getClass()); 
    else {
        // 空间足够，把elements数组的元素放入数组a中
        System.arraycopy(elements, 0, a, 0, len);
        // 如果a还有空闲位置，则把空闲位置置空
        if (a.length > len)
            a[len] = null;
        return a; // 返回结果
    }
}

// 获取指定索引的元素，元素来自传入数组a
@SuppressWarnings("unchecked")
static <E> E elementAt(Object[] a, int index) {
    // 有可能出现数组越界
    return (E) a[index];
}

static String outOfBounds(int index, int size) {
    return "Index: " + index + ", Size: " + size;
}

// 清空列表
public void clear() {
    synchronized (lock) {
        // 申请一个空数组，并作为新结果存入
        setArray(new Object[0]);
    }
}

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

private boolean bulkRemove(Predicate<? super E> filter) {
    synchronized (lock) {
        return bulkRemove(filter, 0, getArray().length);
    }
}

public boolean equals(Object o) {
    if (o == this)
        // 同一个元素
        return true;
    if (!(o instanceof List))
        // 类型不一样，返回false
        return false;

    List<?> list = (List<?>)o;
    Iterator<?> it = list.iterator(); // 获取o的迭代器
    Object[] elements = getArray();
    
    // 逐个对比数组o与原数组的元素
    for (int i = 0, len = elements.length; i < len; i++)
        if (!it.hasNext() || !Objects.equals(elements[i], it.next()))
            return false;

    // 对象o还存在更多元素
    if (it.hasNext())
        return false;

    // 元素完全一致
    return true;
}

// 返回列表所有元素哈希值的总值
public int hashCode() {
    int hashCode = 1;
    for (Object x : getArray())
        // 逐个叠加哈希值
        hashCode = 31 * hashCode + (x == null ? 0 : x.hashCode());
    return hashCode;
}
```

## 七、重置锁

反序列化或克隆后需要重置同步对象

```java
private void resetLock() {
    Field lockField = java.security.AccessController.doPrivileged(
        (java.security.PrivilegedAction<Field>) () -> {
            try {
                // 反射获取lock变量
                Field f = CopyOnWriteArrayList.class
                    .getDeclaredField("lock");
                f.setAccessible(true);
                return f;
            } catch (ReflectiveOperationException e) {
                throw new Error(e);
            }});
    try {
        // 给lock赋对象
        lockField.set(this, new Object());
    } catch (IllegalAccessException e) {
        throw new Error(e);
    }
}
```