---
layout:     post
title:      "Java源码系列(12) -- ConcurrentHashMap"
date:       2018-10-16
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---

JDK11

## 一、类签名

```java
/**
 * A hash table supporting full concurrency of retrievals and
 * high expected concurrency for updates. This class obeys the
 * same functional specification as {@link java.util.Hashtable}, and
 * includes versions of methods corresponding to each method of
 * {@code Hashtable}. However, even though all operations are
 * thread-safe, retrieval operations do <em>not</em> entail locking,
 * and there is <em>not</em> any support for locking the entire table
 * in a way that prevents all access.  This class is fully
 * interoperable with {@code Hashtable} in programs that rely on its
 * thread safety but not on its synchronization details.
 *
 * <p>Retrieval operations (including {@code get}) generally do not
 * block, so may overlap with update operations (including {@code put}
 * and {@code remove}). Retrievals reflect the results of the most
 * recently <em>completed</em> update operations holding upon their
 * onset. (More formally, an update operation for a given key bears a
 * <em>happens-before</em> relation with any (non-null) retrieval for
 * that key reporting the updated value.)  For aggregate operations
 * such as {@code putAll} and {@code clear}, concurrent retrievals may
 * reflect insertion or removal of only some entries.  Similarly,
 * Iterators, Spliterators and Enumerations return elements reflecting the
 * state of the hash table at some point at or since the creation of the
 * iterator/enumeration.  They do <em>not</em> throw {@link
 * java.util.ConcurrentModificationException ConcurrentModificationException}.
 * However, iterators are designed to be used by only one thread at a time.
 * Bear in mind that the results of aggregate status methods including
 * {@code size}, {@code isEmpty}, and {@code containsValue} are typically
 * useful only when a map is not undergoing concurrent updates in other threads.
 * Otherwise the results of these methods reflect transient states
 * that may be adequate for monitoring or estimation purposes, but not
 * for program control.
 *
 * <p>The table is dynamically expanded when there are too many
 * collisions (i.e., keys that have distinct hash codes but fall into
 * the same slot modulo the table size), with the expected average
 * effect of maintaining roughly two bins per mapping (corresponding
 * to a 0.75 load factor threshold for resizing). There may be much
 * variance around this average as mappings are added and removed, but
 * overall, this maintains a commonly accepted time/space tradeoff for
 * hash tables.  However, resizing this or any other kind of hash
 * table may be a relatively slow operation. When possible, it is a
 * good idea to provide a size estimate as an optional {@code
 * initialCapacity} constructor argument. An additional optional
 * {@code loadFactor} constructor argument provides a further means of
 * customizing initial table capacity by specifying the table density
 * to be used in calculating the amount of space to allocate for the
 * given number of elements.  Also, for compatibility with previous
 * versions of this class, constructors may optionally specify an
 * expected {@code concurrencyLevel} as an additional hint for
 * internal sizing.  Note that using many keys with exactly the same
 * {@code hashCode()} is a sure way to slow down performance of any
 * hash table. To ameliorate impact, when keys are {@link Comparable},
 * this class may use comparison order among keys to help break ties.
 *
 * <p>A {@link Set} projection of a ConcurrentHashMap may be created
 * (using {@link #newKeySet()} or {@link #newKeySet(int)}), or viewed
 * (using {@link #keySet(Object)} when only keys are of interest, and the
 * mapped values are (perhaps transiently) not used or all take the
 * same mapping value.
 *
 * <p>A ConcurrentHashMap can be used as a scalable frequency map (a
 * form of histogram or multiset) by using {@link
 * java.util.concurrent.atomic.LongAdder} values and initializing via
 * {@link #computeIfAbsent computeIfAbsent}. For example, to add a count
 * to a {@code ConcurrentHashMap<String,LongAdder> freqs}, you can use
 * {@code freqs.computeIfAbsent(key, k -> new LongAdder()).increment();}
 *
 * <p>This class and its views and iterators implement all of the
 * <em>optional</em> methods of the {@link Map} and {@link Iterator}
 * interfaces.
 *
 * <p>Like {@link Hashtable} but unlike {@link HashMap}, this class
 * does <em>not</em> allow {@code null} to be used as a key or value.
 *
 * <p>ConcurrentHashMaps support a set of sequential and parallel bulk
 * operations that, unlike most {@link Stream} methods, are designed
 * to be safely, and often sensibly, applied even with maps that are
 * being concurrently updated by other threads; for example, when
 * computing a snapshot summary of the values in a shared registry.
 * There are three kinds of operation, each with four forms, accepting
 * functions with keys, values, entries, and (key, value) pairs as
 * arguments and/or return values. Because the elements of a
 * ConcurrentHashMap are not ordered in any particular way, and may be
 * processed in different orders in different parallel executions, the
 * correctness of supplied functions should not depend on any
 * ordering, or on any other objects or values that may transiently
 * change while computation is in progress; and except for forEach
 * actions, should ideally be side-effect-free. Bulk operations on
 * {@link Map.Entry} objects do not support method {@code setValue}.
 *
 * <ul>
 * <li>forEach: Performs a given action on each element.
 * A variant form applies a given transformation on each element
 * before performing the action.
 *
 * <li>search: Returns the first available non-null result of
 * applying a given function on each element; skipping further
 * search when a result is found.
 *
 * <li>reduce: Accumulates each element.  The supplied reduction
 * function cannot rely on ordering (more formally, it should be
 * both associative and commutative).  There are five variants:
 *
 * <ul>
 *
 * <li>Plain reductions. (There is not a form of this method for
 * (key, value) function arguments since there is no corresponding
 * return type.)
 *
 * <li>Mapped reductions that accumulate the results of a given
 * function applied to each element.
 *
 * <li>Reductions to scalar doubles, longs, and ints, using a
 * given basis value.
 *
 * </ul>
 * </ul>
 *
 * <p>These bulk operations accept a {@code parallelismThreshold}
 * argument. Methods proceed sequentially if the current map size is
 * estimated to be less than the given threshold. Using a value of
 * {@code Long.MAX_VALUE} suppresses all parallelism.  Using a value
 * of {@code 1} results in maximal parallelism by partitioning into
 * enough subtasks to fully utilize the {@link
 * ForkJoinPool#commonPool()} that is used for all parallel
 * computations. Normally, you would initially choose one of these
 * extreme values, and then measure performance of using in-between
 * values that trade off overhead versus throughput.
 *
 * <p>The concurrency properties of bulk operations follow
 * from those of ConcurrentHashMap: Any non-null result returned
 * from {@code get(key)} and related access methods bears a
 * happens-before relation with the associated insertion or
 * update.  The result of any bulk operation reflects the
 * composition of these per-element relations (but is not
 * necessarily atomic with respect to the map as a whole unless it
 * is somehow known to be quiescent).  Conversely, because keys
 * and values in the map are never null, null serves as a reliable
 * atomic indicator of the current lack of any result.  To
 * maintain this property, null serves as an implicit basis for
 * all non-scalar reduction operations. For the double, long, and
 * int versions, the basis should be one that, when combined with
 * any other value, returns that other value (more formally, it
 * should be the identity element for the reduction). Most common
 * reductions have these properties; for example, computing a sum
 * with basis 0 or a minimum with basis MAX_VALUE.
 *
 * <p>Search and transformation functions provided as arguments
 * should similarly return null to indicate the lack of any result
 * (in which case it is not used). In the case of mapped
 * reductions, this also enables transformations to serve as
 * filters, returning null (or, in the case of primitive
 * specializations, the identity basis) if the element should not
 * be combined. You can create compound transformations and
 * filterings by composing them yourself under this "null means
 * there is nothing there now" rule before using them in search or
 * reduce operations.
 *
 * <p>Methods accepting and/or returning Entry arguments maintain
 * key-value associations. They may be useful for example when
 * finding the key for the greatest value. Note that "plain" Entry
 * arguments can be supplied using {@code new
 * AbstractMap.SimpleEntry(k,v)}.
 *
 * <p>Bulk operations may complete abruptly, throwing an
 * exception encountered in the application of a supplied
 * function. Bear in mind when handling such exceptions that other
 * concurrently executing functions could also have thrown
 * exceptions, or would have done so if the first exception had
 * not occurred.
 *
 * <p>Speedups for parallel compared to sequential forms are common
 * but not guaranteed.  Parallel operations involving brief functions
 * on small maps may execute more slowly than sequential forms if the
 * underlying work to parallelize the computation is more expensive
 * than the computation itself.  Similarly, parallelization may not
 * lead to much actual parallelism if all processors are busy
 * performing unrelated tasks.
 *
 * <p>All arguments to all task methods must be non-null.
 *
 * <p>This class is a member of the
 * <a href="{@docRoot}/java.base/java/util/package-summary.html#CollectionsFramework">
 * Java Collections Framework</a>.
 *
 * @since 1.5
 * @author Doug Lea
 * @param <K> the type of keys maintained by this map
 * @param <V> the type of mapped values
 */
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable
```

