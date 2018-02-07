---
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
从两个成员变量可看出，`HashSet`底层是由`HashMap`实现数据存储的。、`HashMap`中的key用于存储实际的数据，value则用`HashSet`内定义的一个"虚拟"的static final Object对象填充。

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

//底层使用LinkedHashMap实现
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

## 五. 成员方法

```java
public Iterator<E> iterator() {
    return map.keySet().iterator();
}

public int size() {
    return map.size();
}

public boolean isEmpty() {
    return map.isEmpty();
}

public boolean contains(Object o) {
    return map.containsKey(o);
}


//从此方法可知HashSet可以存储null值，但是不可重复，因为null的hashcode必然为0
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}

public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}

public void clear() {
    map.clear();
}
```
`HashSet`底层方法多采用了`HashMap`的方法具体见`HsahMap`的源码阅读博客吧。

## 总结
- `HashSet`底层由`HashMap`实现，数据存储在`HashMap`的key内。
- `HashSet`可以存储null，但是会被覆盖。
- `HashSet`不是线程安全的。
