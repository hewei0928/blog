---
layout: 'n'
title: JDK源码学习系列06----HashMap
date: 2017-12-15 09:26:47
tags:
    - Java
    - HashMap
    - JDK
---

## HashMap简介
>&ensp;&ensp;`HashMap`是`Java`程序员使用频率最高的用于映射(键值对)处理的数据类型。
网上关于`HashMap`的博客多是关于`JDK1.6`的。在`JDK1.6`中，`HashMap`采用桶位+链表实现，即使用链表处理`hash`冲突，同一`hash`值的元素都存储在一个链表里。但是当位于一个桶中的元素较多，即`hash`值相等的元素较多时，通过key值依次查找的效率较低。而`JDK1.8`中，`HashMap`采用位桶+链表+红黑树实现，当链表长度超过阈值（8）时，将链表转换为红黑树，这样大大减少了查找时间。


<!--more-->

## HashMap结构

![360桌面截图20171215095906.jpg-30.2kB][1]



  [1]: http://static.zybuluo.com/hewei0928/pkntk97vux052q5sxye8so6l/360%E6%A1%8C%E9%9D%A2%E6%88%AA%E5%9B%BE20171215095906.jpg
  
```java
public class HashMap<K,V> extends AbstractMap<K,V>
implements Map<K,V>, Cloneable, Serializable
```
&ensp;&ensp;HashMap 是一个散列表，它存储的内容是键值对(key-value)映射。
&ensp;&ensp;HashMap 继承于AbstractMap，实现了Map、Cloneable、java.io.Serializable接口。
&ensp;&ensp;Map接口定义了所有Map子类必须实现的方法。Map接口中还定义了一个内部接口Entry<K, V>, 所有具体`Map`实现类实际存储数据的节点类继承自此接口；

## 成员变量
```java
//默认初始容量 16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

//最大容量，2的30次方
static final int MAXIMUM_CAPACITY = 1 << 30;

//默认装载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//当桶(bucket)上的结点数大于这个值时会转成红黑树
static final int TREEIFY_THRESHOLD = 8;

//当桶(bucket)上的结点数小于这个值时树转链表
static final int UNTREEIFY_THRESHOLD = 6;

//桶中结构转化为红黑树对应的table的最小大小
static final int MIN_TREEIFY_CAPACITY = 64;

//存储元素的数组
transient Node<K,V>[] table;

//存放具体元素的集，用于迭代元素
transient Set<Map.Entry<K,V>> entrySet;

//存放元素的个数
transient int size;

//每次结构更改时的计数器 put, remove, clear中有调用
transient int modCount;

//临界值 当实际大小(容量*填充因子)超过临界值时，会进行扩容，一定为2的幂次数
int threshold;

//填充因子
final float loadFactor;
```
`table`是一个`Entry`类型数组，`Entry`类型用于存储具体的"key-value键值对"。size是HashMap的大小，它是HashMap保存的实际键值对的数量。 
threshold是HashMap的阈值，用于判断是否需要调整HashMap的容量。threshold的值="容量*加载因子"，当HashMap中存储数据的数量达到threshold时，就需要对HashMap进行扩容。
loadFactor就是加载因子。
modCount是用来实现fail-fast机制的。


## 构造函数
```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;//填充因子设为0.75
}
```
默认构造函数，初始化`HashMap`，并将填充因子设为0.75。 可以看出此时`HashMap`中的`table`并未在堆中分配地址。当首次进行put操作时才会进行table初始化

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    //hashmap 临界值为不小于传入参数的二次幂数
    this.threshold = tableSizeFor(initialCapacity);
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```
传入临界值与填充因子初始化`HashMap`。

```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
`tableSizeFor`方法用于计算临界值。tableSizeFor(initialCapacity)返回大于等于initialCapacity的最小的二次幂数。

```java
public HashMap(Map<? extends K, ? extends V> m) {
    // 负载因子设为0.75
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}

final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        // 
        if (table == null) { // pre-size
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        else if (s > threshold)
            resize();
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

## 重要方法
### put() 和 putVal()
```java
//如果key中已经有对应值会被替代
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

