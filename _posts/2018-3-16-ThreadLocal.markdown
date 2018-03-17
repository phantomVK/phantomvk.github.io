---
layout:     post
title:      "Java源码系列(8) -- ThreadLocal"
date:       2018-03-16
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、ThreadLocal的作用

Thread.ThreadLocal为线程提供自有的线程局部变量，作为线程内部的独立副本，避免线程竞争同一个变量带来锁的时间损耗。

![ThreadLoacalMap](/img/java/thredLocalRef.png)

- __Entry[1]__ : Entry.key通过弱引用持有ThreadLocal实例
- __Entry[10]__ : 没有强应用持有ThreadLocal，下一次GC立即回收此实例，令Entry.key = null

## 二、示例

例子中共有5个线程，每个线程给自有ThreadLocal累加0到9。

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

ThreadLocals依靠捆绑在每个线程的线性探针哈希表。ThreadLocals对象作为Hash的key，通过threadLocalHashCode来搜索。这个值是一个自定义哈希值，只用在ThreadLocalMaps里，即使在同一个线程内连续构建ThreadLocals也可以运行得相当好。

```java
/**
 * ThreadLocals rely on per-thread linear-probe hash maps attached
 * to each thread (Thread.threadLocals and
 * inheritableThreadLocals).  The ThreadLocal objects act as keys,
 * searched via threadLocalHashCode.  This is a custom hash code
 * (useful only within ThreadLocalMaps) that eliminates collisions
 * in the common case where consecutively constructed ThreadLocals
 * are used by the same threads, while remaining well-behaved in
 * less common cases.
 */
private final int threadLocalHashCode = nextHashCode();
```

不同线程访问同一个ThreadLocals，这个ThreadLocals在不同ThreadLocalMap的hash值不是一样的。Hash值由静态[AtomicInteger](https://phantomvk.github.io/2018/01/17/AtomicInteger/)提供，ThreadLocals每创建一个新实例就会累加相同魔数`HASH_INCREMENT`。因此，如果同一个线程ThreadLocals被init再remove，来回执行多次，每次ThreadLocals计算得到的Hash值都是不同的。



```java
/**
 * The next hash code to be given out. Updated atomically. Starts at
 * zero.
 */
// 预计算下一实例的Hash值，从初始值0开始自动累加
private static AtomicInteger nextHashCode = new AtomicInteger();

// The difference between successively generated hash codes - turns
// implicit sequential thread-local IDs into near-optimally spread
// multiplicative hash values for power-of-two-sized tables.
private static final int HASH_INCREMENT = 0x61c88647;
```

### 3.2 nextHashCode

返回下一个哈希值

```java
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

### 3.3 initialValue()

给当前线程返回thread-local变量的初始值，initialValue()方法可见性是protected。当线程首次访问ThreadLoacal.get()方法时调用此方法。除非线程在很早的时候已经主动调用set()设置初始值，这种情况下这个方法就不会被调用。

一般来说这个方法在每个线程中最多只会调用一次。不过，线程如果执行了remove()方法后再调用get()，也会触发initialValue()。如果不想初始化的默认值是null，那必须重写这个方法提供特定的初始化值。

```java
/**
 * Returns the current thread's "initial value" for this
 * thread-local variable.  This method will be invoked the first
 * time a thread accesses the variable with the {@link #get}
 * method, unless the thread previously invoked the {@link #set}
 * method, in which case the {@code initialValue} method will not
 * be invoked for the thread.  Normally, this method is invoked at
 * most once per thread, but it may be invoked again in case of
 * subsequent invocations of {@link #remove} followed by {@link #get}.
 *
 * <p>This implementation simply returns {@code null}; if the
 * programmer desires thread-local variables to have an initial
 * value other than {@code null}, {@code ThreadLocal} must be
 * subclassed, and this method overridden.  Typically, an
 * anonymous inner class will be used.
 *
 * @return the initial value for this thread-local
 */
protected T initialValue() {
    return null;
}
```

```java
/**
 * Creates a thread local variable. The initial value of the variable is
 * determined by invoking the {@code get} method on the {@code Supplier}.
 *
 * @param <S> the type of the thread local's value
 * @param supplier the supplier to be used to determine the initial value
 * @return a new thread local variable
 * @throws NullPointerException if the specified supplier is null
 * @since 1.8
 */
public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
    return new SuppliedThreadLocal<>(supplier);
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
    // map == null，线程尚未初始化ThreadLocalMap
    return setInitialValue();
}
```

### 3.5 setInitialValue()

Thread的ThreadLocalMap已经存在，调map.set设置ThreadLocal和对应value，否则需要先初始化。

```java
/**
 * Variant of set() to establish initialValue. Used instead
 * of set() in case user has overridden the set() method.
 *
 * @return the initial value
 */
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    // Thread已有对应的ThreadLocalMap
    if (map != null)
        // 设置指定的初始化值value
        map.set(this, value);
    else
        // ThreadLocalMap为null，创建后设置value值
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

