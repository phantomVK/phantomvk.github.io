---
layout:     post
title:      "Java源码系列(10) -- Hashtable"
date:       2018-07-02
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、类签名

`Hashtable`是早期的哈希表实现，新版JDK几乎没作修改

```java
public class Hashtable<K,V>
    extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable
```

![Hashtable_UML](/img/java/Hashtable_UML.png)

Hashtable和[HashMap](/2018/06/30/HashMap/)有几个大区别：

- `Hashtable`父类为Dictionary<K,V>，`HashMap`父类为Map<K,V>；
- `HashMap`经过不断优化，性能远远高于`Hashtable`；
- `Hashtable`使用`synchronized`修饰方法保证线程安全，`HashMap`线程不安全；
- `Hashtable`不允许空key或value，但`HashMap`允许；
- `HashTable`使用Enumeration，`HashMap`使用Iterator；
- `HashMap`引入红黑树缓解哈希冲突的影响；

## 二、数据成员

哈希桶

```java
private transient Entry<?,?>[] table;
```

保存元素总数

```java
private transient int count;
```

重哈希阈值

```java
private int threshold;
```

负载因子

```java
private float loadFactor;
```

结构修改次数，或元素添加次数

```java
private transient int modCount = 0;
```

数组最大长度

```java
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

## 三、构造方法

指定初始容量和负载因子

```java
public Hashtable(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal Load: "+loadFactor);

    if (initialCapacity==0)
        initialCapacity = 1;
    this.loadFactor = loadFactor;
    // 通过初始化容量构建哈希桶
    table = new Entry<?,?>[initialCapacity];
    threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
}
```

自定初始容量

```java
public Hashtable(int initialCapacity) {
    this(initialCapacity, 0.75f);
}
```

默认构造方法，初始容量11，初始负载因子0.75

```java
public Hashtable() {
    this(11, 0.75f);
}
```

使用指定Map构造Hashtable

```java
public Hashtable(Map<? extends K, ? extends V> t) {
    // 初始化哈希表
    this(Math.max(2*t.size(), 11), 0.75f);
    // 向Hashtable存入Map
    putAll(t);
}
```

## 四、成员方法

#### 4.1 包含

通过`key`查找对应`Entry`：

```java
public synchronized boolean containsKey(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length; // 选桶

    // 从哈希桶第一个元素开始查找
    for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
        // 匹配哈希值且key完全相同
        if ((e.hash == hash) && e.key.equals(key)) {
            // 找到对应值
            return true;
        }
    }
    
    // 找不到指定key
    return false;
}
```

通过`value`查找对应`Entry`：

```java
public synchronized boolean contains(Object value) {
    // 不支持value为null
    if (value == null) {
        throw new NullPointerException();
    }

    Entry<?,?> tab[] = table;

    // 依次遍历所有哈希桶
    for (int i = tab.length ; i-- > 0 ;) {
        // 依次遍历哈希桶中的链
        for (Entry<?,?> e = tab[i] ; e != null ; e = e.next) {
            // 依次检查Entry.value是否等于在查找的值
            if (e.value.equals(value)) {
                // 找到对应值
                return true;
            }
        }
    }
    
    // 找不到指定value
    return false;
}
```

#### 4.2 获取

获取指定`key`的`value`，没有则返回`null`：

```java
@SuppressWarnings("unchecked")
public synchronized V get(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode(); // 计算key哈希值
    int index = (hash & 0x7FFFFFFF) % tab.length; // 选桶
    
    // 遍历桶中的链表
    for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            // 匹配哈希值则获取对应值
            return (V)e.value;
        }
    }
    
    // 没有匹配到指定key，返回null
    return null;
}
```

获取指定`key`的`value`，没有则返回`defaultValue`：

```java
@Override
public synchronized V getOrDefault(Object key, V defaultValue) {
    V result = get(key);
    return (null == result) ? defaultValue : result;
}
```

#### 4.3 重哈希

扩容并重哈希所有键值

```java
@SuppressWarnings("unchecked")
protected void rehash() {
    // 获取原表的元素数量
    int oldCapacity = table.length;
    Entry<?,?>[] oldMap = table;

    // 上溢检查
    int newCapacity = (oldCapacity << 1) + 1;
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        if (oldCapacity == MAX_ARRAY_SIZE)
            // 已经到达最大值，不能再扩容
            return;
        newCapacity = MAX_ARRAY_SIZE;
    }

    // 构建新哈希桶数组
    Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

    // 修改次数递增
    modCount++;
    // 计算扩容阈值
    threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    // 赋值新哈希表
    table = newMap;
    
    // 哈希桶总数
    for (int i = oldCapacity ; i-- > 0 ;) {
        // 遍历旧表中的哈希桶
        for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
            // 遍历旧表中哈希桶的链表
            Entry<K,V> e = old;
            old = old.next;

            int index = (e.hash & 0x7FFFFFFF) % newCapacity; // 计算新哈希桶索引
            
            // 下面两行通过头插法把节点插入新桶链表中
            e.next = (Entry<K,V>)newMap[index]; 
            newMap[index] = e; 
        }
    }
}
```

#### 4.4 添加

```java
private void addEntry(int hash, K key, V value, int index) {
    Entry<?,?> tab[] = table;
    if (count >= threshold) {
        // 超过阈值大小，哈希表执行扩容
        rehash();

        tab = table;
        // 取键的哈希值
        hash = key.hashCode();
        // 算哈希桶索引
        index = (hash & 0x7FFFFFFF) % tab.length;
    }

    // 获取哈希桶链表首个索引
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>) tab[index];
    // 把Entry放入哈希索引中，头插法
    tab[index] = new Entry<>(hash, key, value, e);
    // 总数递增
    count++;
    // 修改次数递增
    modCount++;
}
```

```java
public synchronized V put(K key, V value) {
    if (value == null) {
        throw new NullPointerException();
    }

    Entry<?,?> tab[] = table;
    // 计算key的哈希值
    int hash = key.hashCode();
    // 计算桶索引
    int index = (hash & 0x7FFFFFFF) % tab.length;
    
    // 获取哈希桶
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            // 临时保存旧值
            V old = entry.value;
            // 新值替换该Entry的旧值
            entry.value = value;
            // 返回旧的value
            return old;
        }
    }
    
    // 不存在该Entry，创建新Entry
    addEntry(hash, key, value, index);
    // 创建新Entry就返回null作为value
    return null;
}
```

递归调用put()

```java
public synchronized void putAll(Map<? extends K, ? extends V> t) {
    for (Map.Entry<? extends K, ? extends V> e : t.entrySet())
        put(e.getKey(), e.getValue());
}
```

仅在key对应Entry不存在时才创建新Entry，否则直接返回旧value

```java
@Override
public synchronized V putIfAbsent(K key, V value) {
    Objects.requireNonNull(value);

    Entry<?,?> tab[] = table;
    // 计算哈希值
    int hash = key.hashCode();
    // 计算哈希桶
    int index = (hash & 0x7FFFFFFF) % tab.length;
    
    // 获取哈希桶
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];
    for (; entry != null; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            // 从拉链匹配对应Entry
            V old = entry.value;
            // 旧value为null，直接放入
            if (old == null) {
                entry.value = value;
            }
            // 直接返回oldValue
            return old;
        }
    }
    
    // 没有旧的Entry，创建新的并存入
    addEntry(hash, key, value, index);
    return null;
}
```

#### 4.5 清除

```java
public synchronized V remove(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    // 计算哈希桶索引
    int index = (hash & 0x7FFFFFFF) % tab.length;
    
    // 获取哈希桶
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>)tab[index]; 
    for(Entry<K,V> prev = null ; e != null ; prev = e, e = e.next) {
        // 哈希值相同，且key相同
        if ((e.hash == hash) && e.key.equals(key)) {
            // 节点从链表中解除引用
            if (prev != null) {
                // 中间节点解除引用
                prev.next = e.next;
            } else {
                // 头结点解除引用
                tab[index] = e.next;
            }
            // 修改递增
            modCount++;
            // 元素数量递减
            count--;
            // 获取旧值
            V oldValue = e.value;
            // 置空回收
            e.value = null;
            return oldValue;
        }
    }
    // key匹配不到Entry
    return null;
}
```

参考删除方法，此方法只是增加检查value是否一致，仅一致时移除

```java
@Override
public synchronized boolean remove(Object key, Object value) {
    Objects.requireNonNull(value);

    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    // 计算哈希桶索引
    int index = (hash & 0x7FFFFFFF) % tab.length;

    // 获取哈希桶
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>)tab[index]; 
    for (Entry<K,V> prev = null; e != null; prev = e, e = e.next) {
        // key和value都匹配的Entry
        if ((e.hash == hash) && e.key.equals(key) && e.value.equals(value)) {
            if (prev != null) {
                // 从链中摘除节点
                prev.next = e.next;
            } else {
                // 从链头摘除节点
                tab[index] = e.next;
            }
            // 置空以便GC
            e.value = null;
            // 修改递增
            modCount++;
            // 元素数量递减
            count--;
            // 匹配Entry成功
            return true;
        }
    }
    // 匹配Entry失败
    return false;
}
```

清空哈希表所有元素，桶表长度不改变

```java
public synchronized void clear() {
    Entry<?,?> tab[] = table;
    
    // 直接把链表从哈希桶上解除关系
    for (int index = tab.length; --index >= 0; )
        tab[index] = null;
    modCount++;
    // 保存元素总数置0
    count = 0;
}
```

#### 4.6 比较

```java
public synchronized boolean equals(Object o) {
    if (o == this)
        // 同一个HashTable
        return true;

    if (!(o instanceof Map))
        // 对象o没有继承Map，返回false
        return false;

    Map<?,?> t = (Map<?,?>) o;

    // 表中保存元素数量不同，不是同一个表
    if (t.size() != size())
        return false;

    try {
        // 逐个对比HashTable和对象o的元素，任何元素不同即判定不是同一个Map
        for (Map.Entry<K, V> e : entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            if (value == null) {
                if (!(t.get(key) == null && t.containsKey(key)))
                    return false;
            } else {
                if (!value.equals(t.get(key)))
                    return false;
            }
        }
    } catch (ClassCastException unused)   {
        return false;
    } catch (NullPointerException unused) {
        return false;
    }

    return true;
}
```

计算HashTable的哈希值

```java
public synchronized int hashCode() {
    int h = 0;
    if (count == 0 || loadFactor < 0)
        // 返回0
        return h;

    loadFactor = -loadFactor;
    Entry<?,?>[] tab = table;
    for (Entry<?,?> entry : tab) {
        while (entry != null) {
            // 累加Entry实例的哈希值
            h += entry.hashCode();
            entry = entry.next;
        }
    }

    loadFactor = -loadFactor;

    return h;
}
```

#### 4.7 替换

用key查找对应Entry，且匹配oldValue，替换oldValue为value

```java
@Override
public synchronized boolean replace(K key, V oldValue, V newValue) {
    // oldValue和newValue均不能为空
    Objects.requireNonNull(oldValue);
    Objects.requireNonNull(newValue);

    Entry<?,?> tab[] = table;
    // 计算key的哈希值
    int hash = key.hashCode();
    // 哈希桶索引
    int index = (hash & 0x7FFFFFFF) % tab.length; 

    // 获取哈希桶
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>)tab[index];
    for (; e != null; e = e.next) {
        // 查找是否有key对应的Entry
        if ((e.hash == hash) && e.key.equals(key)) {
            if (e.value.equals(oldValue)) {
                // 同时还要和oldValue匹配，才能替换为newValue
                e.value = newValue;
                // 替换成功
                return true;
            } else {
                // 替换失败
                return false;
            }
        }
    }
    return false;
}
```

用key查找对应Entry，并替换oldValue为value

```java
@Override
public synchronized V replace(K key, V value) {
    Objects.requireNonNull(value);

    Entry<?,?> tab[] = table;
    // 计算key的哈希值
    int hash = key.hashCode();
    // 计算哈希索引大概位置
    int index = (hash & 0x7FFFFFFF) % tab.length;

    // 选取哈希桶
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>)tab[index];
    for (; e != null; e = e.next) {
        // 逐个遍历链表元素，查找是否有对应Entry
        if ((e.hash == hash) && e.key.equals(key)) {
            V oldValue = e.value;
            e.value = value;
            // 返回旧值
            return oldValue;
        }
    }
    
    // 没有对应key，返回null
    return null;
}
```

## 五、节点

```java
private static class Entry<K,V> implements Map.Entry<K,V> {
    final int hash; // 哈希值
    final K key; // 键
    V value;     // 值
    Entry<K,V> next; // 下一项的引用
    
    // 构造方法
    protected Entry(int hash, K key, V value, Entry<K,V> next) {
        this.hash = hash;
        this.key =  key;
        this.value = value;
        this.next = next;
    }
    
    // 克隆
    @SuppressWarnings("unchecked")
    protected Object clone() {
        return new Entry<>(hash, key, value,
                              (next==null ? null : (Entry<K,V>) next.clone()));
    }

    public K getKey() {
        return key;
    }

    public V getValue() {
        return value;
    }

    public V setValue(V value) {
        if (value == null)
            throw new NullPointerException();
            
        // 设置新值并返回新值
        V oldValue = this.value;
        this.value = value;
        return oldValue;
    }

    public boolean equals(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>)o;

        return (key==null ? e.getKey()==null : key.equals(e.getKey())) &&
           (value==null ? e.getValue()==null : value.equals(e.getValue()));
    }

    public int hashCode() {
        return hash ^ Objects.hashCode(value);
    }

    public String toString() {
        return key.toString()+"="+value.toString();
    }
}
```