//从此计算hash值的方法可以看出，hashMap中的key值可以为null
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 如果是hashMap初始化后首次存储数据，则会对数组进行初始化
    // 其内机制大致为： 1. 如果hashmap初始化时无参数， 则数组长度为16， threshold(阈值)变为12即数组长*填充因子（loadFactor） 2. 如果HashMap初始化时有参数， 则数组长为初始化时的阈值， 新阈值变为原阈值*填充因子（loadFactor）
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 因为n(数组长)为二次幂数，因此n-1的二进制数低位全部为1。例如n = 16, 则n-1二进制表示为00000000 00000000 00001111。
    // (n - 1) & hash 时会使hash的低位为1，且结果在[0, n)之间
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // p为tab[i]处存储的链表的头节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            // 内部逻辑：如果链表长达到8，就进行resize扩展,如果数组长大于64则转换为树.
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                // 直至链表尾部也未找到hash相等且key相等的节点
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    //00000000 00000000 00001111
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                
                // 如果在循环至尾节点的过程中发现key已存在， E指向那个节点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // key值重复， 则覆盖value, 返旧value
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    
    // key值不重复， 即有新节点添加， 改变modCount
    ++modCount;
    // 当实际存储数据数大于阈值时 进行扩容
    // 由此可以发现 负载因子越大,相同大小的数组能存储的数据越多，则散列表的装填程度越高，也就是能容纳更多的元素，元素多了，链表大了，所以此时索引效率就会降低。反之，负载因子越小则链表中的数据量就越稀疏，此时会对空间造成烂费，但是此时索引效率高。（空间换时间，时间换空间）
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}

```

### putAll()、 putIfAbsent()
```
public void putAll(Map<? extends K, ? extends V> m) {
    putMapEntries(m, true);
}

// 如果指定的键未与某个值关联（或映射到null），则将其与给定值关联并返回null，否则返回当前已存在的值。
public V putIfAbsent(K key, V value) {
    return putVal(hash(key), key, value, true, true);
}
```
`putIfAbsent`从putVal源码可知， 当onlyIfAbsent为true时, 如果`key`已经存在，且对应value不为null, 值不会进行覆盖。

### resize()
`HashMap`扩容方法

```java
final Node<K,V>[] resize() {
        //保存当前table
        Node<K,V>[] oldTab = table;
        //保存table大小
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        //保存当前阈值
        int oldThr = threshold;
        int newCap, newThr = 0;
        //原table不为0
        if (oldCap > 0) {
            //原table大小大于最大值
            if (oldCap >= MAXIMUM_CAPACITY) {
                //阈值变为最大int
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //table长度加倍，如果原table长度大于16， 阈值加倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        //原table长度为0， 且阈值不为0时，新table长度为阈值
        // 在第一次带参数初始化时候会有这种情况
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        // 在默认无参数初始化会有这种情况
        else {               // zero initial threshold signifies using defaults 
            newCap = DEFAULT_INITIAL_CAPACITY;//16
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);//12
        }
        
        // 在第一次带参数初始化时候会有这种情况
        // 原table长度为0， 且原阈值不为0时， 新阈值为新数组长度 * 哈希加载因子
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        
        /** 至此可知， hashMap 阈值为 数组长度 * 哈希加载因子 **/
        
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        //创建新数组， 如果是扩容情况下，数组长度变为原来两倍，依然为二次幂数
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        
        // 不是首次初始化， 即为需要扩容时
        if (oldTab != null) {
            // 循环数组， 重新进行赋值
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    //如果数组对应位置有值， 且链表长度为1， 直接进行复制
                    if (e.next == null)
                        // 新位置为原位置或者原位置+旧数组长
                        newTab[e.hash & (newCap - 1)] = e;
                    //该位置结构为红黑树， 暂时跳过
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                        
                        // 链表移动后位置变化， 存入及查找时得出的位置也会变化（简单来说就是数组长度*2， 在二进制中表示即为左移一位， 只需判断hash中的高一位为1或0，为1则新数组存储时计算存储位置会使结果变为原结果+元数组长）
                        // 具体可以看https://zhuanlan.zhihu.com/p/21673805 这篇文章的扩容计算
                    }
                }
            }
        }
        
        // 是首次初始化， 直接返回新数组
        return newTab;
    }
```

### clear()

```java
public void clear() {
    Node<K,V>[] tab;
    // 有结构改变 modCount+1
    modCount++;
    if ((tab = table) != null && size > 0) {
        //实际存储长度变为0
        size = 0;
        //数组个位置指向null
        for (int i = 0; i < tab.length; ++i)
            tab[i] = null;
    }
}
```
`clear`方法将数组各位置变为空，将实际存储数据数`size`变为0

### containKey() 
```
public boolean containsKey(Object key) {
    return getNode(hash(key), key) != null;
}


