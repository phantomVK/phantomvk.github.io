---
layout:     post
title:      "Java源码系列(11) -- LinkedHashMap"
date:       2018-07-09
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Java源码系列
---

## 一、类签名

`LinkedHashMap<K,V>`继承自[HashMap<K,V>](https://phantomvk.github.io/2018/06/30/HashMap/)，可知存入的节点key永远是唯一的。可以通过Android的[LruCache](https://phantomvk.github.io/2017/02/28/LruCache/)了解`LinkedHashMap`用法。

```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
```

![HashMap_UML](/img/java/LinkedHashMap_UML.png)

## 二、节点


`Entry<K,V>`是`HashMap.Node<K,V>`的子类，增加`before`、`after`引用实现双向链表

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after; // 前节点、后节点
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next); // 调用HashMap构造方法
    }
}
```

![HashMap_UML](/img/java/LinkedHashMap_Entry_UML.png)

## 三、数据成员

双向链表头，指向最早（最老）访问节点元素
```java
transient LinkedHashMap.Entry<K,V> head;
```

双向链表尾，指向最近（最晚）访问节点元素
```java
transient LinkedHashMap.Entry<K,V> tail;
```

是否保持访问顺序，为true则每次被访问的节点都会放到链表尾部
```java
final boolean accessOrder;
```

依次插入`Entry_0`到`Entry_5`，当`accessOrder`为true并访问`Entry_4`，则`Entry_4`移到链尾。

![HashMap_UML](/img/java/LinkedHashMap_accessOrder_true.png)

## 四、构造方法
```java
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}

public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}

public LinkedHashMap() {
    super();
    accessOrder = false;
}

public LinkedHashMap(Map<? extends K, ? extends V> m) {
    super();
    accessOrder = false;
    putMapEntries(m, false);
}

// 维持存取顺序仅能通过此构造方法
public LinkedHashMap(int initialCapacity,
                     float loadFactor,
                     boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}
```

## 五、成员方法

```java
// 把节点插入到链表尾部
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    // 如果尾节点为空表明链表没有元素，则p就是头结点
    if (last == null)
        head = p;
    else {
        // 处理双向链表节点
        p.before = last;
        last.after = p;
    }
}

// apply src's links to dst
private void transferLinks(LinkedHashMap.Entry<K,V> src,
                           LinkedHashMap.Entry<K,V> dst) {
    LinkedHashMap.Entry<K,V> b = dst.before = src.before;
    LinkedHashMap.Entry<K,V> a = dst.after = src.after;
    if (b == null)
        head = dst;
    else
        b.after = dst;
    if (a == null)
        tail = dst;
    else
        a.before = dst;
}

// 重写HashMap钩子方法
void reinitialize() {
    super.reinitialize();
    head = tail = null;
}

// 创建新链表节点
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}

// 替换链表节点
Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
    LinkedHashMap.Entry<K,V> q = (LinkedHashMap.Entry<K,V>)p;
    LinkedHashMap.Entry<K,V> t =
        new LinkedHashMap.Entry<>(q.hash, q.key, q.value, next);
    transferLinks(q, t);
    return t;
}

// 创建新红黑树节点
TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
    TreeNode<K,V> p = new TreeNode<>(hash, key, value, next);
    linkNodeLast(p);
    return p;
}

// 替换红黑树节点
TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    LinkedHashMap.Entry<K,V> q = (LinkedHashMap.Entry<K,V>)p;
    TreeNode<K,V> t = new TreeNode<>(q.hash, q.key, q.value, next);
    transferLinks(q, t);
    return t;
}
```

## 六、顺序操作

```java
// 把节点从链表解除链接
void afterNodeRemoval(Node<K,V> e) {
    // p：即是节点e
    // b：e的前一个节点
    // a：e的后一个节点
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;

    // 置空节点p的前后引用
    p.before = p.after = null;
    
    if (b == null) {
        // 可知节点e是链表头结点，则e的下一个节点a作为链表的头结点
        head = a;
    } else {
        // 可知节点e本是中间结点，把e下一个节点a作为e上一个节点的后续节点
        b.after = a;
    }

    if (a == null) {
        // 可知节点e本是链表尾节点，则e的上一个节点b作为链表的尾节点
        tail = b;
    } else {
        // 可知节点e本身是中间节点，把e上一个节点b作为e下一个节点的前置节点
        a.before = b;
    }
}

// 父类HashMap调用putVal()中会调用此方法，移除最少使用的节点
void afterNodeInsertion(boolean evict) {
    // first是最少使用的节点
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        // 获取first的key
        K key = first.key;
        // 通过key找到对应Node并移除
        removeNode(hash(key), key, null, false, true);
    }
}

// 把节点移动到链表尾
void afterNodeAccess(Node<K,V> e) {
    LinkedHashMap.Entry<K,V> last;
    // 仅当accessOrder为true且被访问元素不是尾节点
    if (accessOrder && (last = tail) != e) {
        // p：即是节点e
        // b：e的前一个节点
        // a：e的后一个节点
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;

        // 置空节点p的后引用
        p.after = null;

        if (b == null) {
            // 可知节点e本是头结点，则e的下一个节点a作为链表的头结点
            head = a;
        } else {
            // 可知节点e本是中间结点，把e下一个节点a作为e上一个节点的后续节点
            b.after = a;
        }

        if (a != null) {
            // 可知节点e本身是中间节点，把e上一个节点b作为e下一个节点的前置节点
            a.before = b;
        } else {
            // 可知节点e本是尾节点，则e的上一个节点b作为链表的尾节点
            last = b;
        }
        
        // p之前没有节点，表明p就是头结点
        if (last == null) {
            head = p;
        } else {
            // p作为新的尾节点，链接到上一个尾节点之后
            p.before = last;
            last.after = p;
        }

        tail = p; // tail引用指向p
        ++modCount; // 修改次数递增
    }
}
```

## 七、获取

```java
// 检查LinkedHashMap是否包含指定value
public boolean containsValue(Object value) {
    // 从链表头结点开始遍历，逐个查找Entry.value是否等于value
    for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
        V v = e.value;
        if (v == value || (value != null && value.equals(v)))
            return true; // 包含对应value，返回true
    }
    return false; // 不包含对应value，返回false
}

// 通过Key获取对应Entry的value
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null) {
        // 通过key获取Node为空，返回null作为结果
        return null;
    }

    if (accessOrder) {
        // 通过key获取Node不为空，执行afterNodeAccess(e)调整顺序
        afterNodeAccess(e);
    }

    return e.value; // 最后把获取的Entry.value返回
}

// 通过Key获取对应Entry的value
public V getOrDefault(Object key, V defaultValue) {
   Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null) {
       // 通过key获取Node为空，返回defaultValue作为结果
       return defaultValue;
    }
    
    if (accessOrder) {
        // 通过key获取Node不为空，执行afterNodeAccess(e)调整顺序
        afterNodeAccess(e);
    }

   return e.value; // 最后把获取的Entry.value返回
}
```

## 八、移除

```java
// 清除所有引用
public void clear() {
    super.clear(); // 把HashMap所有Entry都清空
    head = tail = null;  // 置空head和tail引用
}

// 方法主要用于被子类重写，决定最少使用的节点能否被移除
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
} 
```