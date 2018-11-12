---
layout:     post
title:      "SynchronizedMap"
date:       2017-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - tags
---

```java
/**
 * Returns a synchronized (thread-safe) map backed by the specified
 * map.  In order to guarantee serial access, it is critical that
 * <strong>all</strong> access to the backing map is accomplished
 * through the returned map.<p>
 *
 * It is imperative that the user manually synchronize on the returned
 * map when traversing any of its collection views via {@link Iterator},
 * {@link Spliterator} or {@link Stream}:
 * <pre>
 *  Map m = Collections.synchronizedMap(new HashMap());
 *      ...
 *  Set s = m.keySet();  // Needn't be in synchronized block
 *      ...
 *  synchronized (m) {  // Synchronizing on m, not s!
 *      Iterator i = s.iterator(); // Must be in synchronized block
 *      while (i.hasNext())
 *          foo(i.next());
 *  }
 * </pre>
 * Failure to follow this advice may result in non-deterministic behavior.
 *
 * <p>The returned map will be serializable if the specified map is
 * serializable.
 *
 * @param <K> the class of the map keys
 * @param <V> the class of the map values
 * @param  m the map to be "wrapped" in a synchronized map.
 * @return a synchronized view of the specified map.
 */
public static <K,V> Map<K,V> synchronizedMap(Map<K,V> m) {
    return new SynchronizedMap<>(m);
}
```

```java
/**
 * @serial include
 */
private static class SynchronizedMap<K,V>
    implements Map<K,V>, Serializable {
    private static final long serialVersionUID = 1978198479659022715L;

    private final Map<K,V> m;     // Backing Map
    final Object      mutex;        // Object on which to synchronize

    SynchronizedMap(Map<K,V> m) {
        this.m = Objects.requireNonNull(m);
        mutex = this;
    }

    SynchronizedMap(Map<K,V> m, Object mutex) {
        this.m = m;
        this.mutex = mutex;
    }

    public int size() {
        synchronized (mutex) {return m.size();}
    }
    public boolean isEmpty() {
        synchronized (mutex) {return m.isEmpty();}
    }
    public boolean containsKey(Object key) {
        synchronized (mutex) {return m.containsKey(key);}
    }
    public boolean containsValue(Object value) {
        synchronized (mutex) {return m.containsValue(value);}
    }
    public V get(Object key) {
        synchronized (mutex) {return m.get(key);}
    }

    public V put(K key, V value) {
        synchronized (mutex) {return m.put(key, value);}
    }
    public V remove(Object key) {
        synchronized (mutex) {return m.remove(key);}
    }
    public void putAll(Map<? extends K, ? extends V> map) {
        synchronized (mutex) {m.putAll(map);}
    }
    public void clear() {
        synchronized (mutex) {m.clear();}
    }

    private transient Set<K> keySet;
    private transient Set<Map.Entry<K,V>> entrySet;
    private transient Collection<V> values;

    public Set<K> keySet() {
        synchronized (mutex) {
            if (keySet==null)
                keySet = new SynchronizedSet<>(m.keySet(), mutex);
            return keySet;
        }
    }

    public Set<Map.Entry<K,V>> entrySet() {
        synchronized (mutex) {
            if (entrySet==null)
                entrySet = new SynchronizedSet<>(m.entrySet(), mutex);
            return entrySet;
        }
    }

    public Collection<V> values() {
        synchronized (mutex) {
            if (values==null)
                values = new SynchronizedCollection<>(m.values(), mutex);
            return values;
        }
    }

    public boolean equals(Object o) {
        if (this == o)
            return true;
        synchronized (mutex) {return m.equals(o);}
    }
    public int hashCode() {
        synchronized (mutex) {return m.hashCode();}
    }
    public String toString() {
        synchronized (mutex) {return m.toString();}
    }

    // Override default methods in Map
    @Override
    public V getOrDefault(Object k, V defaultValue) {
        synchronized (mutex) {return m.getOrDefault(k, defaultValue);}
    }
    @Override
    public void forEach(BiConsumer<? super K, ? super V> action) {
        synchronized (mutex) {m.forEach(action);}
    }
    @Override
    public void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {
        synchronized (mutex) {m.replaceAll(function);}
    }
    @Override
    public V putIfAbsent(K key, V value) {
        synchronized (mutex) {return m.putIfAbsent(key, value);}
    }
    @Override
    public boolean remove(Object key, Object value) {
        synchronized (mutex) {return m.remove(key, value);}
    }
    @Override
    public boolean replace(K key, V oldValue, V newValue) {
        synchronized (mutex) {return m.replace(key, oldValue, newValue);}
    }
    @Override
    public V replace(K key, V value) {
        synchronized (mutex) {return m.replace(key, value);}
    }
    @Override
    public V computeIfAbsent(K key,
            Function<? super K, ? extends V> mappingFunction) {
        synchronized (mutex) {return m.computeIfAbsent(key, mappingFunction);}
    }
    @Override
    public V computeIfPresent(K key,
            BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        synchronized (mutex) {return m.computeIfPresent(key, remappingFunction);}
    }
    @Override
    public V compute(K key,
            BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        synchronized (mutex) {return m.compute(key, remappingFunction);}
    }
    @Override
    public V merge(K key, V value,
            BiFunction<? super V, ? super V, ? extends V> remappingFunction) {
        synchronized (mutex) {return m.merge(key, value, remappingFunction);}
    }

    private void writeObject(ObjectOutputStream s) throws IOException {
        synchronized (mutex) {s.defaultWriteObject();}
    }
}
```