final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```
判断key是否存在，先根据key确定数组中的位置，再去遍历数组位置上存储的链表或树查找是否存在对应节点，不存在则返回false

### containsValue()
```
public boolean containsValue(Object value) {
    Node<K,V>[] tab; V v;
    if ((tab = table) != null && size > 0) {
        for (int i = 0; i < tab.length; ++i) {
            for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                if ((v = e.value) == value ||
                    (value != null && value.equals(v)))
                    return true;
            }
        }
    }
    return false;
}
```
从数组0下标处依次遍历其中节点，查找是否存在。(`TreeNode`继承自`Node`也有next节点)

### foreach()
```java
public void forEach(BiConsumer<? super K, ? super V> action) {
    Node<K,V>[] tab;
    if (action == null)
        throw new NullPointerException();
    if (size > 0 && (tab = table) != null) {
        int mc = modCount;
        for (int i = 0; i < tab.length; ++i) {
            for (Node<K,V> e = tab[i]; e != null; e = e.next)
                action.accept(e.key, e.value);
        }
        if (modCount != mc)
            throw new ConcurrentModificationException();
    }
}
```
Java8新添加的方法，配合函数接口、Lambda表达式对数据进行遍历。可以改变其内存储对象。用法示例：
```
Map<String, Object> c = new HashMap<>();
        c.put("1", 1);
        c.put("2", 2);

        c.forEach((s, o) -> {
            System.out.println(s + o);
        });
```

### get()和getOrDefault()
```
//key不存在时返回null
public V get(Object key) {
    Node<K,V> e;
    //getNode 根据key确定数组中的位置，再去遍历数组位置上存储的链表或树查找对应节点
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

//key不存在时返回defaultValue
public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
}
```

### isEmpty()
```
//  根据实际存储数据数判断是否为空
public boolean isEmpty() {
    return size == 0;
}
```

### remove()
```java
// 移除key对应节点， 并返回对应value。 不存在时返回null
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

// 移除key, value相对应的节点
public boolean remove(Object key, Object value) {
    return removeNode(hash(key), key, value, true, true) != null;
}

// 从链表或者树结构移除对应节点
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    // 数组不为空，且key对应存储位置有链表或者树结构
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

### replace()
```java
// 替换key对应值， 返回旧值
public V replace(K key, V value) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) != null) {
        V oldValue = e.value;
        e.value = value;
        afterNodeAccess(e);
        return oldValue;
    }
    return null;
}

// 替换key,value对应节点的值， 如果对应节点不存在则返回false
public boolean replace(K key, V oldValue, V newValue) {
    Node<K,V> e; V v;
    if ((e = getNode(hash(key), key)) != null &&
        ((v = e.value) == oldValue || (v != null && v.equals(oldValue)))) {
        e.value = newValue;
        afterNodeAccess(e);
        return true;
    }
    return false;
}

//自定义方法， 将所有节点的value变为自定义值
public void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {
    Node<K,V>[] tab;
    if (function == null)
        throw new NullPointerException();
    if (size > 0 && (tab = table) != null) {
        int mc = modCount;
        for (int i = 0; i < tab.length; ++i) {
            for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                e.value = function.apply(e.key, e.value);
            }
        }
        if (modCount != mc)
            throw new ConcurrentModificationException();
    }
}
```
`replaceAll`结合函数接口,  将每个节点的值转换为自定义方法的结果。 自定义方法以key, value参数，值为value类型

`repaceAll`使用实例：
```java
Map<String, Object> c = new HashMap<>();

c.put("1", 1);
c.put("2", 2);
c.put("3", 3);

c.replaceAll((s, o) -> s + o);
System.out.println(c);// {1=11, 2=22, 3=33}
```

## 类中相关hash计算方法

### key的hash计算
```
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
    key.hashCode() ^ (key.hashCode >>> 16)
&ensp;&ensp;这行代码叫做"扰动函数"。具体作用就是保证`hash()`方法返回值的二进制表示的低位的随机性，尽量减少冲突。
具体原理可以看[JDK 源码中 HashMap 的 hash 方法原理是什么？](https://www.zhihu.com/question/20733617)。
&ensp;&ensp;另外由这个`hash`方法可以看出，`HashMap`中允许key为null。当key为null时会将元素存储在数组0下标处

### 存储位置计算
根据key计算存储的数组下标
```
hash(k) & (n - 1)
```
n为数组长

### 扩容时计算
```
e.hash & oldCap//oldCap 原数组长
```
扩容时的计算机制，根据计算结果判断该节点是否移动至新数组下标处，与存储时的位置计算相配合。

## HashMap中的迭代器

### Values 与 values()
```java
final class Values extends AbstractCollection<V> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<V> iterator()     { return new ValueIterator(); }
    public final boolean contains(Object o) { return containsValue(o); }
    public final Spliterator<V> spliterator() {
        return new ValueSpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super V> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.value);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}

