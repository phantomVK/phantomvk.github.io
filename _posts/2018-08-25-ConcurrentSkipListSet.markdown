---
layout:     post
title:      "Java源码系列(16) -- ConcurrentSkipListSet"
date:       2018-08-25
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---
## 一、类签名

ConcurrentSkipListSet是一个基于ConcurrentSkipListMap的，可扩展的并发(NavigableSet)实现。所有元素根据其可比较的自然排序，或构造时提供的Comparator决定排列顺序，具体由所使用的构造方法决定。

```java
public class ConcurrentSkipListSet<E>
    extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable
```

此实现在contains、add、remove类似操作上预计平均时间开销为`log(n)`。插入、移除、访问操作在多线程下操作都是线程安全的(注：保证最终一致性)。

递增排序视图和其迭代器都比递减方式的速度要快。

需要注意的是，不像大多数集合，size操作并不是一个常数时间的操作。由于集合的异步特性，当前元素的数量由需元素的遍历决定。所以，在遍历过程中集合的元素也在改变，会返回一个不准确的元素数量值。

批量操作如添加、移除、检查多个元素，譬如addAll、removeIf、forEach等都不保证其原子性。就像forEach遍历和另一个线程的addAll并发执行，可能会导致只有一部分新元素能被forEach察觉。

此类和其迭代器实现了Set和Iterator接口的可选方法。像其他多数并发集合实现，此类不允许使用为null的元素。因为数据值null和返回值，不能与缺失元素的null做出区分。（备注：即返回值获取null，表明元素不存在，而不表示元素的值为null）

源码基于JDK10。

## 二、数据成员

ConcurrentSkipListSet基于ConcurrentNavigableMap实现集合去重的特性。由于ConcurrentNavigableMap是<K, V>的，所以使用Boolean.TRUE作为元素的值，表示已存入的元素已存在。此变量被修饰为final是为了线程安全，但同时也导致clone()方法实现的一点丑陋。

```java
private final ConcurrentNavigableMap<E,Object> m;
```

## 三、构造方法

构造一个全新的空集合，元素顺序由其可比较自然次序决定

```java
public ConcurrentSkipListSet() {
    m = new ConcurrentSkipListMap<E,Object>();
}
```

构造一个全新的空集合，元素顺序由其传入的比较器决定。若比较器为空，则默认使用可比较自然次序。

```java
public ConcurrentSkipListSet(Comparator<? super E> comparator) {
    m = new ConcurrentSkipListMap<E,Object>(comparator);
}
```

指定数据集构造一个新集合，元素顺序由其可比较自然次序决定。

```java
public ConcurrentSkipListSet(Collection<? extends E> c) {
    m = new ConcurrentSkipListMap<E,Object>();
    addAll(c);
}
```

构造一个全新的集合，元素顺序由SortedSet里的comparator决定。存入元素的顺序和SortedSet元素的顺序一致。

```java
public ConcurrentSkipListSet(SortedSet<E> s) {
    m = new ConcurrentSkipListMap<E,Object>(s.comparator());
    addAll(s);
}
```

此构造方法由submaps使用

```java
ConcurrentSkipListSet(ConcurrentNavigableMap<E,Object> m) {
    this.m = m;
}
```

## 四、集合操作

返回集合元素数量，如果集合元素数量超过Integer.MAX_VALUE，则返回Integer.MAX_VALUE。

需要注意的是，不像其他多数集合的实现，此方法不是常数时间的操作。因其异步特性，统计元素数量需要遍历集合的所有元素。而且，集合元素实际数量有可能在遍历的过程中发生变化，这会导致最终返回值不准确。因此，此方法在并发应用中的实用性并不高。

由于每次获取总数需要遍历所有元素，所以即使在多线程操作下，也应尽量减少调用此方法。

```java
public int size() {
    return m.size();
}
```

检查集合是否包含指定元素

```java
public boolean isEmpty() {
    return m.isEmpty();
}
```

检查指定元素是否已包含在集合中。元素o为空抛出NullPointerException；从集合用key获取的元素类型转换失败抛出ClassCastException。

```java
public boolean contains(Object o) {
    return m.containsKey(o);
}
```

把指定元素添加到集合中。元素不存在于集合，在添加成功返回true。如果元素已经存在于集合中，原集合将保持不变且返回false

```java
public boolean add(E e) {
    return m.putIfAbsent(e, Boolean.TRUE) == null;
}
```

在指定元素存在于本集合的时候，移除集合匹配元素。移除成功返回true，否则返回false

```java
public boolean remove(Object o) {
    return m.remove(o, Boolean.TRUE);
}
```

清空集合所有元素

```java
public void clear() {
    m.clear();
}
```

返回基于此集合元素的升序迭代器

```java
public Iterator<E> iterator() {
    return m.navigableKeySet().iterator();
}
```

返回基于此集合元素的降序迭代器

```java
public Iterator<E> descendingIterator() {
    return m.descendingKeySet().iterator();
}
```

## 五、AbstractSet重载

检查两个集合的元素是否完全一致。重写此方法，避免父类实现调用此类的size()方法。上文已经提到，size()方法在并发的场景下返回的数值不一定准确。

```java
public boolean equals(Object o) {
    // 两者是相同实例，返回true
    if (o == this)
        return true;
    // o不是集合类，返回false
    if (!(o instanceof Set))
        return false;
    // 检查两个集合的元素是否完全一致
    Collection<?> c = (Collection<?>) o;
    try {
        return containsAll(c) && c.containsAll(this);
    } catch (ClassCastException unused) {
        return false;
    } catch (NullPointerException unused) {
        return false;
    }
}
```

