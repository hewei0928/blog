---
layout: w
title: JDK源码学习系列14----HashTable
date: 2018-05-14 14:27:29
tags:
    - Java
    - JDK
    - Map
---

> `HashTable`和`HashMap`是面试时必问的问题，之前已经对`HashMap`有了较深入的理解。但对`HashTable`的了解就少了不少，这里就对其进行简单的分析。

<!--more-->
## 类结构
```
public class Hashtable<K,V>
    extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable 
```
&ensp;&ensp;`HashTable`继承了`Dictionary`抽象类，实现了`Map`,`Cloneable`,`Serializable`接口。 `Dictionary`是一个类似于`Map`的键值对抽象类，其内的方法也与`Map`有许多相似之处。

## 成员变量
```java
    
    //hashTable底层由数组加链表实现
    private transient Entry<?,?>[] table;

    // 实际存储键值对数量
    private transient int count;

    // 临界值
    private int threshold;

    // 填充因子
    private float loadFactor;

    // fail-fast机制
    private transient int modCount = 0;

    private static final long serialVersionUID = 1421746759512286392L;
```

## 构造函数
```
// 自定义数组大小与填充因子
public Hashtable(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal Load: "+loadFactor);

    if (initialCapacity==0)
        initialCapacity = 1;
    this.loadFactor = loadFactor;
    // 在构造函数中即初始化数组
    table = new Entry<?,?>[initialCapacity];
    // 临界值为数组大小*填充因子
    threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
}

//自定义数组大小
public Hashtable(int initialCapacity) {
    this(initialCapacity, 0.75f);
}

// 默认数组大小与填充因子
public Hashtable() {
    this(11, 0.75f);
}


public Hashtable(Map<? extends K, ? extends V> t) {
    this(Math.max(2*t.size(), 11), 0.75f);
    putAll(t);
}
```

## 成员函数

### put
```java
public synchronized V put(K key, V value) {
    // hashtable 的 value不能为null
    if (value == null) {
        throw new NullPointerException();
    }

    // Makes sure the key is not already in the hashtable.
    Entry<?,?> tab[] = table;
    int hash = key.hashCode(); // key值也不能为空
    // public static final int   MAX_VALUE = 0x7fffffff
    int index = (hash & 0x7FFFFFFF) % tab.length;
    // 根据hash值查找链表存储的数组下标
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];
    // 链表遍历查找对应key值是否存在，如果存在则覆盖value值
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }
    
    // 链表中不存在对应key值 则添加新
    addEntry(hash, key, value, index);
    return null;
}


private void addEntry(int hash, K key, V value, int index) {
    modCount++;

    Entry<?,?> tab[] = table;
    // 实际存储键值对数 >= 容量
    if (count >= threshold) {
        // 数组扩容并将链表放入新下标处
        rehash();
        
        tab = table;
        hash = key.hashCode();
        index = (hash & 0x7FFFFFFF) % tab.length;
    }

    // Creates the new entry.
    @SuppressWarnings("unchecked")
    // 原数组下标处存储的链表
    Entry<K,V> e = (Entry<K,V>) tab[index];
    // 新节点的next指向原链表
    tab[index] = new Entry<>(hash, key, value, e);
    count++;
}


protected void rehash() {
    int oldCapacity = table.length;
    Entry<?,?>[] oldMap = table;

    // 新数组大小为原数组 * 2 + 1
    int newCapacity = (oldCapacity << 1) + 1;
    // 容量过大就不管了
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        if (oldCapacity == MAX_ARRAY_SIZE)
            // Keep running with MAX_ARRAY_SIZE buckets
            return;
        newCapacity = MAX_ARRAY_SIZE;
    }
    Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

    modCount++;
    // 更新容量
    threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    table = newMap;

    for (int i = oldCapacity ; i-- > 0 ;) {
        for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
            Entry<K,V> e = old;
            old = old.next;
            // 计算新下标
            int index = (e.hash & 0x7FFFFFFF) % newCapacity;
            e.next = (Entry<K,V>)newMap[index];// null ?
            newMap[index] = e;
        }
    }
}
```

### putIfAbsent
```
public synchronized V putIfAbsent(K key, V value) {
    Objects.requireNonNull(value);

    // Makes sure the key is not already in the hashtable.
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];
    for (; entry != null; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            // 如果key已存在且原来的值为null 则进行值替代
            // 原来的值不可能为null, 因此只要key值存在则不进行操作
            if (old == null) {
                entry.value = value;
            }
            return old;
        }
    }
    
    // key值不存在则添加新链表节点
    addEntry(hash, key, value, index);
    return null;
}
```

### clear
```java
public synchronized void clear() {
    Entry<?,?> tab[] = table;
    modCount++;
    for (int index = tab.length; --index >= 0; )
        tab[index] = null;
    count = 0;
}
```