## 二、常量

```java
/*
 * Overview:
 *
 * The primary design goal of this hash table is to maintain
 * concurrent readability (typically method get(), but also
 * iterators and related methods) while minimizing update
 * contention. Secondary goals are to keep space consumption about
 * the same or better than java.util.HashMap, and to support high
 * initial insertion rates on an empty table by many threads.
 *
 * This map usually acts as a binned (bucketed) hash table.  Each
 * key-value mapping is held in a Node.  Most nodes are instances
 * of the basic Node class with hash, key, value, and next
 * fields. However, various subclasses exist: TreeNodes are
 * arranged in balanced trees, not lists.  TreeBins hold the roots
 * of sets of TreeNodes. ForwardingNodes are placed at the heads
 * of bins during resizing. ReservationNodes are used as
 * placeholders while establishing values in computeIfAbsent and
 * related methods.  The types TreeBin, ForwardingNode, and
 * ReservationNode do not hold normal user keys, values, or
 * hashes, and are readily distinguishable during search etc
 * because they have negative hash fields and null key and value
 * fields. (These special nodes are either uncommon or transient,
 * so the impact of carrying around some unused fields is
 * insignificant.)
 *
 * The table is lazily initialized to a power-of-two size upon the
 * first insertion.  Each bin in the table normally contains a
 * list of Nodes (most often, the list has only zero or one Node).
 * Table accesses require volatile/atomic reads, writes, and
 * CASes.  Because there is no other way to arrange this without
 * adding further indirections, we use intrinsics
 * (jdk.internal.misc.Unsafe) operations.
 *
 * We use the top (sign) bit of Node hash fields for control
 * purposes -- it is available anyway because of addressing
 * constraints.  Nodes with negative hash fields are specially
 * handled or ignored in map methods.
 *
 * Insertion (via put or its variants) of the first node in an
 * empty bin is performed by just CASing it to the bin.  This is
 * by far the most common case for put operations under most
 * key/hash distributions.  Other update operations (insert,
 * delete, and replace) require locks.  We do not want to waste
 * the space required to associate a distinct lock object with
 * each bin, so instead use the first node of a bin list itself as
 * a lock. Locking support for these locks relies on builtin
 * "synchronized" monitors.
 *
 * Using the first node of a list as a lock does not by itself
 * suffice though: When a node is locked, any update must first
 * validate that it is still the first node after locking it, and
 * retry if not. Because new nodes are always appended to lists,
 * once a node is first in a bin, it remains first until deleted
 * or the bin becomes invalidated (upon resizing).
 *
 * The main disadvantage of per-bin locks is that other update
 * operations on other nodes in a bin list protected by the same
 * lock can stall, for example when user equals() or mapping
 * functions take a long time.  However, statistically, under
 * random hash codes, this is not a common problem.  Ideally, the
 * frequency of nodes in bins follows a Poisson distribution
 * (http://en.wikipedia.org/wiki/Poisson_distribution) with a
 * parameter of about 0.5 on average, given the resizing threshold
 * of 0.75, although with a large variance because of resizing
 * granularity. Ignoring variance, the expected occurrences of
 * list size k are (exp(-0.5) * pow(0.5, k) / factorial(k)). The
 * first values are:
 *
 * 0:    0.60653066
 * 1:    0.30326533
 * 2:    0.07581633
 * 3:    0.01263606
 * 4:    0.00157952
 * 5:    0.00015795
 * 6:    0.00001316
 * 7:    0.00000094
 * 8:    0.00000006
 * more: less than 1 in ten million
 *
 * Lock contention probability for two threads accessing distinct
 * elements is roughly 1 / (8 * #elements) under random hashes.
 *
 * Actual hash code distributions encountered in practice
 * sometimes deviate significantly from uniform randomness.  This
 * includes the case when N > (1<<30), so some keys MUST collide.
 * Similarly for dumb or hostile usages in which multiple keys are
 * designed to have identical hash codes or ones that differs only
 * in masked-out high bits. So we use a secondary strategy that
 * applies when the number of nodes in a bin exceeds a
 * threshold. These TreeBins use a balanced tree to hold nodes (a
 * specialized form of red-black trees), bounding search time to
 * O(log N).  Each search step in a TreeBin is at least twice as
 * slow as in a regular list, but given that N cannot exceed
 * (1<<64) (before running out of addresses) this bounds search
 * steps, lock hold times, etc, to reasonable constants (roughly
 * 100 nodes inspected per operation worst case) so long as keys
 * are Comparable (which is very common -- String, Long, etc).
 * TreeBin nodes (TreeNodes) also maintain the same "next"
 * traversal pointers as regular nodes, so can be traversed in
 * iterators in the same way.
 *
 * The table is resized when occupancy exceeds a percentage
 * threshold (nominally, 0.75, but see below).  Any thread
 * noticing an overfull bin may assist in resizing after the
 * initiating thread allocates and sets up the replacement array.
 * However, rather than stalling, these other threads may proceed
 * with insertions etc.  The use of TreeBins shields us from the
 * worst case effects of overfilling while resizes are in
 * progress.  Resizing proceeds by transferring bins, one by one,
 * from the table to the next table. However, threads claim small
 * blocks of indices to transfer (via field transferIndex) before
 * doing so, reducing contention.  A generation stamp in field
 * sizeCtl ensures that resizings do not overlap. Because we are
 * using power-of-two expansion, the elements from each bin must
 * either stay at same index, or move with a power of two
 * offset. We eliminate unnecessary node creation by catching
 * cases where old nodes can be reused because their next fields
 * won't change.  On average, only about one-sixth of them need
 * cloning when a table doubles. The nodes they replace will be
 * garbage collectable as soon as they are no longer referenced by
 * any reader thread that may be in the midst of concurrently
 * traversing table.  Upon transfer, the old table bin contains
 * only a special forwarding node (with hash field "MOVED") that
 * contains the next table as its key. On encountering a
 * forwarding node, access and update operations restart, using
 * the new table.
 *
 * Each bin transfer requires its bin lock, which can stall
 * waiting for locks while resizing. However, because other
 * threads can join in and help resize rather than contend for
 * locks, average aggregate waits become shorter as resizing
 * progresses.  The transfer operation must also ensure that all
 * accessible bins in both the old and new table are usable by any
 * traversal.  This is arranged in part by proceeding from the
 * last bin (table.length - 1) up towards the first.  Upon seeing
 * a forwarding node, traversals (see class Traverser) arrange to
 * move to the new table without revisiting nodes.  To ensure that
 * no intervening nodes are skipped even when moved out of order,
 * a stack (see class TableStack) is created on first encounter of
 * a forwarding node during a traversal, to maintain its place if
 * later processing the current table. The need for these
 * save/restore mechanics is relatively rare, but when one
 * forwarding node is encountered, typically many more will be.
 * So Traversers use a simple caching scheme to avoid creating so
 * many new TableStack nodes. (Thanks to Peter Levart for
 * suggesting use of a stack here.)
 *
 * The traversal scheme also applies to partial traversals of
 * ranges of bins (via an alternate Traverser constructor)
 * to support partitioned aggregate operations.  Also, read-only
 * operations give up if ever forwarded to a null table, which
 * provides support for shutdown-style clearing, which is also not
 * currently implemented.
 *
 * Lazy table initialization minimizes footprint until first use,
 * and also avoids resizings when the first operation is from a
 * putAll, constructor with map argument, or deserialization.
 * These cases attempt to override the initial capacity settings,
 * but harmlessly fail to take effect in cases of races.
 *
 * The element count is maintained using a specialization of
 * LongAdder. We need to incorporate a specialization rather than
 * just use a LongAdder in order to access implicit
 * contention-sensing that leads to creation of multiple
 * CounterCells.  The counter mechanics avoid contention on
 * updates but can encounter cache thrashing if read too
 * frequently during concurrent access. To avoid reading so often,
 * resizing under contention is attempted only upon adding to a
 * bin already holding two or more nodes. Under uniform hash
 * distributions, the probability of this occurring at threshold
 * is around 13%, meaning that only about 1 in 8 puts check
 * threshold (and after resizing, many fewer do so).
 *
 * TreeBins use a special form of comparison for search and
 * related operations (which is the main reason we cannot use
 * existing collections such as TreeMaps). TreeBins contain
 * Comparable elements, but may contain others, as well as
 * elements that are Comparable but not necessarily Comparable for
 * the same T, so we cannot invoke compareTo among them. To handle
 * this, the tree is ordered primarily by hash value, then by
 * Comparable.compareTo order if applicable.  On lookup at a node,
 * if elements are not comparable or compare as 0 then both left
 * and right children may need to be searched in the case of tied
 * hash values. (This corresponds to the full list search that
 * would be necessary if all elements were non-Comparable and had
 * tied hashes.) On insertion, to keep a total ordering (or as
 * close as is required here) across rebalancings, we compare
 * classes and identityHashCodes as tie-breakers. The red-black
 * balancing code is updated from pre-jdk-collections
 * (http://gee.cs.oswego.edu/dl/classes/collections/RBCell.java)
 * based in turn on Cormen, Leiserson, and Rivest "Introduction to
 * Algorithms" (CLR).
 *
 * TreeBins also require an additional locking mechanism.  While
 * list traversal is always possible by readers even during
 * updates, tree traversal is not, mainly because of tree-rotations
 * that may change the root node and/or its linkages.  TreeBins
 * include a simple read-write lock mechanism parasitic on the
 * main bin-synchronization strategy: Structural adjustments
 * associated with an insertion or removal are already bin-locked
 * (and so cannot conflict with other writers) but must wait for
 * ongoing readers to finish. Since there can be only one such
 * waiter, we use a simple scheme using a single "waiter" field to
 * block writers.  However, readers need never block.  If the root
 * lock is held, they proceed along the slow traversal path (via
 * next-pointers) until the lock becomes available or the list is
 * exhausted, whichever comes first. These cases are not fast, but
 * maximize aggregate expected throughput.
 *
 * Maintaining API and serialization compatibility with previous
 * versions of this class introduces several oddities. Mainly: We
 * leave untouched but unused constructor arguments referring to
 * concurrencyLevel. We accept a loadFactor constructor argument,
 * but apply it only to initial table capacity (which is the only
 * time that we can guarantee to honor it.) We also declare an
 * unused "Segment" class that is instantiated in minimal form
 * only when serializing.
 *
 * Also, solely for compatibility with previous versions of this
 * class, it extends AbstractMap, even though all of its methods
 * are overridden, so it is just useless baggage.
 *
 * This file is organized to make things a little easier to follow
 * while reading than they might otherwise: First the main static
 * declarations and utilities, then fields, then main public
 * methods (with a few factorings of multiple public methods into
 * internal ones), then sizing methods, trees, traversers, and
 * bulk operations.
 */

/**
 * The largest possible table capacity.  This value must be
 * exactly 1<<30 to stay within Java array allocation and indexing
 * bounds for power of two table sizes, and is further required
 * because the top two bits of 32bit hash fields are used for
 * control purposes.
 */
// 最大容量，此值必须为1<<30以保证在Java数组分配和索引范围的2的n次幂
// 整形2个高位用于控制目的所以不能使用
private static final int MAXIMUM_CAPACITY = 1 << 30;

// 表默认初始容量，表容量必须为2的n次幂，最小为1，最大为MAXIMUM_CAPACITY
private static final int DEFAULT_CAPACITY = 16;

/**
 * The largest possible (non-power of two) array size.
 * Needed by toArray and related methods.
 */
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

// 默认哈希表并发等级，此版本没有使用，为旧版本兼容而保留
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;

/**
 * The load factor for this table. Overrides of this value in
 * constructors affect only the initial table capacity.  The
 * actual floating point value isn't normally used -- it is
 * simpler to use expressions such as {@code n - (n >>> 2)} for
 * the associated resizing threshold.
 */
// 哈希表默认负载因子
private static final float LOAD_FACTOR = 0.75f;

/**
 * The bin count threshold for using a tree rather than list for a
 * bin.  Bins are converted to trees when adding an element to a
 * bin with at least this many nodes. The value must be greater
 * than 2, and should be at least 8 to mesh with assumptions in
 * tree removal about conversion back to plain bins upon
 * shrinkage.
 */
static final int TREEIFY_THRESHOLD = 8;

/**
 * The bin count threshold for untreeifying a (split) bin during a
 * resize operation. Should be less than TREEIFY_THRESHOLD, and at
 * most 6 to mesh with shrinkage detection under removal.
 */
// 二叉树退化为链表阈值
static final int UNTREEIFY_THRESHOLD = 6;

/**
 * The smallest table capacity for which bins may be treeified.
 * (Otherwise the table is resized if too many nodes in a bin.)
 * The value should be at least 4 * TREEIFY_THRESHOLD to avoid
 * conflicts between resizing and treeification thresholds.
 */
static final int MIN_TREEIFY_CAPACITY = 64;

/**
 * Minimum number of rebinnings per transfer step. Ranges are
 * subdivided to allow multiple resizer threads.  This value
 * serves as a lower bound to avoid resizers encountering
 * excessive memory contention.  The value should be at least
 * DEFAULT_CAPACITY.
 */
private static final int MIN_TRANSFER_STRIDE = 16;

/**
 * The number of bits used for generation stamp in sizeCtl.
 * Must be at least 6 for 32bit arrays.
 */
private static final int RESIZE_STAMP_BITS = 16;

/**
 * The maximum number of threads that can help resize.
 * Must fit in 32 - RESIZE_STAMP_BITS bits.
 */
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

/**
 * The bit shift for recording size stamp in sizeCtl.
 */
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

/*
 * Encodings for Node hash fields. See above for explanation.
 */
static final int MOVED     = -1; // hash for forwarding nodes
static final int TREEBIN   = -2; // hash for roots of trees
static final int RESERVED  = -3; // hash for transient reservations
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

// 处理器线程数
static final int NCPU = Runtime.getRuntime().availableProcessors();

/**
 * Serialized pseudo-fields, provided only for jdk7 compatibility.
 * @serialField segments Segment[]
 *   The segments, each of which is a specialized hash table.
 * @serialField segmentMask int
 *   Mask value for indexing into segments. The upper bits of a
 *   key's hash code are used to choose the segment.
 * @serialField segmentShift int
 *   Shift value for indexing within segments.
 */
private static final ObjectStreamField[] serialPersistentFields = {
    new ObjectStreamField("segments", Segment[].class),
    new ObjectStreamField("segmentMask", Integer.TYPE),
    new ObjectStreamField("segmentShift", Integer.TYPE),
};
```

