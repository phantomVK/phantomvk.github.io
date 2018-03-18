---
layout:     post
title:      "Java源码系列(8) -- ThreadLocal"
date:       2018-03-17
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、ThreadLocal的作用

ThreadLocal提供线程局部变量，以静态私有数据成员的形式存放在Thread中，线程与线程之间副本相互独立。这些副本存放在Thread.ThreadLocalMap中，避免线程竞争同一个实例带来锁操作的时间损耗。

![ThreadLoacalMap](/img/java/thredLocalRef.png)

- __Entry[1]__ : Entry.key通过弱引用持有ThreadLocal实例；
- __Entry[10]__ : GC没有强应用持有的ThreadLocal实例，令Entry.key为null，Entry.value继续等待清理；
- __其他__ ：Entry没有被使用，或曾经使用但已回收，Entry处于初始状态。

## 二、示例

例子中共有5个线程，每个线程给私有的ThreadLocal累加0到9。

```java
fun main(args: Array<String>) {
    val threads = arrayOfNulls<Thread>(5)
    for (i in 0 until 5) {
        threads[i] = Thread {
            System.out.println("${Thread.currentThread()}'s initial value is ${threadLocal.get()}")

            for (j in 0 until 10) {
                threadLocal.set(threadLocal.get() + j)
            }

            System.out.println("${Thread.currentThread()}'s result is ${threadLocal.get()}")
        }
    }

    for (i in threads) {
        i!!.start()
    }
}

// 重写initialValue并返回初始值0
val threadLocal = object : ThreadLocal<Int>() {
    override fun initialValue() = 0
}
```

每个线程完成累加的结果值不会影响其他线程的初始值，可知每个线程之间的操作是完全独立的。

```
Thread[Thread-0,5,main]'s initial value is 0
Thread[Thread-0,5,main]'s result is 45

Thread[Thread-1,5,main]'s initial value is 0
Thread[Thread-1,5,main]'s result is 45

Thread[Thread-2,5,main]'s initial value is 0
Thread[Thread-2,5,main]'s result is 45

Thread[Thread-3,5,main]'s initial value is 0
Thread[Thread-3,5,main]'s result is 45

Thread[Thread-4,5,main]'s initial value is 0
Thread[Thread-4,5,main]'s result is 45
```

## 三、源码剖析

### 3.1 成员变量

ThreadLocals依靠捆绑在每个线程的线性探针哈希表(ThreadLocalMap.Entry[])。ThreadLocal对象作为Hash的键，通过`threadLocalHashCode`来搜索，且该值只用在ThreadLocalMaps的定制哈希值，即使在同一个线程内连续构建ThreadLocals也可以运行得相当好。

```java
private final int threadLocalHashCode = nextHashCode();
```

不同线程访问同一个ThreadLocal，Hash值由静态的[AtomicInteger：Java源码系列(7)](https://phantomvk.github.io/2018/01/17/AtomicInteger/)提供，ThreadLocal创建新实例就会累加相同魔数`HASH_INCREMENT`。

```java
// 预计算下一实例的Hash值，从初始值0开始自动累加
private static AtomicInteger nextHashCode = new AtomicInteger();

// The difference between successively generated hash codes - turns
// implicit sequential thread-local IDs into near-optimally spread
// multiplicative hash values for power-of-two-sized tables.
private static final int HASH_INCREMENT = 0x61c88647;
```

### 3.2 nextHashCode()

返回下一个哈希值

```java
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

### 3.3 initialValue()

给当前线程返回thread-local变量的初始值，initialValue()方法可见性是protected。当线程首次访问ThreadLoacal.get()方法时调用此方法。线程很早已经主动调用set()设置初始值的话，此方法不会调用。

一般来说这个方法在每个线程中最多只会调用一次。不过，线程如果执行了remove()方法后再调用get()，也会触发initialValue()。

如果不希望初始化值是null，就重写这个方法提供特定初始化值。可以通过子类继承父类或匿名内部类两个方法重写initialValue()，不过后者比较典型。

```java
protected T initialValue() {
    return null;
}
```


### 3.4 get()

此方法返回当前线程在`thread-local`中变量的副本。如果当前线程不存在对应的线程局部变量，则调用`setInitialValue()`，并由`setInitialValue()`中`initialValue()`给出初始化值。

```java
public T get() 
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取当前线程ThreadLocalMap
    ThreadLocalMap map = getMap(t);

    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            // e.value显式类型转换 Object -> T
            T result = (T)e.value;
            return result;
        }
    }
    // map == null，去初始化ThreadLocalMap
    return setInitialValue();
}
```

### 3.5 setInitialValue()

若Thread.ThreadLocalMap已经存在，调map.set()设置该ThreadLocal和对应value。

```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    // Thread已有对应的ThreadLocalMap
    if (map != null)
        // 设置指定的初始化值value
        map.set(this, value);
    else
        // ThreadLocalMap为null，创建后再设置value值
        createMap(t, value);
    return value;
}
```

从示例代码可知`setInitialValue()`中`initialValue()`返回值就是下面的0。

```java
val threadLocal = object : ThreadLocal<Int>() {
    override fun initialValue() = 0
}
```

### 3.6 set( )

把当前线程的thread-local设置为指定值。多数情况下子类没有必要重写这个方法

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

### 3.7 remove( )

用于从当前线程的ThreadLocalMap中移除此thread-local的值

```java
 public void remove() {
     ThreadLocalMap m = getMap(Thread.currentThread());
     if (m != null)
         m.remove(this);
 }
