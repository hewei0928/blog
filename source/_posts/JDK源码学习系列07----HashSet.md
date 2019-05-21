﻿---
title: JDK源码学习系列07----HashSet
date: 2017-12-15 15:12:15
tags:
    - Java
    - JDK
    - HashSet
    - Set
---

## 一. HashSet概述

HashSet实现Set接口，由哈希表（实际上是一个HashMap实例）支持。主要具有以下的特点：
&ensp;&ensp;不保证set的迭代顺序，特别是它不保证该顺序恒久不变有且只允许一个null元素；不允许有重复元素，非同步的。

<!--more-->
## 二. HashSet类结构：
```java
public class HashSet<E> extends AbstractSet<E> implements Set<E>, Cloneable, java.io.Serializable
```
`HashSet`实现了`Set`,`Cloneable`,`Serializable`接口,继承`AbstractSet`抽象类。
其父类及接口如下图：
![360桌面截图20171215161511.jpg-48.3kB][1]


  [1]: http://static.zybuluo.com/hewei0928/s00hz2p41rhkgavakgk60nxp/360%E6%A1%8C%E9%9D%A2%E6%88%AA%E5%9B%BE20171215161511.jpg
  
## 三. 成员变量
```java
private transient HashMap<E,Object> map;

private static final Object PRESENT = new Object();
```
从两个成员变量可看出，`HashSet`底层是由`HashMap`(实际是`HashMap`和`LinkedHashMap`)实现数据存储的。、`HashMap`中的key用于存储实际的数据，value则用`HashSet`内定义的一个"虚拟"的static final Object对象填充。

## 四. 构造函数
```java
//默认构造函数，HashMap默认初始容量为16，加载因子为0.75
public HashSet() {
    map = new HashMap<>();
}

//设置set初始容量，大于等于initialCapacity的最小的2的幂次数，加载因子为0.75
public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}

//设置set初始容量及加载因子
public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}

public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}

//底层使用LinkedHashMap实现, LinkedHashMap中的accessOrder，表示数据读取的顺序只与插入顺序有关
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

## 五. 成员方法

### add()
```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```
`HahsSet`的`add()`方法底层调用`HashMap.put`， 由`HashMap`源码可知：

- `key`值可以为`null`; 
- `key`值唯一
- 如果添加时`key`已存在（`oldKey == newKey` 或者 `oldKey.equals(newKey)`），`key`值不会被覆盖。

由此与`HashSet`已知特性一一对应：

- `HashSet`可以存储`null`值
- `HashSet`中的元素不重复
- `HashSet`添加重复元素时不会进行替换

### clear()
```java
public void clear() {
    map.clear();
}
```

### contains()
```
// map中的containsKey 先根据key值去决定在数组中的存储下标，再遍历其中存储的链表去查找对应节点
public boolean contains(Object o) {
    return map.containsKey(o);
}
```

### isEmpty()
```java
public boolean isEmpty() {
    return map.isEmpty();
}
```

### remove()
```java
// HashMap中根据key查找对应数组存储下标，再去其中的链表或树中查找并删除对应节点
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```

### size()
```java
//返回实际存储的数据数
public int size() {
    return map.size();
}
```

### removeAll()、 addAll()、 containsAll()
这几个方法实现在`AbstractCollection`， 基本思想都是遍历然后调用`HashSet`中的`remove`、`add`、`contains`方法

### iterator
```java
public Spliterator<E> spliterator() {
    return new HashMap.KeySpliterator<E,Object>(map, 0, -1, 0, 0);
}
```

## 总结
- `HashSet`底层由`HashMap`实现，数据存储在`HashMap`的key内。
- `HashSet`可以存储null。
- `HashSet`不是线程安全的。