把当前线程的thread-local变量的副本设置为指定值。多数情况下子类没有必要重写这个方法，独立依赖initialValue()去设置thread-locals的值。

```java
/**
 * Sets the current thread's copy of this thread-local variable
 * to the specified value.  Most subclasses will have no need to
 * override this method, relying solely on the {@link #initialValue}
 * method to set the values of thread-locals.
 *
 * @param value the value to be stored in the current thread's copy of
 *        this thread-local.
 */
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

用于从当前线程的ThreadLocalMap中移除此thread-local的变量。主要调用了ThreadLocalMap.remove()。

```java
/**
 * Removes the current thread's value for this thread-local
 * variable.  If this thread-local variable is subsequently
 * {@linkplain #get read} by the current thread, its value will be
 * reinitialized by invoking its {@link #initialValue} method,
 * unless its value is {@linkplain #set set} by the current thread
 * in the interim.  This may result in multiple invocations of the
 * {@code initialValue} method in the current thread.
 *
 * @since 1.5
 */
 public void remove() {
     ThreadLocalMap m = getMap(Thread.currentThread());
     if (m != null)
         m.remove(this);
 }
```

### 3.8 getMap( )

返回线程中的ThreadLocalMap

```java
/**
 * Get the map associated with a ThreadLocal. Overridden in
 * InheritableThreadLocal.
 *
 * @param  t the current thread
 * @return the map
 */
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

Thread.threadLocals在Thread类的定义如下：

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

### 3.8 createMap( )

在当前线程中初始化对应的ThreadLocalMap，并把firstValue作为第一个Entry放入。

```java
/**
 * Create the map associated with a ThreadLocal. Overridden in
 * InheritableThreadLocal.
 *
 * @param t the current thread
 * @param firstValue value for the initial entry of the map
 */
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

### 3.9 createInheritedMap( )

```java
/**
 * Factory method to create map of inherited thread locals.
 * Designed to be called only from Thread constructor.
 *
 * @param  parentMap the map associated with parent thread
 * @return a map containing the parent's inheritable bindings
 */
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}
```

### 3.10 SuppliedThreadLocal

```java
/**
 * An extension of ThreadLocal that obtains its initial value from
 * the specified {@code Supplier}.
 */
static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

    private final Supplier<? extends T> supplier;

    SuppliedThreadLocal(Supplier<? extends T> supplier) {
        this.supplier = Objects.requireNonNull(supplier);
    }

    @Override
    protected T initialValue() {
        return supplier.get();
    }
}
```


## 四、内部类ThreadLocalMap
```java
/**
 * ThreadLocalMap is a customized hash map suitable only for
 * maintaining thread local values. No operations are exported
 * outside of the ThreadLocal class. The class is package private to
 * allow declaration of fields in class Thread.  To help deal with
 * very large and long-lived usages, the hash table entries use
 * WeakReferences for keys. However, since reference queues are not
 * used, stale entries are guaranteed to be removed only when
 * the table starts running out of space.
 */
static class ThreadLocalMap
```

### 4.1 Entry

Entry继承WeakReference父类，当Entry的key没有被外界引用，这个key在下一次GC的时候直接回收。虽然key已被回收，但是Entry本身和Entry的value依然由强引用持有。后面会说明这种情况下Entry如何被安全回收。

```java
/**
 * The entries in this hash map extend WeakReference, using
 * its main ref field as the key (which is always a
 * ThreadLocal object).  Note that null keys (i.e. entry.get()
 * == null) mean that the key is no longer referenced, so the
 * entry can be expunged from table.  Such entries are referred to
 * as "stale entries" in the code that follows.
 */
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;
    
    // key由父构造方法赋值到WeakReference，value保存在Entry本身
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

### 4.2 内部类数据成员

```java
// 初始容量值，必须为2的幂
private static final int INITIAL_CAPACITY = 16;

/**
 * The table, resized as necessary.
 * table.length MUST always be a power of two.
 */
// 由扩容的方法控制长度符合2的幂
private Entry[] table;

// 已保存在表中的条目数量
private int size = 0;

// 由setThreshold()计算的阀值，超过该值就扩大table，避免哈希冲突；
// 默认容量为16，可算出首个阀值：16 * 2 / 3 = 10
private int threshold; // Default to 0
```

### 4.3 setThreshold(int) 

```java
// 从 len * 2 / 3 可知负载因子约为0.67
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}
```

### 4.4 索引管理

```java
/**
 * Increment i modulo len.
 */
// 如果i+1没有超过长度，则后移索引，否则索引回到头部索引0
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}
```

