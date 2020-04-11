---


layout:     post
title:      "Java源码系列(24) -- HashMap死循环原理"
date:       2019-09-21
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、现象

无论哪个JDK实现 __HashMap__，都不是线程安全。但开发者水平有限，使用 __JDK7 HashMap__ 在多线程操作，或者别人在无意识下把含有 __HashMap__ 的方法在多线程中调用，都将导致不可预测的问题。

由于服务端多线程并发的原因，调用 __HashMap.get(K key)__ 的时候，会偶发处理器100%占用现象。

除了在服务端，**Android** 开发也偶尔需要 **Map** 上的多线程操作。因为移动端开发人员水平参差不齐，并不知道使用 __ConcurrentHashMap__ 或其他线程保护，所以出现这个问题的状况也不罕见。

## 二、起因

上述两者 __HashMap__ 死循环的场景，正出现在多线程同时插入新元素的过程。

为了缓解这个问题，__JDK8__ 对 __HashMap__ 进行调整，优化 __HashMap__ 在多线程出现死循环的场景，把元素迁移的头插法改为尾插法。

不过，该类依然不是线程安全，且需要遍历桶内所有节点才能插入新元素，引入开销。

## 三、源码实现

下面先看JDK7基本代码，这是插入新元素的公开方法：

```java
public V put(K key, V value) {
    if (key == null)
        return putForNullKey(value);
    // 获取键的哈希值
    int hash = hash(key.hashCode());
    // 计算键哈希值对应桶
    int i = indexFor(hash, table.length);
    // 开始从选定哈希桶查找目标元素
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        // 如果该键对应值已存在，则替换并返回该旧值
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    // 键不存在则创建新的节点
    addEntry(hash, key, value, i);
    return null;
}
```

