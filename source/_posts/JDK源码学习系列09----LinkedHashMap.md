---
layout: w
title: JDK源码学习系列09----LinkedHashMap
date: 2018-04-19 10:54:58
tags:
    - Java
    - JDK
    - LinkedHashMap
    - Map
---

> 之前的博客已经分析了`HashMap`的源码，当遍历其内数据时是无序的，而当我们要按照元素插入的顺序来访问键-值对，就需要用到`LinkedHashMap`。他保持着元素的插入顺序，并可以按照我们的插入顺序进行访问。

<!-- more -->
## LinkedHashMap 类结构

```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
```
`LinkedHashMap`  继承了`HashMap`, 后续仔细阅读源码也可以发现，两者之间有许多相似之处，我们只需要 着眼于他们的不同之处即可。



## LinkedHashMap 的数据结构
![image_1cbdvdq381k4i1kqa11cq15rjos9.png-19.8kB][1]

`LinkedHashMap` 在`HashMap`"数组+链表+红黑数"的基础上添加了双向不循环链表结构，用于记录节点的存入顺序。

具体代码从源码中的基础存储节点就可以看出：
```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```
`Entry` 类在`HashMap`的存储节点的基础上又添加了对前后节点的引用。



## LinkedHashMap 源码分析

&ensp;&ensp; 由于之前已经分析过`HashMap`的源码，因此这次主要对二者间的不同进行着重分析。

### 类的成员属性

除了从父类继承的属性外，还有两个`LinkedHashMap.Entry<K,V>`类型的属性，用于表示双向链表的头尾节点。还有一个布尔类型的属性表示访问顺序。
```java
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>
{ 
    /**
     * 双向链表头节点
     */
    transient LinkedHashMap.Entry<K,V> head;

    /**
     * 双向链表尾节点
     */
    transient LinkedHashMap.Entry<K,V> tail;
    
    /**
     * 访问顺序
     */
    final boolean accessOrder;
}
```

### 构造方法

`LinkedHashMap`的构造方法基本与`HashMap`一致, 在创建对象时都不会对数组进行初始化，而是在创建时进行。同时初始化数组时会指定排序顺序（默认为false）
```
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
    //调用父类HashMap中的putMapentries()方法
    putMapEntries(m, false);
}

public LinkedHashMap(int initialCapacity,
                     float loadFactor,
                     boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}
```

### 重要函数分析

#### newNode
```
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    // 生成Node结点
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    //将节点添加至双向链表尾部
    linkNodeLast(p);
    return p;
}
```
重写了父类`HashMap`中的`newNode()`方法主要加入了`linkedNodeLast()`方法， 将新建的节点添加至双向链表末尾。当`LinkedHashMap`存储数据时，会调用`putValue()`方法，进而调用`newNode`方法。

#### afterNodeAccess
```java
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    //根据调用前的代码可以看出，次数的条件为accessOrder == true 且 添加的数据的key在原来的Map中存在，且访问的节点不是双向链表尾节点
    // 因此这个条件可以看为如果accessOrder == true将更新的节点放至双向链表尾部
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        //将节点的后节点置为空
        p.after = null;
        //如果原节点为头节点，则将后节点变为头节点
        if (b == null)
            head = a;
        else
            // 将当前节点的前节点的后节点指向当前节点的后节点
            b.after = a;
        if (a != null)
            // 将当前节点的后节点的前节点指向当前节点的前节点
            a.before = b;
        else
            last = b;
            
        // 以上操作将当前节点从双向列表中取出
        
        // 将当前节点添加至双向列表尾部
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```
 具象化数据结构如下：
 ![屏幕截图.jpg-31.6kB][2]


  [1]: http://static.zybuluo.com/hewei0928/ft2id7n1x62kmoyeqa3padq7/image_1cbdvdq381k4i1kqa11cq15rjos9.png
  [2]: http://static.zybuluo.com/hewei0928/eyyevf2p1ptq1tmo2pdq3k08/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.jpg
  
#### get
  
```java
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```
当`accessOrder`为`true`时， 也会讲取得的节点放到链表末尾

#### containsValue

```java
public boolean containsValue(Object value) {
    for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
        V v = e.value;
        if (v == value || (value != null && value.equals(v)))
            return true;
    }
    return false;
}
```

`LinkedHashMap` 判断value是否存在时，遍历双向链表去查找。效率与`HashMap`基本一致 