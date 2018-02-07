---
layout: 'n'
title: JDK源码学习系列06----HashMap
date: 2017-12-15 09:26:47
tags:
    - Java
    - HashMap
    - JDK
---

## 一. HashMap简介
&ensp;&ensp;HashMap是Java程序员使用频率最高的用于映射(键值对)处理的数据类型。
网上关于`HashMap`的博客多是关于JDK1.6的。在JDK1.6中，HashMap采用桶位+链表实现，即使用链表处理冲突，同一hash值的元素都存储在一个链表里。但是当位于一个桶中的元素较多，即hash值相等的元素较多时，通过key值依次查找的效率较低。而JDK1.8中，HashMap采用位桶+链表+红黑树实现，当链表长度超过阈值（8）时，将链表转换为红黑树，这样大大减少了查找时间。

<!--more-->

## 二. HashMap结构

![360桌面截图20171215095906.jpg-30.2kB][1]



  [1]: http://static.zybuluo.com/hewei0928/pkntk97vux052q5sxye8so6l/360%E6%A1%8C%E9%9D%A2%E6%88%AA%E5%9B%BE20171215095906.jpg
  
```java
public class HashMap<K,V> extends AbstractMap<K,V>
implements Map<K,V>, Cloneable, Serializable
```
&ensp;&ensp;HashMap 是一个散列表，它存储的内容是键值对(key-value)映射。
&ensp;&ensp;HashMap 继承于AbstractMap，实现了Map、Cloneable、java.io.Serializable接口。
&ensp;&ensp;Map接口定义了所有Map子类必须实现的方法。Map接口中还定义了一个内部接口Entry（为什么要弄成内部接口？改天还要学习学习）。

## 三. 成员变量
```java
//默认初始容量
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

//每次结构更改时的计数器
transient int modCount;

//临界值 当实际大小(容量*填充因子)超过临界值时，会进行扩容，一定为2的幂次数
int threshold;

//填充因子
final float loadFactor;
```
table是一个Entry[]数组类型，而Entry实际上就是一个单向链表。哈希表的"key-value键值对"都是存储在Entry数组中的。 size是HashMap的大小，它是HashMap保存的键值对的数量。 
threshold是HashMap的阈值，用于判断是否需要调整HashMap的容量。threshold的值="容量*加载因子"，当HashMap中                    存储数据的数量达到threshold时，就需要将HashMap的容量加倍。
loadFactor就是加载因子。
modCount是用来实现fail-fast机制的。


## 四. 构造函数
```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;//填充因子设为0.75
}
```
默认构造函数，初始化`HashMap`，并将填充因子设为0.75

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
    this.threshold = tableSizeFor(initialCapacity);
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```
传入临界值与填充因子初始化`HashMap`。 tableSizeFor用于计算临界值。tableSizeFor(initialCapacity)返回大于等于initialCapacity的最小的二次幂数。

```java
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}

final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
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

## 五. 成员函数
### (1). put()
```java
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
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}

```

。。。有点难 带我去看过红黑树再来详读。