创建新节点，从以下源码可知是链表的头插法

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    // 获取已有链表头节点
    Entry<K,V> e = table[bucketIndex];
    // 创建新节点，并把上述链表头结点作为后续元素
    // 然后自己作为该哈希表的头节点
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    // 由于元素数量超过阈值，所以触发扩容
    if (size++ >= threshold)
        resize(2 * table.length);
}
```

开始扩容操作

```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    // 新哈希表已经创建
    Entry[] newTable = new Entry[newCapacity];
    // 旧哈希桶表元素迁移到新哈希表
    transfer(newTable);
    // 新哈希表替换旧表
    table = newTable;
    threshold = (int)(newCapacity * loadFactor);
}
```

迁移元素

```java
void transfer(Entry[] newTable) {
    // src代表旧哈希表数组
    Entry[] src = table;
    int newCapacity = newTable.length;
    // 遍历旧哈希表逐个迁移表内元素
    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];
        if (e != null) {
            src[j] = null;
            do {
                // 此代码块内是从旧哈希表元素迁移到新哈希表操作
                Entry<K,V> next = e.next;
                // 在新哈希表选择对应
                int i = indexFor(e.hash, newCapacity);
                // 下面3行代码就是链表头插法操作
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            } while (e != null);
        }
    }
}
```

## 四、图解流程

为了方便描述，基本条件假设为：

- __原哈希表桶数量4__、**负载因子0.75**、**扩容阈值3**、只有 __元素c__；
- __线程2__ 把 **元素b** 头插入旧哈希表，__线程1__ 把 **元素a** 头插入旧哈希表；
- 上述3个元素的哈希值，假设无论重哈希前还是之后，都会运算到同一个桶；
- 插入完成后旧哈希表 **桶索引为3** 元素顺序为 __元素a->元素b->元素c__；
- 两个线程根据系统调度，分别检查扩容阈值并各自触发扩容操作；

由于只用文字表述比较晦涩，所以加入下面图解有助理解。

两个线程插入完成后旧哈希表，在桶索引为3的元素顺序为 __元素a->元素b->元素c__

![oldmap](/img/java/JDK7_HashMap/oldmap.png)

两个线程写入哈希表后达到阈值后，各自开始扩容操作并创建新哈希桶。

![new_arrays](/img/java/JDK7_HashMap/new_arrays.png)

**线程2** 进入 __transfer(Entry[] newTable)__：

- 迁移元素到新哈希桶，操作执行到注释所在位置，处理器时间片用完被暂停；
- 此时对 线程**2** 来说，__引用e__ 指向 __元素a__，__引用e.next__ 指向 __元素b__；
- 时间片分配给 __线程1__ 并继续执行；

```java
void transfer(Entry[] newTable) {
    Entry[] src = table;
    int newCapacity = newTable.length;
    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];
        if (e != null) {
            src[j] = null;
            do {
                Entry<K,V> next = e.next;
                // 线程2首次进入循环且上述代码已执行完成，暂停在此位置
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            } while (e != null);
        }
    }
}
```

**线程1** 也开始以上操作，并把旧哈希桶的 **元素a** 通过头插法，插入到 **线程1** 创建的新哈希桶内：

![element_1](/img/java/JDK7_HashMap/element_1.png)

**线程1** 迁移 **元素b** 到自己的哈希桶内：

![element_2](/img/java/JDK7_HashMap/element_2.png)

**线程1** 头插法迁移 **元素c** 到哈希桶内，最终 **线程1** 哈希桶索引7链表顺序为 `c->b->a`。

![element_3](/img/java/JDK7_HashMap/element_3.png)

这个时候，**线程1** 处理器时间片用完停在以下代码：

```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry[] newTable = new Entry[newCapacity];
    // 线程1把所有元素迁移到自己创建的新哈希表内
    transfer(newTable);
    // 时间片耗尽，上述代码执行完毕暂停在此
    // 新哈希表还没替换旧哈希表
    table = newTable;
    threshold = (int)(newCapacity * loadFactor);
}
```

**线程2** 操作恢复，如下图所示：

- 对 **线程2** 来说，不知道 **线程1** 已经改变以下元素的链表顺序；
- 只知道自己的 **引用e** 依然指向 **元素a**，**e.next** 指向 **元素b**；

![conflict_0](/img/java/JDK7_HashMap/conflict_0.png)

**线程2** 恢复 __void transfer(Entry[] newTable)__ 的操作：

- 把 **引用e** 所指定的 **元素a** 头插法放入自己的 **哈希桶7** 内；

- **引用e** 指向 **元素a** 下一个链表节点 **元素b**；
- 同理 **e.next** 本来指向 **元素b**，根据 **元素b的next** 又指回 **元素a**；

![conflict_1](/img/java/JDK7_HashMap/conflict_1.png)

**元素b** 也通过头插法插入到 **线程2** 的 **哈希桶7**。顺序变成：__哈希桶7->元素b->元素a__

![conflict_2](/img/java/JDK7_HashMap/conflict_2.png)

**线程2** 继续执行：

- **引用e** 继续移动指向 **元素a**，因为 **元素a的next** 为null，所以 **e.next** 也为null；
- 从上图知道原来顺序是 __哈希桶7->元素b->元素a__；
- **元素a** 头插法到 **哈希桶7**，顺序变为下图的 __哈希桶7->元素a->元素b__；
- 至此，**元素a** 被两次插入到 **哈希桶7**，导致 **元素b** 和 **元素a** 互相引用；

![aconflict_3](/img/java/JDK7_HashMap/aconflict_3.png)

最后，如果 **线程1** 先把新哈希桶替换旧哈希桶，后续 **线程2** 又覆盖 **线程1** 的桶替换操作，那么 **元素c** 将永远不会得到访问。

当然，无论哪个线程最终完成旧哈希桶替换，只要执行 __get(Key)__ 操作且命中到 **哈希桶7**，都会发生死循环。

## 五、参考链接

- [老生常谈，HashMap的死循环](https://www.jianshu.com/p/1e9cf0ac07f4)
- [jdk7/java/util/HashMap](http://hg.openjdk.java.net/jdk7/jdk7/jdk/file/9b8c96f96a0f/src/share/classes/java/util/HashMap.java)





