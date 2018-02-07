---
title: JDK源码学习系列03----StringBilder和StringBuffer
date: 2017-12-12 13:02:31
tags:
   - Java 
   - JDK源码

---

>由于前面学习了StringBuffer和StringBuilder的父类AbstractStringBuilder,他们俩的很多方法都是直接super了父类的，也为了较好的比较StringBuffer和StringBuilder，所以把二者放在同一博文中。

<!--more-->

## 一丶 StringBuffer

```java
public final class StringBuffer
extends AbstractStringBuilder
implements java.io.Serializable, CharSequence
```
`final`关键字修饰类， 静态类， 不能被继承。实现了`Serializable`和`CharSequence`接口

### 成员变量
```java
private transient char[] toStringCache;

static final long serialVersionUID = 3388685877147921107L;

char[] value;

int count;
```
toStringCache变量用`transient`关键字修饰，标识该变量反序列化，不会在对象序列化时自动序列化。toString方法调用，用于输出`StringBuff`er中的`char`数组。
```java
public synchronized String toString() {  
    if (toStringCache == null) {  
        toStringCache = Arrays.copyOfRange(value, 0, count);  
    }  
    return new String(toStringCache, true); 
}  
```
可以看出，如果多次连续调用toString方法的时候由于这个字段的缓存就可以少了Arrays.copyOfRange的操作。(每次调用其他的修改StringBuffer对象的方法时，这些方法的第一步都会先将toStringCache设置为null，如下面的append方法等等。）

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
return String.indexOf(value, 0, count,
          str.toCharArray(), 0, str.length(), fromIndex);
}

public synchronized StringBuffer reverse() {
super.reverse();
return this;
}

public synchronized String toString() {
return new String(value, 0, count);
}

private static final java.io.ObjectStreamField[] serialPersistentFields = 
{ 
new java.io.ObjectStreamField("value", char[].class), 
new java.io.ObjectStreamField("count", Integer.TYPE),
new java.io.ObjectStreamField("shared", Boolean.TYPE),
};


private synchronized void writeObject(java.io.ObjectOutputStream s)
throws java.io.IOException {
java.io.ObjectOutputStream.PutField fields = s.putFields();
fields.put("value", value);
fields.put("count", count);
fields.put("shared", false);
s.writeFields();
}


private void readObject(java.io.ObjectInputStream s)
throws java.io.IOException, ClassNotFoundException {
java.io.ObjectInputStream.GetField fields = s.readFields();
value = (char[])fields.get("value", null);
count = (int)fields.get("count", 0);
}
}
```

## 二. StringBuilder

静态类，不能被继承
```java
public final class StringBuilder
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence
```

### 1. 构造器
与`StringBuffer`相同

### 2. 成员方法
与`StringBuffer`类同， 大量用到了`AbstractStringBuilder`内的方法


## 三. 对比
    String, StringBuffer, StringBuilder比较
    
相同点：

 - 1. 三者都是final类

不同点：

 - 1.`String`是不可变类，`StringBuffer`和`StringBuilder`是可变的。
 - 2.`String` 中的成员变量 value,siaze,count都是`final`修饰的，不可改变，而 `StringBuffer`和`StringBuilder ` 同继承于`AbstractStringBuilder`,成员变量没有被`final`修饰。
 - 3.少量数据拼接用`String`的“+”；大量数据多线程时用`StringBuffer`;大量数据单线程时用`StringBuilder`.





