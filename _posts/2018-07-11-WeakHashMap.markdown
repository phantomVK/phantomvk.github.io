---
layout:     post
title:      "Java源码系列(12) -- WeakHashMap"
date:       2018-07-11
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、类签名

WeakHashMap的元素`Entry`继承自WeakReference，当元素没有外部引用被回收，或因虚拟机内存不足而回收，元素会被放入到`queue`。虽然WeakHashMap和[HashMap](http://phantomvk.github.io/2018/06/30/HashMap/)拥有相同父类，但在具体实现上HashMap有更好的优化，和WeakHashMap更相似反倒是[HashTable](http://phantomvk.github.io/2018/07/02/HashTable/)。

```java
public class WeakHashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V> 
```

UML:

![WeakHashMap_UML](/img/java/WeakHashMap_UML.png)

## 二、数据成员

```java
// 默认初始化容量
private static final int DEFAULT_INITIAL_CAPACITY = 16;

// 保存键值对最大容量值
private static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认负载因子
private static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 代表tables里的null_key
private static final Object NULL_KEY = new Object();

// 哈希表，长度必须为2的幂
Entry<K,V>[] table;

// 已保存键值对数量
private int size;

// 扩容阈值：threshold = capacity * load factor
private int threshold;

// 实际负载因子
private final float loadFactor;

//  保存已经清理WeakEntries的引用队列
private final ReferenceQueue<Object> queue = new ReferenceQueue<>();

// 修改次数
int modCount;
```

## 三、构造方法

```java
public WeakHashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Initial Capacity: "+
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;

    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal Load factor: "+
                                           loadFactor);
    int capacity = 1;
    while (capacity < initialCapacity)
        capacity <<= 1;
    table = newTable(capacity);
    this.loadFactor = loadFactor;
    threshold = (int)(capacity * loadFactor);
}

public WeakHashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public WeakHashMap() {
    this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
}

public WeakHashMap(Map<? extends K, ? extends V> m) {
    this(Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1,
            DEFAULT_INITIAL_CAPACITY),
         DEFAULT_LOAD_FACTOR);
    putAll(m);
}

// 构造方法中使用，用于创建Entry数组的方法
@SuppressWarnings("unchecked")
private Entry<K,V>[] newTable(int n) {
    return (Entry<K,V>[]) new Entry<?,?>[n];
}
```

## 四、成员方法

```java
// 检查是否为空，为空使用NULL_KEY表示
private static Object maskNull(Object key) {
    return (key == null) ? NULL_KEY : key;
}

// 把NULL_KEY通过null表示
static Object unmaskNull(Object key) {
    return (key == NULL_KEY) ? null : key;
}

// 对象对比
private static boolean eq(Object x, Object y) {
    return x == y || x.equals(y);
}

// 计算key的哈希值
final int hash(Object k) {
    int h = k.hashCode();

    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

// 根据哈希值和表长度算出索引位置
private static int indexFor(int h, int length) {
    return h & (length-1);
}

// 清理废弃的Entry
private void expungeStaleEntries() {
    // 依次遍历队列的元素，里面保存的全是已经废弃的元素
    for (Object x; (x = queue.poll()) != null; ) {
        synchronized (queue) {
            @SuppressWarnings("unchecked")
            Entry<K,V> e = (Entry<K,V>) x; // 类型转换

            int i = indexFor(e.hash, table.length); // 通过Entry的哈希值选桶

            Entry<K,V> prev = table[i]; // 获取哈希桶的首个元素
            Entry<K,V> p = prev;
            while (p != null) {
                Entry<K,V> next = p.next;
                if (p == e) {
                    if (prev == e)
                        table[i] = next; // e是头结点，解除链接
                    else
                        prev.next = next; // e是中间节点，解除链接

                    // 不能把e.next置null，因为废弃的元素可能还在被HashIterator使用
                    e.value = null; // 把e的value置空
                    size--; // 保存元素数量递减
                    break;
                }
                prev = p;
                p = next;
            }
        }
    }
}

// 清理废弃Entry后返回table
private Entry<K,V>[] getTable() {
    expungeStaleEntries();
    return table;
}

// 清理废弃Entry后返回已保存Entry数量
public int size() {
    if (size == 0)
        return 0;
    expungeStaleEntries();
    return size;
}

// 把所有src表的节点移动到dest表里
private void transfer(Entry<K,V>[] src, Entry<K,V>[] dest) {
    // 遍历src表的哈希桶索引
    for (int j = 0; j < src.length; ++j) {
        Entry<K,V> e = src[j]; // 获取哈希桶索引
        src[j] = null; // 把当前处理的链表从src解链接
        while (e != null) {
            Entry<K,V> next = e.next;
            Object key = e.get();
            if (key == null) {
                e.next = null;  // 帮助GC
                e.value = null;
                size--;
            } else {
                int i = indexFor(e.hash, dest.length);
                e.next = dest[i]; // 头插法放入dest表中
                dest[i] = e;
            }
            e = next;
        }
    }
}
```
## 五、获取
```java
public V get(Object key) {
    Object k = maskNull(key);
    int h = hash(k); // 计算key的哈希值
    Entry<K,V>[] tab = getTable(); // 获取表
    int index = indexFor(h, tab.length); // 获取哈希桶索引
    Entry<K,V> e = tab[index]; // 获取哈希桶
    while (e != null) {
        if (e.hash == h && eq(k, e.get())) // 匹配对应key
            return e.value; // 返回Entry.value
        e = e.next;
    }
    return null;
}

Entry<K,V> getEntry(Object key) {
    Object k = maskNull(key);
    int h = hash(k); // 计算key的哈希值
    Entry<K,V>[] tab = getTable(); // 获取表
    int index = indexFor(h, tab.length); // 获取哈希桶索引
    Entry<K,V> e = tab[index]; // 获取哈希桶
    // 循环查找，直到匹配对应key
    while (e != null && !(e.hash == h && eq(k, e.get())))
        e = e.next;
    return e;
}
```
## 六、存入

```java
public V put(K key, V value) {
    Object k = maskNull(key);
    int h = hash(k); // 计算key的哈希值
    Entry<K,V>[] tab = getTable(); // 获取表
    int i = indexFor(h, tab.length); // 计算桶索引
    
    // 依次遍历桶里链表的元素
    for (Entry<K,V> e = tab[i]; e != null; e = e.next) {
        // 匹配到已有Entry
        if (h == e.hash && eq(k, e.get())) {
            // 用newValue替换oldValue
            V oldValue = e.value;
            if (value != oldValue)
                e.value = value;
            return oldValue; // 返回oldValue
        }
    }
    
    // 不存在以后元素，执行下面的逻辑
    modCount++; // 修改次数递增
    Entry<K,V> e = tab[i]; // 获取哈希桶
    tab[i] = new Entry<>(k, value, queue, h, e); // 创建新Entry，通过头插法的方式放入链表
    if (++size >= threshold) // 如果已保存元素数量超过阈值，触发重哈希逻辑
        resize(tab.length * 2);

    return null; // 创建新的Entry会返回null，因为不存在oldValue
}

public void putAll(Map<? extends K, ? extends V> m) {
    int numKeysToBeAdded = m.size();
    if (numKeysToBeAdded == 0)
        return;

    if (numKeysToBeAdded > threshold) {
        int targetCapacity = (int)(numKeysToBeAdded / loadFactor + 1);
        
        // 保存键值对数量不能超过最大值
        if (targetCapacity > MAXIMUM_CAPACITY)
            targetCapacity = MAXIMUM_CAPACITY;
        
        int newCapacity = table.length;
        while (newCapacity < targetCapacity)
            newCapacity <<= 1;
        if (newCapacity > table.length)
            resize(newCapacity);
    }

    // 递归调用put(K key, V value)
    for (Map.Entry<? extends K, ? extends V> e : m.entrySet())
        put(e.getKey(), e.getValue());
}

void resize(int newCapacity) {
    Entry<K,V>[] oldTable = getTable();
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry<K,V>[] newTable = newTable(newCapacity); // 构建新哈希表
    transfer(oldTable, newTable); // 把旧哈希表的元素放入新哈希表
    table = newTable;

    if (size >= threshold / 2) {
        threshold = (int)(newCapacity * loadFactor);
    } else {
        expungeStaleEntries();
        transfer(newTable, oldTable);
        table = oldTable;
    }
}
```
## 七、移除
```java
// 根据key移除Entry
public V remove(Object key) {
    Object k = maskNull(key);
    int h = hash(k); // 用key计算hash
    Entry<K,V>[] tab = getTable();
    int i = indexFor(h, tab.length); // 用计算出来的hash选桶索引
    Entry<K,V> prev = tab[i]; // 用桶索引获取对应哈希桶
    Entry<K,V> e = prev;

    while (e != null) {
        Entry<K,V> next = e.next;
        // 匹配对应key，把Entry从链中移除
        if (h == e.hash && eq(k, e.get())) {
            modCount++; // 修改次数递增
            size--; // 已保存元素数量递减
            if (prev == e)
                tab[i] = next; // 被移除的是链表头节点
            else
                prev.next = next; // 被移除的是表中节点
            return e.value; // 返回被移除节点的value
        }
        prev = e;
        e = next;
    }

    return null; // 没有移除任何节点，返回null
}

boolean removeMapping(Object o) {
    if (!(o instanceof Map.Entry))
        return false;
    Entry<K,V>[] tab = getTable();
    Map.Entry<?,?> entry = (Map.Entry<?,?>)o; // 类型转换
    Object k = maskNull(entry.getKey());
    int h = hash(k); // 用key计算hash
    int i = indexFor(h, tab.length); // 用计算出来的hash选桶索引
    Entry<K,V> prev = tab[i]; // 用桶索引获取对应哈希桶
    Entry<K,V> e = prev;

    while (e != null) {
        Entry<K,V> next = e.next;
        // 匹配对应key，把Entry从链中移除
        if (h == e.hash && e.equals(entry)) {
            modCount++; // 修改次数递增
            size--; // 已保存元素数量递减
            if (prev == e)
                tab[i] = next; // 被移除的是链表头节点
            else
                prev.next = next; // 被移除的是表中节点
            return true; // 移除成功
        }
        prev = e;
        e = next;
    }

    return false; // 没有移除元素
}

// 清除所有键值对
public void clear() {
    // 队列元素全部出队
    while (queue.poll() != null)
        ;

    modCount++; // 修改次数递增
    Arrays.fill(table, null); // 把哈希桶置null
    size = 0; // 以保存元素数量置0

    // 队列元素全部出队
    while (queue.poll() != null)
        ;
}
```

## 八、包含

```java
public boolean containsValue(Object value) {
    // 查找的value为null，调用containsNullValue()
    if (value==null)
        return containsNullValue();

    Entry<K,V>[] tab = getTable();

    // 遍历表的所有哈希桶
    for (int i = tab.length; i-- > 0;)
        // 遍历哈希桶中所有Entry
        for (Entry<K,V> e = tab[i]; e != null; e = e.next)
            // 检查该Entry的value是否匹配目标value
            if (value.equals(e.value))
                return true;

    return false; // 所有Entry都没有包含该value
}

// 检查所有Entry是否包含null的value
private boolean containsNullValue() {
    Entry<K,V>[] tab = getTable();

    // 遍历表的所有哈希桶
    for (int i = tab.length; i-- > 0;)
        // 遍历哈希桶中所有Entry
        for (Entry<K,V> e = tab[i]; e != null; e = e.next)
            // 如果发现Entry的存在value为null，返回true
            if (e.value==null)
                return true;
  
    return false; // 没有Entry的value为null
}
```

## 九、节点

```java
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
    V value;
    final int hash;
    Entry<K,V> next;

    Entry(Object key, V value,
          ReferenceQueue<Object> queue,
          int hash, Entry<K,V> next) {
        super(key, queue);
        this.value = value;
        this.hash  = hash;
        this.next  = next;
    }

    @SuppressWarnings("unchecked")
    public K getKey() {
        return (K) WeakHashMap.unmaskNull(get());
    }

    public V getValue() {
        return value;
    }

    public V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public boolean equals(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>)o;
        K k1 = getKey();
        Object k2 = e.getKey();
        if (k1 == k2 || (k1 != null && k1.equals(k2))) {
            V v1 = getValue();
            Object v2 = e.getValue();
            if (v1 == v2 || (v1 != null && v1.equals(v2)))
                return true;
        }
        return false;
    }

    public int hashCode() {
        K k = getKey();
        V v = getValue();
        return Objects.hashCode(k) ^ Objects.hashCode(v);
    }

    public String toString() {
        return getKey() + "=" + getValue();
    }
}
```