```java
/**
 * Decrement i modulo len.
 */
// 如果i-1没有小于长度，则前移索引，否则索引跳到尾部索引len-1
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
```

```java
/**
 * Construct a new map initially containing (firstKey, firstValue).
 * ThreadLocalMaps are constructed lazily, so we only create
 * one when we have at least one entry to put in it.
 */
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

```java
/**
 * Construct a new map including all Inheritable ThreadLocals
 * from given parent map. Called only by createInheritedMap.
 *
 * @param parentMap the map associated with parent thread.
 */
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```

### 4.5 getEntry(ThreadLocal<?>)

```java
/**
 * Get the entry associated with key.  This method
 * itself handles only the fast path: a direct hit of existing
 * key. It otherwise relays to getEntryAfterMiss.  This is
 * designed to maximize performance for direct hits, in part
 * by making this method readily inlinable.
 *
 * @param  key the thread local object
 * @return the entry associated with key, or null if no such
 */
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

### 4.6 getEntryAfterMiss(ThreadLocal<?>, int, Entry)

```java
/**
 * Version of getEntry method for use when key is not found in
 * its direct hash slot.
 *
 * @param  key the thread local object
 * @param  i the table index for key's hash code
 * @param  e the entry at table[i]
 * @return the entry associated with key, or null if no such
 */
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

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

### 4.7 set(ThreadLocal<?>, Object)

```java
/**
 * Set the value associated with key.
 *
 * @param key the thread local object
 * @param value the value to be set
 */
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

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

### 4.8 remove(ThreadLocal<?>) 

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

### 4.9 replaceStaleEntry(ThreadLocal<?>, Object, int)

```java
/**
 * Replace a stale entry encountered during a set operation
 * with an entry for the specified key.  The value passed in
 * the value parameter is stored in the entry, whether or not
 * an entry already exists for the specified key.
 *
 * As a side effect, this method expunges all stale entries in the
 * "run" containing the stale entry.  (A run is a sequence of entries
 * between two null slots.)
 *
 * @param  key the key
 * @param  value the value to be associated with key
 * @param  staleSlot index of the first stale entry encountered while
 *         searching for key.
 */
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

    // If key not found, put new entry in stale slot
    // key没找到，创建新的Entry到位置staleSlot上
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // If there are any other stale entries in run, expunge them
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

### 4.10 expungeStaleEntry(int)

```java
/**
 * Expunge a stale entry by rehashing any possibly colliding entries
 * lying between staleSlot and the next null slot.  This also expunges
 * any other stale entries encountered before the trailing null.  See
 * Knuth, Section 6.4
 *
 * @param staleSlot index of slot known to have null key
 * @return the index of the next null slot after staleSlot
 * (all between staleSlot and this slot will have been checked
 * for expunging).
 */
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 移除Entry.value的值
    tab[staleSlot].value = null;
    // 后Entry从Entry[]中移除
    tab[staleSlot] = null;
    // 元素总数减一
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
        // 如果key为null，则清空Entry的value并将其移出Entry[]
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


### 4.11 cleanSomeSlots(int, int)

```java
/**
 * Heuristically scan some cells looking for stale entries.
 * This is invoked when either a new element is added, or
 * another stale one has been expunged. It performs a
 * logarithmic number of scans, as a balance between no
 * scanning (fast but retains garbage) and a number of scans
 * proportional to number of elements, that would find all
 * garbage but would cause some insertions to take O(n) time.
 *
 * @param i a position known NOT to hold a stale entry. The
 * scan starts at the element after i.
 *
 * @param n scan control: {@code log2(n)} cells are scanned,
 * unless a stale entry is found, in which case
 * {@code log2(table.length)-1} additional cells are scanned.
 * When called from insertions, this parameter is the number
 * of elements, but when from replaceStaleEntry, it is the
 * table length. (Note: all this could be changed to be either
 * more or less aggressive by weighting n instead of just
 * using straight log n. But this version is simple, fast, and
 * seems to work well.)
 *
 * @return true if any stale entries have been removed.
 */
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

### 4.12 rehash()

此方法会触发一次清理整个表废弃元素的操作expungeStaleEntries()。一般来说，经过全面清理后会腾出一谢新空间。然后检查现在是否需要扩容，即已用空间占总空间3/4。

```java
private void rehash() {
    // 先清空所有无效的Entry腾出空间
    expungeStaleEntries();

    // Use lower threshold for doubling to avoid hysteresis
    // 已用空间大于等于总空间的3/4，resize()扩容为原大小的2倍    
    if (size >= threshold - threshold / 4)
        resize();
}
```

### 4.13 resize()

table数组容量扩容为原大小2倍，因此也符合tableSize为2的幂大小的要求

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

### 4.14 expungeStaleEntries()

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

https://en.wikipedia.org/wiki/Hysteresis

