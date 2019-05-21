---
title: JDK源码学习系列16----System.arraycopy与对象的复制
date: 2018-07-26 09:28:33
tags:
    - Java
    - JDK
---

&ensp;&ensp;观察源码可以发现类中大量方法用到了`System.arraycopy()`方法:
```java
/**
*   src:源数组;
*   srcPos:源数组要复制的起始位置; 起始下标，包括
*   dest:目的数组;
*   destPos:目的数组放置的起始位置; 起始下标，包括
*   length:复制的长度; 不包括下标length+start
*/
public static void arraycopy(Object src,
                             int srcPos,
                             Object dest,
                             int destPos,
                             int length)
```