## 三、Node

```java
/**
 * Key-value entry.  This class is never exported out as a
 * user-mutable Map.Entry (i.e., one supporting setValue; see
 * MapEntry below), but can be used for read-only traversals used
 * in bulk tasks.  Subclasses of Node with a negative hash field
 * are special, and contain null keys and values (but are never
 * exported).  Otherwise, keys and vals are never null.
 */
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;

    Node(int hash, K key, V val) {
        this.hash = hash;
        this.key = key;
        this.val = val;
    }

    Node(int hash, K key, V val, Node<K,V> next) {
        this(hash, key, val);
        this.next = next;
    }

    public final K getKey()     { return key; }
    public final V getValue()   { return val; }
    public final int hashCode() { return key.hashCode() ^ val.hashCode(); }
    public final String toString() {
        return Helpers.mapEntryToString(key, val);
    }
    public final V setValue(V value) {
        throw new UnsupportedOperationException();
    }

    public final boolean equals(Object o) {
        Object k, v, u; Map.Entry<?,?> e;
        return ((o instanceof Map.Entry) &&
                (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                (v = e.getValue()) != null &&
                (k == key || k.equals(key)) &&
                (v == (u = val) || v.equals(u)));
    }

    // 对map.get()的虚拟化支持，由子类重写
    Node<K,V> find(int h, Object k) {
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}
```