### contains、containValue
```java
public synchronized boolean contains(Object value) {
    if (value == null) {
        throw new NullPointerException();
    }

    Entry<?,?> tab[] = table;
    //遍历数组及链表查找对应value
    for (int i = tab.length ; i-- > 0 ;) {
        for (Entry<?,?> e = tab[i] ; e != null ; e = e.next) {
            if (e.value.equals(value)) {
                return true;
            }
        }
    }
    return false;
}

public boolean containsValue(Object value) {
    return contains(value);
}
```

### containsKey
```
public synchronized boolean containsKey(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    // 根据hash值确定下标，遍历下标处链表查找对应key
    for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            return true;
        }
    }
    return false;
}
```

### entrySet
```java
public Set<Map.Entry<K,V>> entrySet() {
    if (entrySet==null)
        entrySet = Collections.synchronizedSet(new EntrySet(), this);
    return entrySet;
}

private class EntrySet extends AbstractSet<Map.Entry<K,V>> {
    public Iterator<Map.Entry<K,V>> iterator() {
        return getIterator(ENTRIES);
    }

    public boolean add(Map.Entry<K,V> o) {
        return super.add(o);
    }

    public boolean contains(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> entry = (Map.Entry<?,?>)o;
        Object key = entry.getKey();
        Entry<?,?>[] tab = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;

        for (Entry<?,?> e = tab[index]; e != null; e = e.next)
            if (e.hash==hash && e.equals(entry))
                return true;
        return false;
    }

    public boolean remove(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> entry = (Map.Entry<?,?>) o;
        Object key = entry.getKey();
        Entry<?,?>[] tab = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;

        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>)tab[index];
        for(Entry<K,V> prev = null; e != null; prev = e, e = e.next) {
            if (e.hash==hash && e.equals(entry)) {
                modCount++;
                if (prev != null)
                    prev.next = e.next;
                else
                    tab[index] = e.next;

                count--;
                e.value = null;
                return true;
            }
        }
        return false;
    }

    public int size() {
        return count;
    }

    public void clear() {
        Hashtable.this.clear();
    }
}
```

### foreach
```
public synchronized void forEach(BiConsumer<? super K, ? super V> action) {
    Objects.requireNonNull(action);     // explicit check required in case
                                        // table is empty.
    final int expectedModCount = modCount;

    Entry<?, ?>[] tab = table;
    for (Entry<?, ?> entry : tab) {
        while (entry != null) {
            action.accept((K)entry.key, (V)entry.value);
            entry = entry.next;

            if (expectedModCount != modCount) {
                throw new ConcurrentModificationException();
            }
        }
    }
}
```

### get
```java
public synchronized V get(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            return (V)e.value;
        }
    }
    return null;
}
```

### keySet
```java
public Set<K> keySet() {
    if (keySet == null)
        keySet = Collections.synchronizedSet(new KeySet(), this);
    return keySet;
}

private class KeySet extends AbstractSet<K> {
    public Iterator<K> iterator() {
        return getIterator(KEYS);
    }
    public int size() {
        return count;
    }
    public boolean contains(Object o) {
        return containsKey(o);
    }
    public boolean remove(Object o) {
        return Hashtable.this.remove(o) != null;
    }
    public void clear() {
        Hashtable.this.clear();
    }
}
```

### remove
```
public synchronized V remove(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>)tab[index];
    // 找到对应链表删除对应节点
    for(Entry<K,V> prev = null ; e != null ; prev = e, e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            modCount++;
            if (prev != null) {
                prev.next = e.next;
            } else {
                tab[index] = e.next;
            }
            count--;
            V oldValue = e.value;
            e.value = null;
            return oldValue;
        }
    }
    return null;
}

// 如果key存在 且对应的值==value，删除对应节点 返回true；否则返回false
public synchronized boolean remove(Object key, Object value) {
    Objects.requireNonNull(value);

    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>)tab[index];
    for (Entry<K,V> prev = null; e != null; prev = e, e = e.next) {
        if ((e.hash == hash) && e.key.equals(key) && e.value.equals(value)) {
            modCount++;
            if (prev != null) {
                prev.next = e.next;
            } else {
                tab[index] = e.next;
            }
            count--;
            e.value = null;
            return true;
        }
    }
    return false;
}
```

## 总结与比较
`HashTable`的源码比较`HashMap`简单了不少，毕竟已经许久未更新了。读完源码后对这两个类的区别有了更加深入的理解：

1. `HashTable`的方法都添加了`synchronized`关键字，确保了线程安全，而`HashMap`则不是线程安全的。
2. `HashTable`的key、value不能为空，而`HashMap`可以
3. `HashMap`底层使用数组+链表+红黑树的实现方式，`HashTable`使用的是数组+链表的实现方式
4. 两个`Map`扩容以及`rehash`的方式也大不相同。
5. `HashMap`对象创建时数组不初始化，当首次调用put才初始化数组；而`HashTable`则在对象创建时初始化数组。