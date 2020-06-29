---
layout:     post
title:      "ConcurrentLinkedQueue"
date:       2018-10-21
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
 * An unbounded thread-safe {@linkplain Queue queue} based on linked nodes.
 * This queue orders elements FIFO (first-in-first-out).
 * The <em>head</em> of the queue is that element that has been on the
 * queue the longest time.
 * The <em>tail</em> of the queue is that element that has been on the
 * queue the shortest time. New elements
 * are inserted at the tail of the queue, and the queue retrieval
 * operations obtain elements at the head of the queue.
 * A {@code ConcurrentLinkedQueue} is an appropriate choice when
 * many threads will share access to a common collection.
 * Like most other concurrent collection implementations, this class
 * does not permit the use of {@code null} elements.
 *
 * <p>This implementation employs an efficient <em>non-blocking</em>
 * algorithm based on one described in
 * <a href="http://www.cs.rochester.edu/~scott/papers/1996_PODC_queues.pdf">
 * Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue
 * Algorithms</a> by Maged M. Michael and Michael L. Scott.
 *
 * <p>Iterators are <i>weakly consistent</i>, returning elements
 * reflecting the state of the queue at some point at or since the
 * creation of the iterator.  They do <em>not</em> throw {@link
 * java.util.ConcurrentModificationException}, and may proceed concurrently
 * with other operations.  Elements contained in the queue since the creation
 * of the iterator will be returned exactly once.
 *
 * <p>Beware that, unlike in most collections, the {@code size} method
 * is <em>NOT</em> a constant-time operation. Because of the
 * asynchronous nature of these queues, determining the current number
 * of elements requires a traversal of the elements, and so may report
 * inaccurate results if this collection is modified during traversal.
 *
 * <p>Bulk operations that add, remove, or examine multiple elements,
 * such as {@link #addAll}, {@link #removeIf} or {@link #forEach},
 * are <em>not</em> guaranteed to be performed atomically.
 * For example, a {@code forEach} traversal concurrent with an {@code
 * addAll} operation might observe only some of the added elements.
 *
 * <p>This class and its iterator implement all of the <em>optional</em>
 * methods of the {@link Queue} and {@link Iterator} interfaces.
 *
 * <p>Memory consistency effects: As with other concurrent
 * collections, actions in a thread prior to placing an object into a
 * {@code ConcurrentLinkedQueue}
 * <a href="package-summary.html#MemoryVisibility"><i>happen-before</i></a>
 * actions subsequent to the access or removal of that element from
 * the {@code ConcurrentLinkedQueue} in another thread.
 *
 * <p>This class is a member of the
 * <a href="{@docRoot}/java.base/java/util/package-summary.html#CollectionsFramework">
 * Java Collections Framework</a>.
 *
 * @since 1.5
 * @author Doug Lea
 * @param <E> the type of elements held in this queue
 */
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E>
        implements Queue<E>, java.io.Serializable
```

## 二、节点

```java
/*
 * This is a modification of the Michael & Scott algorithm,
 * adapted for a garbage-collected environment, with support for
 * interior node deletion (to support e.g. remove(Object)).  For
 * explanation, read the paper.
 *
 * Note that like most non-blocking algorithms in this package,
 * this implementation relies on the fact that in garbage
 * collected systems, there is no possibility of ABA problems due
 * to recycled nodes, so there is no need to use "counted
 * pointers" or related techniques seen in versions used in
 * non-GC'ed settings.
 *
 * The fundamental invariants are:
 * - There is exactly one (last) Node with a null next reference,
 *   which is CASed when enqueueing.  This last Node can be
 *   reached in O(1) time from tail, but tail is merely an
 *   optimization - it can always be reached in O(N) time from
 *   head as well.
 * - The elements contained in the queue are the non-null items in
 *   Nodes that are reachable from head.  CASing the item
 *   reference of a Node to null atomically removes it from the
 *   queue.  Reachability of all elements from head must remain
 *   true even in the case of concurrent modifications that cause
 *   head to advance.  A dequeued Node may remain in use
 *   indefinitely due to creation of an Iterator or simply a
 *   poll() that has lost its time slice.
 *
 * The above might appear to imply that all Nodes are GC-reachable
 * from a predecessor dequeued Node.  That would cause two problems:
 * - allow a rogue Iterator to cause unbounded memory retention
 * - cause cross-generational linking of old Nodes to new Nodes if
 *   a Node was tenured while live, which generational GCs have a
 *   hard time dealing with, causing repeated major collections.
 * However, only non-deleted Nodes need to be reachable from
 * dequeued Nodes, and reachability does not necessarily have to
 * be of the kind understood by the GC.  We use the trick of
 * linking a Node that has just been dequeued to itself.  Such a
 * self-link implicitly means to advance to head.
 *
 * Both head and tail are permitted to lag.  In fact, failing to
 * update them every time one could is a significant optimization
 * (fewer CASes). As with LinkedTransferQueue (see the internal
 * documentation for that class), we use a slack threshold of two;
 * that is, we update head/tail when the current pointer appears
 * to be two or more steps away from the first/last node.
 *
 * Since head and tail are updated concurrently and independently,
 * it is possible for tail to lag behind head (why not)?
 *
 * CASing a Node's item reference to null atomically removes the
 * element from the queue, leaving a "dead" node that should later
 * be unlinked (but unlinking is merely an optimization).
 * Interior element removal methods (other than Iterator.remove())
 * keep track of the predecessor node during traversal so that the
 * node can be CAS-unlinked.  Some traversal methods try to unlink
 * any deleted nodes encountered during traversal.  See comments
 * in bulkRemove.
 *
 * When constructing a Node (before enqueuing it) we avoid paying
 * for a volatile write to item.  This allows the cost of enqueue
 * to be "one-and-a-half" CASes.
 *
 * Both head and tail may or may not point to a Node with a
 * non-null item.  If the queue is empty, all items must of course
 * be null.  Upon creation, both head and tail refer to a dummy
 * Node with null item.  Both head and tail are only updated using
 * CAS, so they never regress, although again this is merely an
 * optimization.
 */

static final class Node<E> {
    volatile E item;
    volatile Node<E> next;

    /**
     * Constructs a node holding item.  Uses relaxed write because
     * item can only be seen after piggy-backing publication via CAS.
     */
    // 构建一个节点，并通过此节点持有实例item
    Node(E item) {
        ITEM.set(this, item);
    }

    Node() {}

    void appendRelaxed(Node<E> next) {
        // assert next != null;
        // assert this.next == null;
        NEXT.set(this, next);
    }

    boolean casItem(E cmp, E val) {
        // assert item == cmp || item == null;
        // assert cmp != null;
        // assert val == null;
        return ITEM.compareAndSet(this, cmp, val);
    }
}
```

## 三、成员变量

```java
/**
 * A node from which the first live (non-deleted) node (if any)
 * can be reached in O(1) time.
 * Invariants:
 * - all live nodes are reachable from head via succ()
 * - head != null
 * - (tmp = head).next != tmp || tmp != head
 * Non-invariants:
 * - head.item may or may not be null.
 * - it is permitted for tail to lag behind head, that is, for tail
 *   to not be reachable from head!
 */
transient volatile Node<E> head;

/**
 * A node from which the last node on list (that is, the unique
 * node with node.next == null) can be reached in O(1) time.
 * Invariants:
 * - the last node is always reachable from tail via succ()
 * - tail != null
 * Non-invariants:
 * - tail.item may or may not be null.
 * - it is permitted for tail to lag behind head, that is, for tail
 *   to not be reachable from head!
 * - tail.next may or may not be self-linked.
 */
private transient volatile Node<E> tail;
```

## 四、构造方法

构造空队列实例

```java
public ConcurrentLinkedQueue() {
    head = tail = new Node<E>();
}
```

通过一个包含实例的集合构建新队列，元素的顺序由集合迭代器决定。如果c为空，抛出NullPointerException

```java
/**
 * Creates a {@code ConcurrentLinkedQueue}
 * initially containing the elements of the given collection,
 * added in traversal order of the collection's iterator.
 *
 * @param c the collection of elements to initially contain
 * @throws NullPointerException if the specified collection or any
 *         of its elements are null
 */
public ConcurrentLinkedQueue(Collection<? extends E> c) {
    Node<E> h = null, t = null;
    for (E e : c) {
        Node<E> newNode = new Node<E>(Objects.requireNonNull(e));
        if (h == null)
            h = t = newNode;
        else
            t.appendRelaxed(t = newNode);
    }
    if (h == null)
        h = t = new Node<E>();
    head = h;
    tail = t;
}
```

## 五、成员方法

```java
/**
 * Tries to CAS head to p. If successful, repoint old head to itself
 * as sentinel for succ(), below.
 */
final void updateHead(Node<E> h, Node<E> p) {
    // assert h != null && p != null && (h == p || h.item == null);
    if (h != p && HEAD.compareAndSet(this, h, p))
        NEXT.setRelease(h, h);
}
```

```java
/**
 * Returns the successor of p, or the head node if p.next has been
 * linked to self, which will only be true if traversing with a
 * stale pointer that is now off the list.
 */
final Node<E> succ(Node<E> p) {
    if (p == (p = p.next))
        p = head;
    return p;
}
```

```java
/**
 * Tries to CAS pred.next (or head, if pred is null) from c to p.
 * Caller must ensure that we're not unlinking the trailing node.
 */
private boolean tryCasSuccessor(Node<E> pred, Node<E> c, Node<E> p) {
    // assert p != null;
    // assert c.item == null;
    // assert c != p;
    if (pred != null)
        return NEXT.compareAndSet(pred, c, p);
    if (HEAD.compareAndSet(this, c, p)) {
        NEXT.setRelease(c, c);
        return true;
    }
    return false;
}
```

```java
/**
 * Collapse dead nodes between pred and q.
 * @param pred the last known live node, or null if none
 * @param c the first dead node
 * @param p the last dead node
 * @param q p.next: the next live node, or null if at end
 * @return either old pred or p if pred dead or CAS failed
 */
private Node<E> skipDeadNodes(Node<E> pred, Node<E> c, Node<E> p, Node<E> q) {
    // assert pred != c;
    // assert p != q;
    // assert c.item == null;
    // assert p.item == null;
    if (q == null) {
        // Never unlink trailing node.
        if (c == p) return pred;
        q = p;
    }
    return (tryCasSuccessor(pred, c, q)
            && (pred == null || ITEM.get(pred) != null))
        ? pred : p;
}
```

```java
/**
 * Inserts the specified element at the tail of this queue.
 * As the queue is unbounded, this method will never return {@code false}.
 *
 * @return {@code true} (as specified by {@link Queue#offer})
 * @throws NullPointerException if the specified element is null
 */
public boolean offer(E e) {
    final Node<E> newNode = new Node<E>(Objects.requireNonNull(e));

    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            // p is last node
            if (NEXT.compareAndSet(p, null, newNode)) {
                // Successful CAS is the linearization point
                // for e to become an element of this queue,
                // and for newNode to become "live".
                if (p != t) // hop two nodes at a time; failure is OK
                    TAIL.weakCompareAndSet(this, t, newNode);
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
        else if (p == q)
            // We have fallen off list.  If tail is unchanged, it
            // will also be off-list, in which case we need to
            // jump to head, from which all live nodes are always
            // reachable.  Else the new tail is a better bet.
            p = (t != (t = tail)) ? t : head;
        else
            // Check for tail updates after two hops.
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

```java
public E poll() {
    restartFromHead: for (;;) {
        for (Node<E> h = head, p = h, q;; p = q) {
            final E item;
            if ((item = p.item) != null && p.casItem(item, null)) {
                // Successful CAS is the linearization point
                // for item to be removed from this queue.
                if (p != h) // hop two nodes at a time
                    updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
            else if ((q = p.next) == null) {
                updateHead(h, p);
                return null;
            }
            else if (p == q)
                continue restartFromHead;
        }
    }
}
```

```java
public E peek() {
    restartFromHead: for (;;) {
        for (Node<E> h = head, p = h, q;; p = q) {
            final E item;
            if ((item = p.item) != null
                || (q = p.next) == null) {
                updateHead(h, p);
                return item;
            }
            else if (p == q)
                continue restartFromHead;
        }
    }
}
```

```java
/**
 * Returns the first live (non-deleted) node on list, or null if none.
 * This is yet another variant of poll/peek; here returning the
 * first node, not element.  We could make peek() a wrapper around
 * first(), but that would cost an extra volatile read of item,
 * and the need to add a retry loop to deal with the possibility
 * of losing a race to a concurrent poll().
 */
Node<E> first() {
    restartFromHead: for (;;) {
        for (Node<E> h = head, p = h, q;; p = q) {
            boolean hasItem = (p.item != null);
            if (hasItem || (q = p.next) == null) {
                updateHead(h, p);
                return hasItem ? p : null;
            }
            else if (p == q)
                continue restartFromHead;
        }
    }
}
```

```java
public boolean isEmpty() {
    return first() == null;
}
```

```java
/**
 * Returns the number of elements in this queue.  If this queue
 * contains more than {@code Integer.MAX_VALUE} elements, returns
 * {@code Integer.MAX_VALUE}.
 *
 * <p>Beware that, unlike in most collections, this method is
 * <em>NOT</em> a constant-time operation. Because of the
 * asynchronous nature of these queues, determining the current
 * number of elements requires an O(n) traversal.
 * Additionally, if elements are added or removed during execution
 * of this method, the returned result may be inaccurate.  Thus,
 * this method is typically not very useful in concurrent
 * applications.
 *
 * @return the number of elements in this queue
 */
public int size() {
    restartFromHead: for (;;) {
        int count = 0;
        for (Node<E> p = first(); p != null;) {
            if (p.item != null)
                if (++count == Integer.MAX_VALUE)
                    break;  // @see Collection.size()
            if (p == (p = p.next))
                continue restartFromHead;
        }
        return count;
    }
}

/**
 * Returns {@code true} if this queue contains the specified element.
 * More formally, returns {@code true} if and only if this queue contains
 * at least one element {@code e} such that {@code o.equals(e)}.
 *
 * @param o object to be checked for containment in this queue
 * @return {@code true} if this queue contains the specified element
 */
public boolean contains(Object o) {
    if (o == null) return false;
    restartFromHead: for (;;) {
        for (Node<E> p = head, pred = null; p != null; ) {
            Node<E> q = p.next;
            final E item;
            if ((item = p.item) != null) {
                if (o.equals(item))
                    return true;
                pred = p; p = q; continue;
            }
            for (Node<E> c = p;; q = p.next) {
                if (q == null || q.item != null) {
                    pred = skipDeadNodes(pred, c, p, q); p = q; break;
                }
                if (p == (p = q)) continue restartFromHead;
            }
        }
        return false;
    }
}
```

```java
/**
 * Removes a single instance of the specified element from this queue,
 * if it is present.  More formally, removes an element {@code e} such
 * that {@code o.equals(e)}, if this queue contains one or more such
 * elements.
 * Returns {@code true} if this queue contained the specified element
 * (or equivalently, if this queue changed as a result of the call).
 *
 * @param o element to be removed from this queue, if present
 * @return {@code true} if this queue changed as a result of the call
 */
public boolean remove(Object o) {
    if (o == null) return false;
    restartFromHead: for (;;) {
        for (Node<E> p = head, pred = null; p != null; ) {
            Node<E> q = p.next;
            final E item;
            if ((item = p.item) != null) {
                if (o.equals(item) && p.casItem(item, null)) {
                    skipDeadNodes(pred, p, p, q);
                    return true;
                }
                pred = p; p = q; continue;
            }
            for (Node<E> c = p;; q = p.next) {
                if (q == null || q.item != null) {
                    pred = skipDeadNodes(pred, c, p, q); p = q; break;
                }
                if (p == (p = q)) continue restartFromHead;
            }
        }
        return false;
    }
}
```

```java
/**
 * Appends all of the elements in the specified collection to the end of
 * this queue, in the order that they are returned by the specified
 * collection's iterator.  Attempts to {@code addAll} of a queue to
 * itself result in {@code IllegalArgumentException}.
 *
 * @param c the elements to be inserted into this queue
 * @return {@code true} if this queue changed as a result of the call
 * @throws NullPointerException if the specified collection or any
 *         of its elements are null
 * @throws IllegalArgumentException if the collection is this queue
 */
public boolean addAll(Collection<? extends E> c) {
    if (c == this)
        // As historically specified in AbstractQueue#addAll
        throw new IllegalArgumentException();

    // Copy c into a private chain of Nodes
    Node<E> beginningOfTheEnd = null, last = null;
    for (E e : c) {
        Node<E> newNode = new Node<E>(Objects.requireNonNull(e));
        if (beginningOfTheEnd == null)
            beginningOfTheEnd = last = newNode;
        else
            last.appendRelaxed(last = newNode);
    }
    if (beginningOfTheEnd == null)
        return false;

    // Atomically append the chain at the tail of this collection
    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            // p is last node
            if (NEXT.compareAndSet(p, null, beginningOfTheEnd)) {
                // Successful CAS is the linearization point
                // for all elements to be added to this queue.
                if (!TAIL.weakCompareAndSet(this, t, last)) {
                    // Try a little harder to update tail,
                    // since we may be adding many elements.
                    t = tail;
                    if (last.next == null)
                        TAIL.weakCompareAndSet(this, t, last);
                }
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
        else if (p == q)
            // We have fallen off list.  If tail is unchanged, it
            // will also be off-list, in which case we need to
            // jump to head, from which all live nodes are always
            // reachable.  Else the new tail is a better bet.
            p = (t != (t = tail)) ? t : head;
        else
            // Check for tail updates after two hops.
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

```java
/**
 * Tolerate this many consecutive dead nodes before CAS-collapsing.
 * Amortized cost of clear() is (1 + 1/MAX_HOPS) CASes per element.
 */
private static final int MAX_HOPS = 8;

/** Implementation of bulk remove methods. */
private boolean bulkRemove(Predicate<? super E> filter) {
    boolean removed = false;
    restartFromHead: for (;;) {
        int hops = MAX_HOPS;
        // c will be CASed to collapse intervening dead nodes between
        // pred (or head if null) and p.
        for (Node<E> p = head, c = p, pred = null, q; p != null; p = q) {
            q = p.next;
            final E item; boolean pAlive;
            if (pAlive = ((item = p.item) != null)) {
                if (filter.test(item)) {
                    if (p.casItem(item, null))
                        removed = true;
                    pAlive = false;
                }
            }
            if (pAlive || q == null || --hops == 0) {
                // p might already be self-linked here, but if so:
                // - CASing head will surely fail
                // - CASing pred's next will be useless but harmless.
                if ((c != p && !tryCasSuccessor(pred, c, c = p))
                    || pAlive) {
                    // if CAS failed or alive, abandon old pred
                    hops = MAX_HOPS;
                    pred = p;
                    c = q;
                }
            } else if (p == q)
                continue restartFromHead;
        }
        return removed;
    }
}
```

## 六、常量

```java
// VarHandle mechanics
private static final VarHandle HEAD;
private static final VarHandle TAIL;
static final VarHandle ITEM;
static final VarHandle NEXT;
static {
    try {
        MethodHandles.Lookup l = MethodHandles.lookup();
        HEAD = l.findVarHandle(ConcurrentLinkedQueue.class, "head",
                               Node.class);
        TAIL = l.findVarHandle(ConcurrentLinkedQueue.class, "tail",
                               Node.class);
        ITEM = l.findVarHandle(Node.class, "item", Object.class);
        NEXT = l.findVarHandle(Node.class, "next", Node.class);
    } catch (ReflectiveOperationException e) {
        throw new ExceptionInInitializerError(e);
    }
}
```
