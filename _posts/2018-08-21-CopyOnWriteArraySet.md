---
layout:     post
title:      "Java源码系列(15) -- CopyOnWriteArraySet"
date:       2018-08-21
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、类签名

CopyOnWriteArraySet基于[CopyOnWriteArrayList](https://phantomvk.github.io/2018/08/09/CopyOnWriteArrayList/)实现功能，通过检查元素是否包含在列表中，实现集合的功能。来自JDK10。

```java
public class CopyOnWriteArraySet<E> extends AbstractSet<E>
        implements java.io.Serializable
```

## 二、数据成员

由于CopyOnWriteArraySet利用CopyOnWriteArrayList实现功能，所以数据成员中包含了一个CopyOnWriteArrayList实例。

```java
private final CopyOnWriteArrayList<E> al;
```

## 三、构造方法

初始化空集合

```java
public CopyOnWriteArraySet() {
    al = new CopyOnWriteArrayList<E>();
}
```

通过指定集合初始化，集合c为空则抛出NullPointerException

```java
public CopyOnWriteArraySet(Collection<? extends E> c) {
    if (c.getClass() == CopyOnWriteArraySet.class) {
        // 集合c是CopyOnWriteArraySet，里面所有元素都是唯一的
        @SuppressWarnings("unchecked") CopyOnWriteArraySet<E> cc =
            (CopyOnWriteArraySet<E>)c;
        // 直接用集合c构造实例
        al = new CopyOnWriteArrayList<E>(cc.al);
    }
    else {
        // 集合c类型不是CopyOnWriteArraySet，不能保证集合c的元素是唯一的
        al = new CopyOnWriteArrayList<E>();
        // 需通过addAllAbsent()逐个添加元素去重
        al.addAllAbsent(c);
    }
}
```

## 四、成员方法

### 4.1 增加

```java
// 添加指定元素，如果元素已存在，则该元素不会添加
public boolean add(E e) {
    return al.addIfAbsent(e); // 此元素成功添加返回true，否则返回false
}

// 把集合c中所有的元素添加到ArraySet实例中
public boolean addAll(Collection<? extends E> c) {
    return al.addAllAbsent(c) > 0; // 有任何元素添加到
}
```

### 4.2 删除

```java
// 清空Set中所有元素
public void clear() {
    al.clear();
}

// 移除指定元素
public boolean remove(Object o) {
    return al.remove(o);
}

// 移除Set与集合c共有的元素
public boolean removeAll(Collection<?> c) {
    return al.removeAll(c);
}
```

### 4.3 查询

```java
// 查找指定对象是否包含在那本实例
public boolean contains(Object o) {
    return al.contains(o);
}

// 检查Set是否全包含集合c的元素
public boolean containsAll(Collection<?> c) {
    return (c instanceof Set)
        ? compareSets(al.getArray(), (Set<?>) c) >= 0
        : al.containsAll(c);
}

// 检查集合是否为空
public boolean isEmpty() {
    return al.isEmpty();
}
```

### 4.4 修改

```java
// 仅保留Set与集合c共有的元素
public boolean retainAll(Collection<?> c) {
    return al.retainAll(c);
}
```

### 4.5 其他

```java
// 返回集合已保存元素的数量
public int size() {
    return al.size();
}

// 通过ArraySet返回一个数组
public Object[] toArray() {
    return al.toArray();
}

public <T> T[] toArray(T[] a) {
    return al.toArray(a);
}

// 检查snapshot是否为set的超集:
// *  -1: snapshot不是set的超集;
// *  +0: 两个集合包含的元素完全相同;
// *  +1: snapshot是set的超集，即所有set的元素snapshot都包含;
private static int compareSets(Object[] snapshot, Set<?> set) {
    // Uses O(n^2) algorithm, that is only appropriate for small
    // sets, which CopyOnWriteArraySets should be.
    //
    // Optimize up to O(n) if the two sets share a long common prefix,
    // as might happen if one set was created as a copy of the other set.

    final int len = snapshot.length;
    // Mark matched elements to avoid re-checking
    final boolean[] matched = new boolean[len];

    // j is the largest int with matched[i] true for { i | 0 <= i < j }
    int j = 0;
    outer: for (Object x : set) {
        for (int i = j; i < len; i++) {
            if (!matched[i] && Objects.equals(x, snapshot[i])) {
                matched[i] = true;
                if (i == j)
                    do { j++; } while (j < len && matched[j]);
                continue outer;
            }
        }
        return -1;
    }
    return (j == len) ? 0 : 1;
}

// 返回迭代器，遍历的集合是一份快照，且不支持remove()操作
public Iterator<E> iterator() {
    return al.iterator();
}

// 检查对象o和被实例是否为同一个对象，或其中包含的元素完全相同
public boolean equals(Object o) {
    return (o == this)
        || ((o instanceof Set)
            && compareSets(al.getArray(), (Set<?>) o) == 0);
}
```

## 五、Spliterator

```java
public Spliterator<E> spliterator() {
    return Spliterators.spliterator
        (al.getArray(), Spliterator.IMMUTABLE | Spliterator.DISTINCT);
}
```