public Collection<V> values() {
    Collection<V> vs = values;
    if (vs == null) {
        // 先初始化空Values对象， 调用其他方法时在实例化其迭代器
        vs = new Values();
        values = vs;
    }
    return vs;
}


```

### KeySet 与 keySet()

```java
public Set<K> keySet() {
    Set<K> ks = keySet;
    if (ks == null) {
        // 先初始化KeySet()对象， 调用其他方法时再实例化迭代器
        ks = new KeySet();
        keySet = ks;
    }
    return ks;
}

final class KeySet extends AbstractSet<K> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<K> iterator()     { return new KeyIterator(); }
    public final boolean contains(Object o) { return containsKey(o); }
    public final boolean remove(Object key) {
        return removeNode(hash(key), key, null, false, true) != null;
    }
    public final Spliterator<K> spliterator() {
        return new KeySpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super K> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.key);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}
```

### EntrySet 和 entrySet()
```java
public Set<Map.Entry<K,V>> entrySet() {
    Set<Map.Entry<K,V>> es;
    return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
}

final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<Map.Entry<K,V>> iterator() {
        return new EntryIterator();
    }
    public final boolean contains(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>) o;
        Object key = e.getKey();
        Node<K,V> candidate = getNode(hash(key), key);
        return candidate != null && candidate.equals(e);
    }
    public final boolean remove(Object o) {
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>) o;
            Object key = e.getKey();
            Object value = e.getValue();
            return removeNode(hash(key), key, value, true, true) != null;
        }
        return false;
    }
    public final Spliterator<Map.Entry<K,V>> spliterator() {
        return new EntrySpliterator<>(HashMap.this, 0, -1, 0, 0);
    }
    public final void forEach(Consumer<? super Map.Entry<K,V>> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e);
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
}
```

### Interator
```
abstract class HashIterator {
    Node<K,V> next;        // next entry to return
    Node<K,V> current;     // current entry
    int expectedModCount;  // for fast-fail
    int index;             // current slot

    HashIterator() {
        // fail-fast 机制， 在iterator遍历过程中不能对hashMap有结构性变化
        expectedModCount = modCount;
        Node<K,V>[] t = table;
        current = next = null;
        index = 0;
        if (t != null && size > 0) { // advance to first entry
            // iterator 初始化时, t[index++] == null只运行一次，即next为t[0]的头节点
            do {} while (index < t.length && (next = t[index++]) == null);
        }
    }

    public final boolean hasNext() {
        return next != null;
    }

    final Node<K,V> nextNode() {
        Node<K,V>[] t;
        Node<K,V> e = next;
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        if ((next = (current = e).next) == null && (t = table) != null) {
            do {} while (index < t.length && (next = t[index++]) == null);
        }
        return e;
    }

    public final void remove() {
        Node<K,V> p = current;
        if (p == null)
            throw new IllegalStateException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        current = null;
        K key = p.key;
        removeNode(hash(key), key, null, false, false);
        expectedModCount = modCount;
    }
}

final class KeyIterator extends HashIterator
    implements Iterator<K> {
    public final K next() { return nextNode().key; }
}

final class ValueIterator extends HashIterator
    implements Iterator<V> {
    public final V next() { return nextNode().value; }
}

final class EntryIterator extends HashIterator
    implements Iterator<Map.Entry<K,V>> {
    public final Map.Entry<K,V> next() { return nextNode(); }
}
```

`expectedModCount = modCount` fail-fast 机制，限制了在iterator创建后到遍历结束的过程中不能对hashMap有结构性变化。 示例如下：
```java
Map<String, Student> c = new HashMap<>();
Student s1 = new Student("1");
System.out.println(s1.equals(s1));
c.put("1", s1);
c.put("2", s1);

Set<Map.Entry<String, Student>> entries = c.entrySet();
Iterator iterator = entries.iterator();
c.put("6", s1);
while (iterator.hasNext()) {
    System.out.println(iterator.next());// 抛出java.util.ConcurrentModificationException
}
```

## 未完待续

**这次看源码对涉及到红黑树的部分进行了跳过，留待日后对红黑树进行更深入了解后再更新。**


    