```

### 3.8 getMap( )

返回线程的ThreadLocalMap，此方法在InheritableThreadLocal被重写了

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

Thread.threadLocals从属于Thread，但是维护却由ThreadLocal类负责

```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

### 3.8 createMap( )

在当前线程中初始化ThreadLocalMap，并把firstValue作为第一个Entry放入。InheritableThreadLocal重写了这个方法。

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

### 3.10 SuppliedThreadLocal

从Supplier获取ThreadLocal初始值，重写的initialValue()从Supplier.get()中取得变量。

```java
static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

    private final Supplier<? extends T> supplier;

    SuppliedThreadLocal(Supplier<? extends T> supplier) {
        this.supplier = Objects.requireNonNull(supplier);
    }

    // 重写了ThreadLocal.initialValue()
    @Override
    protected T initialValue() {
        return supplier.get();
    }
}
```

## 四、内部类ThreadLocalMap

ThreadLocalMap是一个定制的哈希表，只给线程局部值使用。所有操作都不会导出ThreadLocal类以外。此类包私有。为了应对大量长生命周期的对象，哈希表使用了弱引用持有所有键。由于没有用引用队列(例如LruCache)，那些废弃的记录只保证在空间用尽时(触发扩容阀值时)才尝试移除并回收。

```java
static class ThreadLocalMap
```

### 4.1 Entry

Entry继承WeakReference父类，当Entry的key没有被外界引用，key对应的ThreadLocal在下一次GC回收。虽然key已被回收，但是Entry本身和Entry的value依然由强引用持有。后面会说明这种情况下Entry如何被安全回收。

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    // ThreadLocal关联的value
    Object value;
    
    // key由父构造方法赋值到WeakReference，value保存在Entry本身
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

### 4.2 数据成员

```java
// 初始容量值16，为2的幂
private static final int INITIAL_CAPACITY = 16;

// 由扩容方法控制长度符合2的幂，且仅在必要时扩容
private Entry[] table;

// 已保存在表中的条目数量
private int size = 0;

// 由setThreshold()计算的阀值，超过该值就扩大table，避免哈希冲突；
// 假设默认容量为16，可算出首个阀值(地板数)：16 * 2 / 3 = 10
private int threshold;
```

### 4.3 setThreshold(int) 

```java
// 从 len * 2 / 3 可知负载因子约为0.66
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}
```

### 4.4 索引管理

```java
// 如果i+1没有超过长度则后移索引，否则索引回到头部索引0
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}
```

```java
// 如果i-1没有小于长度则前移索引，否则索引跳到尾部索引len-1
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
```

### 4.5 构造方法

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    // 构造大小为16的table
    table = new Entry[INITIAL_CAPACITY];
    // 计算ThreadLocal的Hash在上述长度的下标
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

### 4.6 getEntry(ThreadLocal<?>)

用关联的key获取对应entry。如果索引位置的entry合法就直接返回，否则需要getEntryAfterMiss方法查找。方法的设计主要为了最大限度达到直接命中的目的，且令方法更容易被内联。

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

### 4.7 getEntryAfterMiss(ThreadLocal<?>, int, Entry)

从getEntry(ThreadLocal<?>)到getEntryAfterMiss有两种情况：

- e == null
- e.get != key

 getEntry方法没有找到直接下标的key，就调用此方法继续向后查找，一遍清理一边查找，直到找到目标键或遇到e == null时退出。

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    // e.get == null走此分支
    while (e != null) {
        ThreadLocal<?> k = e.get();
        // 命中指定ThreadLocal，返回该Entry
        if (k == key)
            return e;
        // Entry[]遇到的entry为空，清理该废弃的Entry
        // 如果遇到的entry不为空，继续迭代下一个下标
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    
    // e为null直接退出
    return null;
}
```

### 4.8 set(ThreadLocal<?>, Object)

```java
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    // 计算Key所在的下标
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        // key已经存在，修改Entry.value为新值
        if (k == key) {
            e.value = value;
            return;
        }

        // k为空
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    // 这个key不存在，创建新的entry放入
    tab[i] = new Entry(key, value);
    // 创建新的entry需要更改数量
    int sz = ++size;
    // 看看是否到了扩容阀值进行扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

### 4.9 remove(ThreadLocal<?>) 