移除集合c中包含的元素。重写此方法，避免父类实现调用此类的size()方法。上文已经提到，size()方法在并发的场景下返回的数值不一定准确。
```java
public boolean removeAll(Collection<?> c) {
    boolean modified = false;
    for (Object e : c)
        if (remove(e))
            modified = true;
    return modified;
}
```

## 六、相关操作

```java
// 类型转换失败抛出ClassCastException
// 元素e为空抛出NullPointerException
public E lower(E e) {
    return m.lowerKey(e);
}

// 类型转换失败抛出ClassCastException
// 元素e为空抛出NullPointerException
public E floor(E e) {
    return m.floorKey(e);
}

// 类型转换失败抛出ClassCastException
// 元素e为空抛出NullPointerException
public E ceiling(E e) {
    return m.ceilingKey(e);
}

// 类型转换失败抛出ClassCastException
// 元素e为空抛出NullPointerException
public E higher(E e) {
    return m.higherKey(e);
}

public E pollFirst() {
    Map.Entry<E,Object> e = m.pollFirstEntry();
    return (e == null) ? null : e.getKey();
}

public E pollLast() {
    Map.Entry<E,Object> e = m.pollLastEntry();
    return (e == null) ? null : e.getKey();
}

```

## 七、SortedSet操作

返回集合拥有的比较器

```java
public Comparator<? super E> comparator() {
    return m.comparator();
}
```

没有此元素抛出NoSuchElementException

```java
public E first() {
    return m.firstKey();
}
```

没有此元素抛出NoSuchElementException

```java
public E last() {
    return m.lastKey();
}
```


从集合中获取的元素无法转换为集合元素指定类型抛出ClassCastException。若fromElement或toElement为null抛出NullPointerException。
参数错误抛出IllegalArgumentException；

```java
public NavigableSet<E> subSet(E fromElement,
                              boolean fromInclusive,
                              E toElement,
                              boolean toInclusive) {
    return new ConcurrentSkipListSet<E>
        (m.subMap(fromElement, fromInclusive,
                  toElement,   toInclusive));
}
```

从集合中获取的元素无法转换为集合元素指定类型抛出ClassCastException。若toElement为null抛出NullPointerException。参数错误抛出IllegalArgumentException；

```java
public NavigableSet<E> headSet(E toElement, boolean inclusive) {
    return new ConcurrentSkipListSet<E>(m.headMap(toElement, inclusive));
}
```

从集合中获取的元素无法转换为集合元素指定类型抛出ClassCastException。若fromElement为null抛出NullPointerException。参数错误抛出IllegalArgumentException

```java
public NavigableSet<E> tailSet(E fromElement, boolean inclusive) {
    return new ConcurrentSkipListSet<E>(m.tailMap(fromElement, inclusive));
}
```

从集合中获取的元素无法转换为集合元素指定类型抛出ClassCastException。若fromElement或toElement为null抛出NullPointerException。参数错误抛出IllegalArgumentException

```java
public NavigableSet<E> subSet(E fromElement, E toElement) {
    return subSet(fromElement, true, toElement, false);
}
```

从集合中获取的元素无法转换为集合元素指定类型抛出ClassCastException。若toElement为null抛出NullPointerException。参数错误抛出IllegalArgumentException

```java
public NavigableSet<E> headSet(E toElement) {
    return headSet(toElement, false);
}
```

从集合中获取的元素无法转换为集合元素指定类型抛出ClassCastException。若fromElement为null抛出NullPointerException。参数错误抛出IllegalArgumentException

```java
public NavigableSet<E> tailSet(E fromElement) {
    return tailSet(fromElement, true);
}
```

返回倒序排列的集合。由于新ConcurrentSkipListSet数据是基于原ConcurrentSkipListSet实例的。修改新ConcurrentSkipListSet会影响原ConcurrentSkipListSet的数据，反之亦然

```java
public NavigableSet<E> descendingSet() {
    return new ConcurrentSkipListSet<E>(m.descendingMap());
}
```

spliterator

```java
public Spliterator<E> spliterator() {
    return (m instanceof ConcurrentSkipListMap)
        ? ((ConcurrentSkipListMap<E,?>)m).keySpliterator()
        : ((ConcurrentSkipListMap.SubMap<E,?>)m).new SubMapKeyIterator();
}
```

## 八、克隆

返回ConcurrentSkipListSet实例中元素的浅拷贝

```java
public ConcurrentSkipListSet<E> clone() {
    try {
        @SuppressWarnings("unchecked")
        ConcurrentSkipListSet<E> clone =
            (ConcurrentSkipListSet<E>) super.clone();
        clone.setMap(new ConcurrentSkipListMap<E,Object>(m));
        return clone;
    } catch (CloneNotSupportedException e) {
        throw new InternalError();
    }
}
```

clone()方法专用

```java
private void setMap(ConcurrentNavigableMap<E,Object> map) {
    Field mapField = java.security.AccessController.doPrivileged(
        (java.security.PrivilegedAction<Field>) () -> {
            try {
                // 反射获取克隆后实例的变量m
                Field f = ConcurrentSkipListSet.class
                    .getDeclaredField("m");
                f.setAccessible(true);
                return f;
            } catch (ReflectiveOperationException e) {
                throw new Error(e);
            }});
    try {
        mapField.set(this, map);
    } catch (IllegalAccessException e) {
        throw new Error(e);
    }
}
```