## 四、静态方法

#### 位分散

把哈希值`高16位`与`低16位`进行异或运算。

```java
/**
 * Spreads (XORs) higher bits of hash to lower and also forces top
 * bit to 0. Because the table uses power-of-two masking, sets of
 * hashes that vary only in bits above the current mask will
 * always collide. (Among known examples are sets of Float keys
 * holding consecutive whole numbers in small tables.)  So we
 * apply a transform that spreads the impact of higher bits
 * downward. There is a tradeoff between speed, utility, and
 * quality of bit-spreading. Because many common sets of hashes
 * are already reasonably distributed (so don't benefit from
 * spreading), and because we use trees to handle large sets of
 * collisions in bins, we just XOR some shifted bits in the
 * cheapest possible way to reduce systematic lossage, as well as
 * to incorporate impact of the highest bits that would otherwise
 * never be used in index calculations because of table bounds.
 */
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

下图展示hashCode高低位运算：

![HashMap_bit_hashCode](/img/java/HashMap_bit_hashCode.png)

这种运算方式有以下好处：

* 满足速度、功效、位分散质量等条件；
* 避免使用%操作的同时，令位运算发挥更好效果；
* 避免性能损耗，开销较低

#### 计算哈希表大小

通过给定值计算获得`相等或更大`且`大小为2^n`的数值：

```java
private static final int tableSizeFor(int c) {
    int n = -1 >>> Integer.numberOfLeadingZeros(c - 1);
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

下面假设cap为15，通过`tableSizeFor()`计算：

![HashMap_tableSizeFor](/img/java/HashMap_tableSizeFor.png)

从图中看出，当流程进行到`n|=n>>>4`，后续步骤运算结果已经固定不变。

```java
// 如果对象实现Comparable<C>接口，返回其类型，否则返回null
static Class<?> comparableClassFor(Object x) {
    if (x instanceof Comparable) {
        Class<?> c; Type[] ts, as; ParameterizedType p;
        if ((c = x.getClass()) == String.class) // bypass checks
            return c;
        if ((ts = c.getGenericInterfaces()) != null) {
            for (Type t : ts) {
                if ((t instanceof ParameterizedType) &&
                    ((p = (ParameterizedType)t).getRawType() ==
                     Comparable.class) &&
                    (as = p.getActualTypeArguments()) != null &&
                    as.length == 1 && as[0] == c) // type arg is c
                    return c;
            }
        }
    }
    return null;
}

/**
 * Returns k.compareTo(x) if x matches kc (k's screened comparable
 * class), else 0.
 */
@SuppressWarnings({"rawtypes","unchecked"}) // for cast to Comparable
static int compareComparables(Class<?> kc, Object k, Object x) {
    return (x == null || x.getClass() != kc ? 0 :
            ((Comparable)k).compareTo(x));
}

/* ---------------- Table element access -------------- */

/*
 * Atomic access methods are used for table elements as well as
 * elements of in-progress next table while resizing.  All uses of
 * the tab arguments must be null checked by callers.  All callers
 * also paranoically precheck that tab's length is not zero (or an
 * equivalent check), thus ensuring that any index argument taking
 * the form of a hash value anded with (length - 1) is a valid
 * index.  Note that, to be correct wrt arbitrary concurrency
 * errors by users, these checks must operate on local variables,
 * which accounts for some odd-looking inline assignments below.
 * Note that calls to setTabAt always occur within locked regions,
 * and so require only release ordering.
 */

@SuppressWarnings("unchecked")
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectAcquire(tab, ((long)i << ASHIFT) + ABASE);
}

static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSetObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}

static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectRelease(tab, ((long)i << ASHIFT) + ABASE, v);
}
```

## 五、成员变量

```java
// 哈希桶表，长度为2的n次幂，不超过MAXIMUM_CAPACITY
// 首次插入时进行懒加载，变量能被iterators直接访问
transient volatile Node<K,V>[] table;

// 下个可被使用的表，仅在resizing过程中为非空
private transient volatile Node<K,V>[] nextTable;

/**
 * Base counter value, used mainly when there is no contention,
 * but also as a fallback during table initialization
 * races. Updated via CAS.
 */
private transient volatile long baseCount;

/**
 * Table initialization and resizing control.  When negative, the
 * table is being initialized or resized: -1 for initialization,
 * else -(1 + the number of active resizing threads).  Otherwise,
 * when table is null, holds the initial table size to use upon
 * creation, or 0 for default. After initialization, holds the
 * next element count value upon which to resize the table.
 */
private transient volatile int sizeCtl;

// resizing时分割的下个表索引值(+1)
private transient volatile int transferIndex;

// resizing 和/或 创建CounterCells时由自旋锁使用(通过CAS上锁)
private transient volatile int cellsBusy;

// CounterCell表。非空是大小为为2的n次幂
private transient volatile CounterCell[] counterCells;
```

## 六、构造方法

```java
// 构建空Map，默认容量为16，默认负载因子为0.75
public ConcurrentHashMap() {
}

// 通过指定的初始容量构造Map，避免后期多次动态扩容
// 若initialCapacity为负数抛出IllegalArgumentException
public ConcurrentHashMap(int initialCapacity) {
    this(initialCapacity, LOAD_FACTOR, 1);
}

// 通过给定Map创建新的Map，默认负载因子为0.75，容量值为足够保存m中键值对的大小
// 若m为null抛出NullPointerException
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}

// 通过指定初始容量、负载因子构造Map
// 若initialCapacity为负数或loadFactor不是非负数，抛出IllegalArgumentException
public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}

// 通过指定初始容量、负载因子、和并发更新线程数构造Map
// 若initialCapacity为负数，抛出IllegalArgumentException
// 若loadFactor或concurrencyLevel不是非负数，抛出IllegalArgumentException
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

## 七、成员方法

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

// 检查map映射数量是否为空
public boolean isEmpty() {
    return sumCount() <= 0L; // ignore transient negative values
}

// 通过指定key获取映射value，若映射不存在返回null
// 若key为空抛出NullPointerException
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}

// 检查指定key是否为保存在哈希表中的键
// 若指定key为空抛出NullPointerException
public boolean containsKey(Object key) {
    return get(key) != null;
}

// 检查指定value是否由表中一个或多个key所映射
// 方法可能需要对哈希表进行全遍历，因此会比方法containsKey慢很多
// 若指定value为空抛出NullPointerException
public boolean containsValue(Object value) {
    if (value == null)
        throw new NullPointerException();
    Node<K,V>[] t;
    if ((t = table) != null) {
        Traverser<K,V> it = new Traverser<K,V>(t, t.length, 0, t.length);
        for (Node<K,V> p; (p = it.advance()) != null; ) {
            V v;
            if ((v = p.val) == value || (v != null && value.equals(v)))
                return true;
        }
    }
    return false;
}

// 向map中存入键值对，若指定key或value为空抛出NullPointerException
public V put(K key, V value) {
    return putVal(key, value, false);
}

// put()和putIfAbsent()的实现
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // key或value为空抛出NullPointerException
    if (key == null || value == null) throw new NullPointerException();
    // key的哈希值还做了一次高低位操作
    int hash = spread(key.hashCode());
    // 哈希桶数量
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh; K fk; V fv;
        if (tab == null || (n = tab.length) == 0)
            // 哈希表不存在或长度为0，则需要初始化哈希表
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else if (onlyIfAbsent // 检查第一个节点时不获取锁
                 && fh == hash
                 && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                 && (fv = f.val) != null)
            return fv;
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}

// 从指定map中拷贝所有键值对到此map。若对应key已有值，则新值替换旧值
public void putAll(Map<? extends K, ? extends V> m) {
    tryPresize(m.size());
    for (Map.Entry<? extends K, ? extends V> e : m.entrySet())
        putVal(e.getKey(), e.getValue(), false);
}

// 根据key移除对应键值对，若价值对不存在则不做处理，否则返回key的映射值
// 若key为空则抛出NullPointerException
public V remove(Object key) {
    return replaceNode(key, null, null);
}

/**
 * Implementation for the four public remove/replace methods:
 * Replaces node value with v, conditional upon match of cv if
 * non-null.  If resulting value is null, delete.
 */
// 4个remove()/replace()方法的具体实现
final V replaceNode(Object key, V value, Object cv) {
    int hash = spread(key.hashCode());
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0 ||
            (f = tabAt(tab, i = (n - 1) & hash)) == null)
            break;
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            boolean validated = false;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        validated = true;
                        for (Node<K,V> e = f, pred = null;;) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                V ev = e.val;
                                if (cv == null || cv == ev ||
                                    (ev != null && cv.equals(ev))) {
                                    oldVal = ev;
                                    if (value != null)
                                        e.val = value;
                                    else if (pred != null)
                                        pred.next = e.next;
                                    else
                                        setTabAt(tab, i, e.next);
                                }
                                break;
                            }
                            pred = e;
                            if ((e = e.next) == null)
                                break;
                        }
                    }
                    else if (f instanceof TreeBin) {
                        validated = true;
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> r, p;
                        if ((r = t.root) != null &&
                            (p = r.findTreeNode(hash, key, null)) != null) {
                            V pv = p.val;
                            if (cv == null || cv == pv ||
                                (pv != null && cv.equals(pv))) {
                                oldVal = pv;
                                if (value != null)
                                    p.val = value;
                                else if (t.removeTreeNode(p))
                                    setTabAt(tab, i, untreeify(t.first));
                            }
                        }
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            if (validated) {
                if (oldVal != null) {
                    if (value == null)
                        addCount(-1L, -1);
                    return oldVal;
                }
                break;
            }
        }
    }
    return null;
}

// 把map中所有键值对映射移除
public void clear() {
    long delta = 0L; // negative number of deletions
    int i = 0;
    Node<K,V>[] tab = table;
    while (tab != null && i < tab.length) {
        int fh;
        Node<K,V> f = tabAt(tab, i);
        if (f == null)
            ++i;
        else if ((fh = f.hash) == MOVED) {
            tab = helpTransfer(tab, f);
            i = 0; // restart
        }
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> p = (fh >= 0 ? f :
                                   (f instanceof TreeBin) ?
                                   ((TreeBin<K,V>)f).first : null);
                    while (p != null) {
                        --delta;
                        p = p.next;
                    }
                    setTabAt(tab, i++, null);
                }
            }
        }
    }
    if (delta != 0L)
        addCount(delta, -1);
}

/**
 * Returns the hash code value for this {@link Map}, i.e.,
 * the sum of, for each key-value pair in the map,
 * {@code key.hashCode() ^ value.hashCode()}.
 *
 * @return the hash code value for this map
 */
public int hashCode() {
    int h = 0;
    Node<K,V>[] t;
    if ((t = table) != null) {
        Traverser<K,V> it = new Traverser<K,V>(t, t.length, 0, t.length);
        for (Node<K,V> p; (p = it.advance()) != null; )
            h += p.key.hashCode() ^ p.val.hashCode();
    }
    return h;
}

/**
 * Compares the specified object with this map for equality.
 * Returns {@code true} if the given object is a map with the same
 * mappings as this map.  This operation may return misleading
 * results if either map is concurrently modified during execution
 * of this method.
 *
 * @param o object to be compared for equality with this map
 * @return {@code true} if the specified object is equal to this map
 */
public boolean equals(Object o) {
    if (o != this) {
        if (!(o instanceof Map))
            return false;
        Map<?,?> m = (Map<?,?>) o;
        Node<K,V>[] t;
        int f = (t = table) == null ? 0 : t.length;
        Traverser<K,V> it = new Traverser<K,V>(t, f, 0, f);
        for (Node<K,V> p; (p = it.advance()) != null; ) {
            V val = p.val;
            Object v = m.get(p.key);
            if (v == null || (v != val && !v.equals(val)))
                return false;
        }
        for (Map.Entry<?,?> e : m.entrySet()) {
            Object mk, mv, v;
            if ((mk = e.getKey()) == null ||
                (mv = e.getValue()) == null ||
                (v = get(mk)) == null ||
                (mv != v && !mv.equals(v)))
                return false;
        }
    }
    return true;
}

/**
 * Stripped-down version of helper class used in previous version,
 * declared for the sake of serialization compatibility.
 */
static class Segment<K,V> extends ReentrantLock implements Serializable {
    private static final long serialVersionUID = 2249069246763182397L;
    final float loadFactor;
    Segment(float lf) { this.loadFactor = lf; }
}

// 若key映射值不存在就存入value作为值，若旧值存在则不做处理
// key或value为空抛出NullPointerException
public V putIfAbsent(K key, V value) {
    return putVal(key, value, true);
}

// 通过指定key和value移除键值对
// 若key为空抛出NullPointerException
public boolean remove(Object key, Object value) {
    if (key == null)
        throw new NullPointerException();
    return value != null && replaceNode(key, null, value) != null;
}

// 用指定newValue替换key已映射的值oldValue
// key、oldValue或newValue抛出NullPointerException
public boolean replace(K key, V oldValue, V newValue) {
    if (key == null || oldValue == null || newValue == null)
        throw new NullPointerException();
    return replaceNode(key, newValue, oldValue) != null;
}

// 用value替换指定key的旧值
// key或value为空抛出NullPointerException
public V replace(K key, V value) {
    if (key == null || value == null)
        throw new NullPointerException();
    return replaceNode(key, value, null);
}

// 返回指定key映射的value，如果key没有对应映射值，则返回defaultValue
// 若key为空抛出NullPointerException
public V getOrDefault(Object key, V defaultValue) {
    V v;
    return (v = get(key)) == null ? defaultValue : v;
}

/**
 * Tests if some key maps into the specified value in this table.
 *
 * <p>Note that this method is identical in functionality to
 * {@link #containsValue(Object)}, and exists solely to ensure
 * full compatibility with class {@link java.util.Hashtable},
 * which supported this method prior to introduction of the
 * Java Collections Framework.
 *
 * @param  value a value to search for
 * @return {@code true} if and only if some key maps to the
 *         {@code value} argument in this table as
 *         determined by the {@code equals} method;
 *         {@code false} otherwise
 * @throws NullPointerException if the specified value is null
 */
public boolean contains(Object value) {
    return containsValue(value);
}

// ConcurrentHashMap-only methods

/**
 * Returns the number of mappings. This method should be used
 * instead of {@link #size} because a ConcurrentHashMap may
 * contain more mappings than can be represented as an int. The
 * value returned is an estimate; the actual count may differ if
 * there are concurrent insertions or removals.
 *
 * @return the number of mappings
 * @since 1.8
 */
public long mappingCount() {
    long n = sumCount();
    return (n < 0L) ? 0L : n; // ignore transient negative values
}

/* ---------------- Table Initialization and Resizing -------------- */

/**
 * Returns the stamp bits for resizing a table of size n.
 * Must be negative when shifted left by RESIZE_STAMP_SHIFT.
 */
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}

// 使用保存在sizeCtl的值初始化表
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}

/**
 * Adds to count, and if table is too small and not already
 * resizing, initiates transfer. If already resizing, helps
 * perform transfer if work is available.  Rechecks occupancy
 * after a transfer to see if another resize is already needed
 * because resizings are lagging additions.
 *
 * @param x the count to add
 * @param check if <0, don't check resize, if <= 1 only check if uncontended
 */
private final void addCount(long x, int check) {
    CounterCell[] cs; long b, s;
    if ((cs = counterCells) != null ||
        !U.compareAndSetLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell c; long v; int m;
        boolean uncontended = true;
        if (cs == null || (m = cs.length - 1) < 0 ||
            (c = cs[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSetInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}

/**
 * Helps transfer if a resize is in progress.
 */
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}

/**
 * Tries to presize table to accommodate the given number of elements.
 *
 * @param size number of elements (doesn't need to be perfectly accurate)
 */
private final void tryPresize(int size) {
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            int rs = resizeStamp(n);
            if (U.compareAndSetInt(this, SIZECTL, sc,
                                    (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}

/**
 * Moves and/or copies the nodes in each bin to new table. See
 * above for explanation.
 */
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSetInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            if (U.compareAndSetInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)
            advance = true; // 已完成处理
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}

/* ---------------- Counter support -------------- */

/**
 * A padded cell for distributing counts.  Adapted from LongAdder
 * and Striped64.  See their internal docs for explanation.
 */
@jdk.internal.vm.annotation.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}

final long sumCount() {
    CounterCell[] cs = counterCells;
    long sum = baseCount;
    if (cs != null) {
        for (CounterCell c : cs)
            if (c != null)
                sum += c.value;
    }
    return sum;
}

// See LongAdder version for explanation
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();      // force initialization
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] cs; CounterCell c; int n; long v;
        if ((cs = counterCells) != null && (n = cs.length) > 0) {
            if ((c = cs[(n - 1) & h]) == null) {
                if (cellsBusy == 0) {            // Try to attach new Cell
                    CounterCell r = new CounterCell(x); // Optimistic create
                    if (cellsBusy == 0 &&
                        U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            else if (U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))
                break;
            else if (counterCells != cs || n >= NCPU)
                collide = false;            // At max size or stale
            else if (!collide)
                collide = true;
            else if (cellsBusy == 0 &&
                     U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
                try {
                    if (counterCells == cs) // Expand table unless stale
                        counterCells = Arrays.copyOf(cs, n << 1);
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            h = ThreadLocalRandom.advanceProbe(h);
        }
        else if (cellsBusy == 0 && counterCells == cs &&
                 U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {                           // Initialize table
                if (counterCells == cs) {
                    CounterCell[] rs = new CounterCell[2];
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
        }
        else if (U.compareAndSetLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```

## 链树互转

```java
/**
 * Replaces all linked nodes in bin at given index unless table is
 * too small, in which case resizes instead.
 */
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}

/**
 * Returns a list of non-TreeNodes replacing those in given list.
 */
static <K,V> Node<K,V> untreeify(Node<K,V> b) {
    Node<K,V> hd = null, tl = null;
    for (Node<K,V> q = b; q != null; q = q.next) {
        Node<K,V> p = new Node<K,V>(q.hash, q.key, q.val);
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    return hd;
}
```

## 八、TreeNode

```java
/**
 * Nodes for use in TreeBins.
 */
static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;

    TreeNode(int hash, K key, V val, Node<K,V> next,
             TreeNode<K,V> parent) {
        super(hash, key, val, next);
        this.parent = parent;
    }

    Node<K,V> find(int h, Object k) {
        return findTreeNode(h, k, null);
    }

    /**
     * Returns the TreeNode (or null if not found) for the given key
     * starting at given root.
     */
    final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {
        if (k != null) {
            TreeNode<K,V> p = this;
            do {
                int ph, dir; K pk; TreeNode<K,V> q;
                TreeNode<K,V> pl = p.left, pr = p.right;
                if ((ph = p.hash) > h)
                    p = pl;
                else if (ph < h)
                    p = pr;
                else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                    return p;
                else if (pl == null)
                    p = pr;
                else if (pr == null)
                    p = pl;
                else if ((kc != null ||
                          (kc = comparableClassFor(k)) != null) &&
                         (dir = compareComparables(kc, k, pk)) != 0)
                    p = (dir < 0) ? pl : pr;
                else if ((q = pr.findTreeNode(h, k, kc)) != null)
                    return q;
                else
                    p = pl;
            } while (p != null);
        }
        return null;
    }
}
```

## 九、TreeBin

```java
/**
 * TreeNodes used at the heads of bins. TreeBins do not hold user
 * keys or values, but instead point to list of TreeNodes and
 * their root. They also maintain a parasitic read-write lock
 * forcing writers (who hold bin lock) to wait for readers (who do
 * not) to complete before tree restructuring operations.
 */
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root;
    volatile TreeNode<K,V> first;
    volatile Thread waiter;
    volatile int lockState;
    // values for lockState
    static final int WRITER = 1; // set while holding write lock
    static final int WAITER = 2; // set when waiting for write lock
    static final int READER = 4; // increment value for setting read lock

    /**
     * Tie-breaking utility for ordering insertions when equal
     * hashCodes and non-comparable. We don't require a total
     * order, just a consistent insertion rule to maintain
     * equivalence across rebalancings. Tie-breaking further than
     * necessary simplifies testing a bit.
     */
    static int tieBreakOrder(Object a, Object b) {
        int d;
        if (a == null || b == null ||
            (d = a.getClass().getName().
             compareTo(b.getClass().getName())) == 0)
            d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
                 -1 : 1);
        return d;
    }

    /**
     * Creates bin with initial set of nodes headed by b.
     */
    TreeBin(TreeNode<K,V> b) {
        super(TREEBIN, null, null);
        this.first = b;
        TreeNode<K,V> r = null;
        for (TreeNode<K,V> x = b, next; x != null; x = next) {
            next = (TreeNode<K,V>)x.next;
            x.left = x.right = null;
            if (r == null) {
                x.parent = null;
                x.red = false;
                r = x;
            }
            else {
                K k = x.key;
                int h = x.hash;
                Class<?> kc = null;
                for (TreeNode<K,V> p = r;;) {
                    int dir, ph;
                    K pk = p.key;
                    if ((ph = p.hash) > h)
                        dir = -1;
                    else if (ph < h)
                        dir = 1;
                    else if ((kc == null &&
                              (kc = comparableClassFor(k)) == null) ||
                             (dir = compareComparables(kc, k, pk)) == 0)
                        dir = tieBreakOrder(k, pk);
                    TreeNode<K,V> xp = p;
                    if ((p = (dir <= 0) ? p.left : p.right) == null) {
                        x.parent = xp;
                        if (dir <= 0)
                            xp.left = x;
                        else
                            xp.right = x;
                        r = balanceInsertion(r, x);
                        break;
                    }
                }
            }
        }
        this.root = r;
        assert checkInvariants(root);
    }

    /**
     * Acquires write lock for tree restructuring.
     */
    private final void lockRoot() {
        if (!U.compareAndSetInt(this, LOCKSTATE, 0, WRITER))
            contendedLock(); // offload to separate method
    }

    /**
     * Releases write lock for tree restructuring.
     */
    private final void unlockRoot() {
        lockState = 0;
    }

    /**
     * Possibly blocks awaiting root lock.
     */
    private final void contendedLock() {
        boolean waiting = false;
        for (int s;;) {
            if (((s = lockState) & ~WAITER) == 0) {
                if (U.compareAndSetInt(this, LOCKSTATE, s, WRITER)) {
                    if (waiting)
                        waiter = null;
                    return;
                }
            }
            else if ((s & WAITER) == 0) {
                if (U.compareAndSetInt(this, LOCKSTATE, s, s | WAITER)) {
                    waiting = true;
                    waiter = Thread.currentThread();
                }
            }
            else if (waiting)
                LockSupport.park(this);
        }
    }

    /**
     * Returns matching node or null if none. Tries to search
     * using tree comparisons from root, but continues linear
     * search when lock not available.
     */
    final Node<K,V> find(int h, Object k) {
        if (k != null) {
            for (Node<K,V> e = first; e != null; ) {
                int s; K ek;
                if (((s = lockState) & (WAITER|WRITER)) != 0) {
                    if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                    e = e.next;
                }
                else if (U.compareAndSetInt(this, LOCKSTATE, s,
                                             s + READER)) {
                    TreeNode<K,V> r, p;
                    try {
                        p = ((r = root) == null ? null :
                             r.findTreeNode(h, k, null));
                    } finally {
                        Thread w;
                        if (U.getAndAddInt(this, LOCKSTATE, -READER) ==
                            (READER|WAITER) && (w = waiter) != null)
                            LockSupport.unpark(w);
                    }
                    return p;
                }
            }
        }
        return null;
    }

    /**
     * Finds or adds a node.
     * @return null if added
     */
    // 查找或添加节点，若已添加添加返回null
    final TreeNode<K,V> putTreeVal(int h, K k, V v) {
        Class<?> kc = null;
        boolean searched = false;
        for (TreeNode<K,V> p = root;;) {
            int dir, ph; K pk;
            if (p == null) {
                first = root = new TreeNode<K,V>(h, k, v, null, null);
                break;
            }
            else if ((ph = p.hash) > h)
                dir = -1;
            else if (ph < h)
                dir = 1;
            else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                return p;
            else if ((kc == null &&
                      (kc = comparableClassFor(k)) == null) ||
                     (dir = compareComparables(kc, k, pk)) == 0) {
                if (!searched) {
                    TreeNode<K,V> q, ch;
                    searched = true;
                    if (((ch = p.left) != null &&
                         (q = ch.findTreeNode(h, k, kc)) != null) ||
                        ((ch = p.right) != null &&
                         (q = ch.findTreeNode(h, k, kc)) != null))
                        return q;
                }
                dir = tieBreakOrder(k, pk);
            }

            TreeNode<K,V> xp = p;
            if ((p = (dir <= 0) ? p.left : p.right) == null) {
                TreeNode<K,V> x, f = first;
                first = x = new TreeNode<K,V>(h, k, v, f, xp);
                if (f != null)
                    f.prev = x;
                if (dir <= 0)
                    xp.left = x;
                else
                    xp.right = x;
                if (!xp.red)
                    x.red = true;
                else {
                    lockRoot();
                    try {
                        root = balanceInsertion(root, x);
                    } finally {
                        unlockRoot();
                    }
                }
                break;
            }
        }
        assert checkInvariants(root);
        return null;
    }

    /**
     * Removes the given node, that must be present before this
     * call.  This is messier than typical red-black deletion code
     * because we cannot swap the contents of an interior node
     * with a leaf successor that is pinned by "next" pointers
     * that are accessible independently of lock. So instead we
     * swap the tree linkages.
     *
     * @return true if now too small, so should be untreeified
     */
    final boolean removeTreeNode(TreeNode<K,V> p) {
        TreeNode<K,V> next = (TreeNode<K,V>)p.next;
        TreeNode<K,V> pred = p.prev;  // unlink traversal pointers
        TreeNode<K,V> r, rl;
        if (pred == null)
            first = next;
        else
            pred.next = next;
        if (next != null)
            next.prev = pred;
        if (first == null) {
            root = null;
            return true;
        }
        if ((r = root) == null || r.right == null || // too small
            (rl = r.left) == null || rl.left == null)
            return true;
        lockRoot();
        try {
            TreeNode<K,V> replacement;
            TreeNode<K,V> pl = p.left;
            TreeNode<K,V> pr = p.right;
            if (pl != null && pr != null) {
                TreeNode<K,V> s = pr, sl;
                while ((sl = s.left) != null) // find successor
                    s = sl;
                boolean c = s.red; s.red = p.red; p.red = c; // swap colors
                TreeNode<K,V> sr = s.right;
                TreeNode<K,V> pp = p.parent;
                if (s == pr) { // p was s's direct parent
                    p.parent = s;
                    s.right = p;
                }
                else {
                    TreeNode<K,V> sp = s.parent;
                    if ((p.parent = sp) != null) {
                        if (s == sp.left)
                            sp.left = p;
                        else
                            sp.right = p;
                    }
                    if ((s.right = pr) != null)
                        pr.parent = s;
                }
                p.left = null;
                if ((p.right = sr) != null)
                    sr.parent = p;
                if ((s.left = pl) != null)
                    pl.parent = s;
                if ((s.parent = pp) == null)
                    r = s;
                else if (p == pp.left)
                    pp.left = s;
                else
                    pp.right = s;
                if (sr != null)
                    replacement = sr;
                else
                    replacement = p;
            }
            else if (pl != null)
                replacement = pl;
            else if (pr != null)
                replacement = pr;
            else
                replacement = p;
            if (replacement != p) {
                TreeNode<K,V> pp = replacement.parent = p.parent;
                if (pp == null)
                    r = replacement;
                else if (p == pp.left)
                    pp.left = replacement;
                else
                    pp.right = replacement;
                p.left = p.right = p.parent = null;
            }

            root = (p.red) ? r : balanceDeletion(r, replacement);

            if (p == replacement) {  // detach pointers
                TreeNode<K,V> pp;
                if ((pp = p.parent) != null) {
                    if (p == pp.left)
                        pp.left = null;
                    else if (p == pp.right)
                        pp.right = null;
                    p.parent = null;
                }
            }
        } finally {
            unlockRoot();
        }
        assert checkInvariants(root);
        return false;
    }

    /* ------------------------------------------------------------ */
    // Red-black tree methods, all adapted from CLR

    static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                          TreeNode<K,V> p) {
        TreeNode<K,V> r, pp, rl;
        if (p != null && (r = p.right) != null) {
            if ((rl = p.right = r.left) != null)
                rl.parent = p;
            if ((pp = r.parent = p.parent) == null)
                (root = r).red = false;
            else if (pp.left == p)
                pp.left = r;
            else
                pp.right = r;
            r.left = p;
            p.parent = r;
        }
        return root;
    }

    static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                           TreeNode<K,V> p) {
        TreeNode<K,V> l, pp, lr;
        if (p != null && (l = p.left) != null) {
            if ((lr = p.left = l.right) != null)
                lr.parent = p;
            if ((pp = l.parent = p.parent) == null)
                (root = l).red = false;
            else if (pp.right == p)
                pp.right = l;
            else
                pp.left = l;
            l.right = p;
            p.parent = l;
        }
        return root;
    }

    static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                TreeNode<K,V> x) {
        x.red = true;
        for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
            if ((xp = x.parent) == null) {
                x.red = false;
                return x;
            }
            else if (!xp.red || (xpp = xp.parent) == null)
                return root;
            if (xp == (xppl = xpp.left)) {
                if ((xppr = xpp.right) != null && xppr.red) {
                    xppr.red = false;
                    xp.red = false;
                    xpp.red = true;
                    x = xpp;
                }
                else {
                    if (x == xp.right) {
                        root = rotateLeft(root, x = xp);
                        xpp = (xp = x.parent) == null ? null : xp.parent;
                    }
                    if (xp != null) {
                        xp.red = false;
                        if (xpp != null) {
                            xpp.red = true;
                            root = rotateRight(root, xpp);
                        }
                    }
                }
            }
            else {
                if (xppl != null && xppl.red) {
                    xppl.red = false;
                    xp.red = false;
                    xpp.red = true;
                    x = xpp;
                }
                else {
                    if (x == xp.left) {
                        root = rotateRight(root, x = xp);
                        xpp = (xp = x.parent) == null ? null : xp.parent;
                    }
                    if (xp != null) {
                        xp.red = false;
                        if (xpp != null) {
                            xpp.red = true;
                            root = rotateLeft(root, xpp);
                        }
                    }
                }
            }
        }
    }

    static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root,
                                               TreeNode<K,V> x) {
        for (TreeNode<K,V> xp, xpl, xpr;;) {
            if (x == null || x == root)
                return root;
            else if ((xp = x.parent) == null) {
                x.red = false;
                return x;
            }
            else if (x.red) {
                x.red = false;
                return root;
            }
            else if ((xpl = xp.left) == x) {
                if ((xpr = xp.right) != null && xpr.red) {
                    xpr.red = false;
                    xp.red = true;
                    root = rotateLeft(root, xp);
                    xpr = (xp = x.parent) == null ? null : xp.right;
                }
                if (xpr == null)
                    x = xp;
                else {
                    TreeNode<K,V> sl = xpr.left, sr = xpr.right;
                    if ((sr == null || !sr.red) &&
                        (sl == null || !sl.red)) {
                        xpr.red = true;
                        x = xp;
                    }
                    else {
                        if (sr == null || !sr.red) {
                            if (sl != null)
                                sl.red = false;
                            xpr.red = true;
                            root = rotateRight(root, xpr);
                            xpr = (xp = x.parent) == null ?
                                null : xp.right;
                        }
                        if (xpr != null) {
                            xpr.red = (xp == null) ? false : xp.red;
                            if ((sr = xpr.right) != null)
                                sr.red = false;
                        }
                        if (xp != null) {
                            xp.red = false;
                            root = rotateLeft(root, xp);
                        }
                        x = root;
                    }
                }
            }
            else { // symmetric
                if (xpl != null && xpl.red) {
                    xpl.red = false;
                    xp.red = true;
                    root = rotateRight(root, xp);
                    xpl = (xp = x.parent) == null ? null : xp.left;
                }
                if (xpl == null)
                    x = xp;
                else {
                    TreeNode<K,V> sl = xpl.left, sr = xpl.right;
                    if ((sl == null || !sl.red) &&
                        (sr == null || !sr.red)) {
                        xpl.red = true;
                        x = xp;
                    }
                    else {
                        if (sl == null || !sl.red) {
                            if (sr != null)
                                sr.red = false;
                            xpl.red = true;
                            root = rotateLeft(root, xpl);
                            xpl = (xp = x.parent) == null ?
                                null : xp.left;
                        }
                        if (xpl != null) {
                            xpl.red = (xp == null) ? false : xp.red;
                            if ((sl = xpl.left) != null)
                                sl.red = false;
                        }
                        if (xp != null) {
                            xp.red = false;
                            root = rotateRight(root, xp);
                        }
                        x = root;
                    }
                }
            }
        }
    }

    /**
     * Checks invariants recursively for the tree of Nodes rooted at t.
     */
    static <K,V> boolean checkInvariants(TreeNode<K,V> t) {
        TreeNode<K,V> tp = t.parent, tl = t.left, tr = t.right,
            tb = t.prev, tn = (TreeNode<K,V>)t.next;
        if (tb != null && tb.next != t)
            return false;
        if (tn != null && tn.prev != t)
            return false;
        if (tp != null && t != tp.left && t != tp.right)
            return false;
        if (tl != null && (tl.parent != t || tl.hash > t.hash))
            return false;
        if (tr != null && (tr.parent != t || tr.hash < t.hash))
            return false;
        if (t.red && tl != null && tl.red && tr != null && tr.red)
            return false;
        if (tl != null && !checkInvariants(tl))
            return false;
        if (tr != null && !checkInvariants(tr))
            return false;
        return true;
    }

    private static final Unsafe U = Unsafe.getUnsafe();
    private static final long LOCKSTATE
            = U.objectFieldOffset(TreeBin.class, "lockState");
}
```

## 十、Unsafe机制

```java
private static final Unsafe U = Unsafe.getUnsafe();
private static final long SIZECTL;
private static final long TRANSFERINDEX;
private static final long BASECOUNT;
private static final long CELLSBUSY;
private static final long CELLVALUE;
private static final int ABASE;
private static final int ASHIFT;

static {
    SIZECTL = U.objectFieldOffset
        (ConcurrentHashMap.class, "sizeCtl");
    TRANSFERINDEX = U.objectFieldOffset
        (ConcurrentHashMap.class, "transferIndex");
    BASECOUNT = U.objectFieldOffset
        (ConcurrentHashMap.class, "baseCount");
    CELLSBUSY = U.objectFieldOffset
        (ConcurrentHashMap.class, "cellsBusy");

    CELLVALUE = U.objectFieldOffset
        (CounterCell.class, "value");

    ABASE = U.arrayBaseOffset(Node[].class);
    int scale = U.arrayIndexScale(Node[].class);
    if ((scale & (scale - 1)) != 0)
        throw new ExceptionInInitializerError("array index scale not a power of two");
    ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);

    // Reduce the risk of rare disastrous classloading in first call to
    // LockSupport.park: https://bugs.openjdk.java.net/browse/JDK-8074773
    Class<?> ensureLoaded = LockSupport.class;

    // Eager class load observed to help JIT during startup
    ensureLoaded = ReservationNode.class;
}
```