```java
// 通过Key查找对应Entry并移除
private void remove(ThreadLocal<?> key) {
    // 获取ThreadLocalMap的Entry数组
    Entry[] tab = table;
    // 取数组的长度，值为2的幂大小
    int len = tab.length;
    // 假设len为16，二进制表示为1000，len-1即15的二进制是0111
    // key.threadLocalHashCode与(len-1)进行位运算定位Entry[]的下标
    // 通过%也能实现相同目的，但位运算执行性能更好
    int i = key.threadLocalHashCode & (len-1);
    // 从下标开始查找直到遇到null，或匹配到Key的Entry执行移除操作
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        // 匹配到对应key，移除e，并执行
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```

### 4.10 replaceStaleEntry(ThreadLocal<?>, Object, int)

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // Back up to check for prior stale entry in current run.
    // We clean out whole runs at a time to avoid continual
    // incremental rehashing due to garbage collector freeing
    // up refs in bunches (i.e., whenever the collector runs).
    int slotToExpunge = staleSlot;
    // 向前遍历，查找遇到的key为null的Entry
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    // Find either the key or trailing null slot of run, whichever
    // occurs first
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // If we find key, then we need to swap it
        // with the stale entry to maintain hash table order.
        // The newly stale slot, or any other stale slot
        // encountered above it, can then be sent to expungeStaleEntry
        // to remove or rehash all of the other entries in run.
        // 命中指定ThreadLoacal的Entry
        if (k == key) {
            // 新value替换原value
            e.value = value;
            
            // 交换位置i与位置staleSlot的Entry以维护hash顺序
            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            // 清除Entry.key为空的Entry
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // If we didn't find stale entry on backward scan, the
        // first stale entry seen while scanning for key is the
        // first still present in the run.
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // key没找到，创建新的Entry到位置staleSlot上
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // 有任何废弃的entry就触发清理逻辑
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

### 4.11 expungeStaleEntry(int)

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 移除Entry.value的值
    tab[staleSlot].value = null;
    // 后Entry从Entry[]中移除
    tab[staleSlot] = null;
    // 元素总数递减
    size--;

    // Rehash until we encounter null
    // 开放地址法需重Hash后续元素直到遇到null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        // 取Entry
        ThreadLocal<?> k = e.get();
        // 如果key为null，则清空Entry的value并将Entry移出Entry[]
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            // key非空，重新hash该Entry的新下标
            int h = k.threadLocalHashCode & (len - 1);
            // 出现Hash则继续通过开放地址法找后续空下标
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```


### 4.12 cleanSomeSlots(int, int)

```java
private boolean cleanSomeSlots(int i, int n) {
    // 记录是否有元素被回收
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    // 从i的下一个下标开始，i = i + 1
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        // 检查i位置的Entry.key
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i); // 清除Entry[i]
        }
    } while ( (n >>>= 1) != 0); // aka: (n = n / 2) != 0
    return removed;
}
```

### 4.13 rehash()

此方法会触发一次清理整个表废弃元素的操作expungeStaleEntries()。一般来说，经过全面清理后会腾出新空间，然后检查是否还需要扩容，即是否已达到扩容阀值的3 / 4。

假设总容量是16：

- 扩容阀值：16 * 2 / 3 = 10
- 扩容阀值的3 / 4 : 10 * 3 / 4 = 7

```java
private void rehash() {
    // 先清空所有无效的Entry腾出空间
    expungeStaleEntries();

    // 已用空间大于等于阀值3/4，resize()扩容为原来2倍，以此避免迟滞现象  
    if (size >= threshold - threshold / 4)
        resize();
}
```

### 4.14 resize()

table数组容量扩容2倍，tableSize为2的幂大小的要求

```java
private void resize() {
    Entry[] oldTab = table;
    // 假设oldTab.length为16，可知newLen为32
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;

    // 创建新的Entry[]
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    // 按顺序遍历旧表的元素，过时元素就地清理，否则重Hash并放入新位置
    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                // 通过Hash计算元素在新Entry[]的下标
                int h = k.threadLocalHashCode & (newLen - 1);
                // 利用开放地址法找到空位置的下标.
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                // 元素放入下标的数组位置中
                newTab[h] = e;
                // 数量自增
                count++;
            }
        }
    }
    
    // 当所有旧元素重Hash并放入新Entry[]后，更新下一次扩容的阀值；
    // 假设容量已扩展到32，新threshold为32*2/3=21
    setThreshold(newLen);
    // 更新已保存Entry数量
    size = count;
    // 新Entry[]替换旧的Entry[]，新容量为旧容量的2倍
    table = newTab;
}
```

### 4.15 expungeStaleEntries()

查找整个Entry[]，移除所有Entry.key为null的Entry

```java
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        // 发现key为null，对该下标的Entry进行处理
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
```

## 五、参考链接

- Hysteresis: [https://en.wikipedia.org/wiki/Hysteresis](https://en.wikipedia.org/wiki/Hysteresis)

