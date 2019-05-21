---
title: JDK源码学习系列03----StringBilder和StringBuffer
date: 2017-12-03 13:02:31
tags:
   - Java 
   - JDK
   - String
---

>由于前面学习了`StringBuffer`和`StringBuilder`的父类`AbstractStringBuilder`,他们俩的很多方法都是直接`super()`调用父类的方法，也为了较好的比较`StringBuffer`和`StringBuilder`，所以把二者放在同一博文中。

<!--more-->

##  StringBuffer

```java
public final class StringBuffer
extends AbstractStringBuilder
implements java.io.Serializable, CharSequence
```
`final`关键字修饰类， 不能被继承。实现了`Serializable`和`CharSequence`接口

### 成员变量
```java
private transient char[] toStringCache;

static final long serialVersionUID = 3388685877147921107L;

char[] value;

int count;
```
toStringCache变量用`transient`关键字修饰，标识该变量反序列化，不会在对象序列化时自动序列化。toString方法调用，用于输出`StringBuffer`中的`char`数组。
```java
public synchronized String toString() {  
    if (toStringCache == null) {  
        toStringCache = Arrays.copyOfRange(value, 0, count);  
    }  
    return new String(toStringCache, true); 
}  
```
可以看出，如果多次连续调用`toString`方法的时候由于这个字段的缓存就可以少了`Arrays.copyOfRange`的操作。(每次调用其他的修改`StringBuffer`对象的方法时，这些方法的第一步都会先将toStringCache设置为`null`，如`append`方法等等。）

value， count属性权限为包权限，继承自`AbstractStringBuilder`

### 构造函数
```java
//初始化char数组长度为16
public StringBuffer() {
    super(16);
}


public StringBuffer(int capacity) {
    super(capacity);
}

//若初始化StringBuffer时传入字符串，则 容量 为字符串长度+默认容量16  
public StringBuffer(String str) {
    super(str.length() + 16);
    append(str);
}

public StringBuffer(CharSequence seq) {
    this(seq.length() + 16);
    append(seq);
}
```
### 成员方法
&ensp;&ensp;方法大部分定义为`synchronized`, 这是`StringBuffer`和`StringBuilder`最大的区别。`StringBuffer`是同步的，线程安全的，而`StringBuilder`是非线程安全的。
```java
public synchronized int length() {
return count;
}

public synchronized int capacity() {
return value.length;
}


public synchronized void ensureCapacity(int minimumCapacity) {
    if (minimumCapacity > value.length) {
        expandCapacity(minimumCapacity);
    }
}

public synchronized void trimToSize() {
    super.trimToSize();
}

public synchronized void setLength(int newLength) {
    super.setLength(newLength);
}

public synchronized char charAt(int index) {
    if ((index < 0) || (index >= count))
        throw new StringIndexOutOfBoundsException(index);
    return value[index];
}

public synchronized StringBuffer append(Object obj) {
    super.append(String.valueOf(obj));
    return this;
}

public synchronized StringBuffer append(String str) {
    super.append(str);
    return this;
}

public synchronized StringBuffer append(StringBuffer sb) {
    super.append(sb);
    return this;
}

public StringBuffer append(CharSequence s) {
    // Note, synchronization achieved via other invocations
    if (s == null)
    s = "null";
    if (s instanceof String)
        return this.append((String)s);
    if (s instanceof StringBuffer)
        return this.append((StringBuffer)s);
    return this.append(s, 0, s.length());
}

public synchronized StringBuffer delete(int start, int end) {
    super.delete(start, end);
    return this;
}


public synchronized StringBuffer deleteCharAt(int index) {
    super.deleteCharAt(index);
    return this;
}

public synchronized String substring(int start) {
    return substring(start, count);
}

public int indexOf(String str) {
    return indexOf(str, 0);
}

public synchronized int indexOf(String str, int fromIndex) {
    return String.indexOf(value, 0, count, str.toCharArray(), 0, str.length(), fromIndex);
}

public synchronized StringBuffer reverse() {
    super.reverse();
    return this;
}

//toString 方法添加了一个缓存，
// 优化在不改变StringBuffer对象的情况下多次连续调用toString
public synchronized String toString() {
    if (toStringCache == null) {
        toStringCache = Arrays.copyOfRange(value, 0, count);
    }
    return new String(toStringCache, true);
}

```

##  StringBuilder

final类，不能被继承
```java
public final class StringBuilder
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence
```

### 构造器
与`StringBuffer`相同

### 成员方法
与`StringBuffer`类同， 大量用到了`AbstractStringBuilder`内的方法


## 对比
    String, StringBuffer, StringBuilder比较
    
相同点：

 - 1. 三者都是final类

不同点：

 - 1.`String`是不可变类，`StringBuffer`和`StringBuilder`是可变的。
 - 2.`String` 中的成员变量 value,siaze,count都是`final`修饰的，不可改变，而 `StringBuffer`和`StringBuilder`同继承于`AbstractStringBuilder`,成员变量没有被`final`修饰。
 - 3.单条语句拼接用`String`的“+”；要考虑线程安全的情况下循环中出现字符串拼接时用`StringBuffer`;不用考虑线程安全的情况下循环中出现字符串拼接时用`StringBuilder`.





