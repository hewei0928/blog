---
title: JDK源码学习系列02----AbstractStringBuilder
date: 2017-11-30 13:02:31
tags: 
   - Java 
   - JDK
   - String

---

## 前言
>因为看`StringBuffer` 和 `StringBuilder` 的源码时发现两者都继承了`AbstractStringBuilder`，并且很多方法都是直接`super`的父类`AbstractStringBuilder`的方法，所以还是决定先看`AbstractStringBuilder`的源码，然后再看`StringBuffer` 和 `StringBuilder`.

<!--more-->

##方法汇总

### 成员变量
&ensp;&ensp;`AbstractStringBuilder`和`String`一样，在其内部都是以字符数组的形式存储数据的。也就是`String`,`StringBuffer`以及`StringBuilder`在其内部都是以字符数组的形式存储的。

```java
char value[]; //用于存储字符串
int count; //表示字符串的实际长度
```

### 构造函数
&ensp;&ensp;`AbstractStringBuilder`的构造函数中传入的`capacity`是指`char`数组容量，实际长度是以count表示的。
```java
AbstractStringBuilder() {
}

AbstractStringBuilder(int capacity) {
    value = new char[capacity];
}
```

### 成员方法

#### 容量和长度
```java
/**
* 返回字符串长度，即实际存储的字符数量
*/
public int length() {
    return count;
}

/**
* 返回char数组容量
*/
public int capacity() {
    return value.length;
}
```

#### `AbstractStringBuilder`的扩容

```java
public void ensureCapacity(int minimumCapacity) {
    if (minimumCapacity > 0)
        ensureCapacityInternal(minimumCapacity);
}

private void ensureCapacityInternal(int minimumCapacity) {
    // overflow-conscious code
    if (minimumCapacity - value.length > 0) {
        //将char数组进行扩容，并将原数据赋值入新数组内， 浅拷贝
        value = Arrays.copyOf(value,
                newCapacity(minimumCapacity));
    }
}

/**
* 计算扩容的长度 
* 不考虑数组长度超出int长的情况下， 数组长度变为value.length*2+2和minCapacity的较大值
*/
private int newCapacity(int minCapacity) {

    //默认扩容量为原char数组长度*2 + 2;
    int newCapacity = (value.length << 1) + 2;
    //如果传入的容量大于默认扩容量，则传入容量为新容量  
    if (newCapacity - minCapacity < 0) {
        newCapacity = minCapacity;
    }
    
    //newCapacity <= 0 说明(value.length << 1) + 2 操作数值溢出 （>Integer.MAX_VALUE） 且newCapacity小于规定最大值
    return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
        ? hugeCapacity(minCapacity)
        : newCapacity;
}

/**
* 处理超大容量
*/
private int hugeCapacity(int minCapacity) {
    if (Integer.MAX_VALUE - minCapacity < 0) { // overflow
        throw new OutOfMemoryError();
    }
    return (minCapacity > MAX_ARRAY_SIZE)
        ? minCapacity : MAX_ARRAY_SIZE;
}
```

#### 字符串减少存储空间
```java
// 把数组长度削减到最小
public void trimToSize() {
    if (count < value.length) {
        //返回新数组，长度为count
        value = Arrays.copyOf(value, count);
    }
}
```

#### 字符数组设置新长度
```java
public void setLength(int newLength) {
    if (newLength < 0)
        throw new StringIndexOutOfBoundsException(newLength);
    ensureCapacityInternal(newLength);

    if (count < newLength) {//设置长度大于原来长度时，后面部分补以空白字符
        Arrays.fill(value, count, newLength, '\0');
    }

    // 更新字符数量
    count = newLength;
}
```

#### 字符获取，设置
```java
//根据下标获取对应字符
public char charAt(int index) {
    if ((index < 0) || (index >= count))
        throw new StringIndexOutOfBoundsException(index);
    return value[index];
}

// 返回对应下标处的字符对应ASCII码
public int codePointAt(int index) {
    if ((index < 0) || (index >= count)) {
        throw new StringIndexOutOfBoundsException(index);
    }
    return Character.codePointAtImpl(value, index, count);
}

// 返回对应下标前一个字符的ASCII码
public int codePointBefore(int index) {
    int i = index - 1;
    if ((i < 0) || (i >= count)) {
        throw new StringIndexOutOfBoundsException(index);
    }
    return Character.codePointBeforeImpl(value, index, 0);
}

// 获取子串，把它拷贝到目标字符数组指定起始位置 （不包括srcEnd下标）
public void getChars(int srcBegin, int srcEnd, char[] dst, int dstBegin)
{
    if (srcBegin < 0)
        throw new StringIndexOutOfBoundsException(srcBegin);
    if ((srcEnd < 0) || (srcEnd > count))
        throw new StringIndexOutOfBoundsException(srcEnd);
    if (srcBegin > srcEnd)
        throw new StringIndexOutOfBoundsException("srcBegin > srcEnd");
    System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
}


//字符数组中指定下标字符替换 index:[0~count]
public void setCharAt(int index, char ch) {
    if ((index < 0) || (index >= count))
        throw new StringIndexOutOfBoundsException(index);
    value[index] = ch;
}
```


#### 比较常用的各类append方法

```java
public AbstractStringBuilder append(Object obj) {
    return append(String.valueOf(obj));//如果obj==null则返回"null"， 否则调用obj.toString()方法
}

public AbstractStringBuilder append(String str) {
    if (str == null) //如果为空则添加"null"
        return appendNull();
    int len = str.length();
    ensureCapacityInternal(count + len);// count + len > 数组长则进行扩容
    str.getChars(0, len, value, count);//将str中的char数组添加至value数组末尾
    count += len;//改变字符串实际长度
    return this;
}

public AbstractStringBuilder append(StringBuffer sb) {
    if (sb == null)
        return appendNull();
    int len = sb.length();
    ensureCapacityInternal(count + len);
    sb.getChars(0, len, value, count);
    count += len;
    return this;
}

// 末尾追加AbstractStringBuilder引用指向的对象
AbstractStringBuilder append(AbstractStringBuilder asb) {
    if (asb == null)
        return appendNull();
    int len = asb.length();//返回asb内char数组实际字符数
    ensureCapacityInternal(count + len);//char数组扩容
    asb.getChars(0, len, value, count);//在value数组末尾添加asb内的char数组
    count += len;
    return this;
}

// 末尾追加CharSequence引用指向的对象
public AbstractStringBuilder append(CharSequence s) {
    if (s == null)
        return appendNull();
    if (s instanceof String)
        return this.append((String)s);
    if (s instanceof AbstractStringBuilder)
        return this.append((AbstractStringBuilder)s);

    return this.append(s, 0, s.length());//将charsequence添加至value尾部
}

// 将字符序列start(包括)到end(不包括)中的字符添加至末尾
public AbstractStringBuilder append(CharSequence s, int start, int end) {
    if (s == null)
        s = "null";
    if ((start < 0) || (start > end) || (end > s.length()))
        throw new IndexOutOfBoundsException(
            "start " + start + ", end " + end + ", s.length() "
            + s.length());
    int len = end - start;
    ensureCapacityInternal(count + len);
    for (int i = start, j = count; i < end; i++, j++)
        value[j] = s.charAt(i);
    count += len;
    return this;
}


public AbstractStringBuilder append(char[] str) {
    int len = str.length;
    ensureCapacityInternal(count + len);
    System.arraycopy(str, 0, value, count, len);
    count += len;
    return this;
}

// 包括str的offset下标， 不包括offset+len下标
public AbstractStringBuilder append(char str[], int offset, int len) {
    if (len > 0)                // let arraycopy report AIOOBE for len < 0
        ensureCapacityInternal(count + len);
    System.arraycopy(str, offset, value, count, len);
    count += len;
    return this;
}

public AbstractStringBuilder append(boolean b) {
    if (b) {
        ensureCapacityInternal(count + 4);
        value[count++] = 't';
        value[count++] = 'r';
        value[count++] = 'u';
        value[count++] = 'e';
    } else {
        ensureCapacityInternal(count + 5);
        value[count++] = 'f';
        value[count++] = 'a';
        value[count++] = 'l';
        value[count++] = 's';
        value[count++] = 'e';
    }
    return this;
}

public AbstractStringBuilder append(char c) {
    ensureCapacityInternal(count + 1);
    value[count++] = c;
    return this;
}


public AbstractStringBuilder append(int i) {
    if (i == Integer.MIN_VALUE) {//为什么单独拎出最小值来添加？？？
        append("-2147483648");
        return this;
    }
    int appendedLength = (i < 0) ? Integer.stringSize(-i) + 1
                                 : Integer.stringSize(i);
    int spaceNeeded = count + appendedLength;
    ensureCapacityInternal(spaceNeeded);
    Integer.getChars(i, spaceNeeded, value);
    count = spaceNeeded;
    return this;
}

public AbstractStringBuilder append(long l) {
    if (l == Long.MIN_VALUE) {
        append("-9223372036854775808");
        return this;
    }
    int appendedLength = (l < 0) ? Long.stringSize(-l) + 1
                                 : Long.stringSize(l);
    int spaceNeeded = count + appendedLength;
    ensureCapacityInternal(spaceNeeded);
    Long.getChars(l, spaceNeeded, value);
    count = spaceNeeded;
    return this;
}

public AbstractStringBuilder append(float f) {
    FloatingDecimal.appendTo(f,this);
    return this;
}

public AbstractStringBuilder append(double d) {
    FloatingDecimal.appendTo(d,this);
    return this;
}

/**
* append null
*/
private AbstractStringBuilder appendNull() {
    int c = count;
    ensureCapacityInternal(c + 4);
    // 数组对象及变量在堆栈中的存储需加深了解
    final char[] value = this.value;
    value[c++] = 'n';
    value[c++] = 'u';
    value[c++] = 'l';
    value[c++] = 'l';
    count = c;
    return this;
}
```

#### 字符删除 delete

```java
/**
*   char数组部分去除
*   
*/
public AbstractStringBuilder delete(int start, int end) {
    if (start < 0)
        throw new StringIndexOutOfBoundsException(start);
    if (end > count)
        end = count;
    if (start > end)
        throw new StringIndexOutOfBoundsException();
    int len = end - start;
    if (len > 0) {
        // 复制了之后数组后也许会多出重复的数据，但由于count的限制，外部无法读取这些数据，因此可以当成不存在
        System.arraycopy(value, start+len, value, start, count-end);//从下标为start(包括)删除到下标为end(不包括)
        count -= len;
    }
    return this;
}


public AbstractStringBuilder deleteCharAt(int index) {
    if ((index < 0) || (index >= count))
        throw new StringIndexOutOfBoundsException(index);
    System.arraycopy(value, index+1, value, index, count-index-1);
    count--;
    return this;
}
```

#### 字符串替换 replace
```java
/**
* 下标start(包括)至end(不包括)替换为str
*/
public AbstractStringBuilder replace(int start, int end, String str) {
    if (start < 0)
        throw new StringIndexOutOfBoundsException(start);
    if (start > count)
        throw new StringIndexOutOfBoundsException("start > length()");
    if (start > end)
        throw new StringIndexOutOfBoundsException("start > end");

    if (end > count)
        end = count;
    int len = str.length();
    int newCount = count + len - (end - start);
    ensureCapacityInternal(newCount);

    System.arraycopy(value, end, value, start + len, count - end);
    str.getChars(value, start);
    count = newCount;
    return this;
}
```

#### 字符串截取 substring
```java
// 包括start下标处的字符
public String substring(int start) {
    return substring(start, count);
}

// 
public CharSequence subSequence(int start, int end) {
    return substring(start, end);
}

/**
* char数组截取
* start下标（包括） 至 end下标（不包括） 
*/
public String substring(int start, int end) {
    if (start < 0)
        throw new StringIndexOutOfBoundsException(start);
    if (end > count)
        // end超出的话会报越界错误
        throw new StringIndexOutOfBoundsException(end);
    if (start > end)
        throw new StringIndexOutOfBoundsException(end - start);
    return new String(value, start, end - start);//构造函数中调用Arrays.copyOfRange())进行数组截取
    // 最后相当于调用了System.arraycopy(value, start, 新数组, 0, end-start)
}
```

#### 插入字符 insert
```java
//普通对象插入
public AbstractStringBuilder insert(int offset, Object obj) {
    return insert(offset, String.valueOf(obj));//调用obj.toString()
}


//从下标offset处起(包括)插入str
public AbstractStringBuilder insert(int offset, String str) {
    if ((offset < 0) || (offset > length()))
        throw new StringIndexOutOfBoundsException(offset);
    if (str == null)
        str = "null";
    int len = str.length();
    ensureCapacityInternal(count + len);//数组扩容len长度
    System.arraycopy(value, offset, value, offset + len, count - offset);//从下标offset处起（包括）将后部char数组后移至len格
    str.getChars(value, offset);//将str内的整个char数组复制入value数组内, 存放位置从下标offset处起（包括）
    count += len;//实际字符数增加
    return this;
}


//value数组在下标index字符之前插入str数组从offset下标起（包括）len长度的字符
public AbstractStringBuilder insert(int index, char[] str, int offset,int len)
{
    if ((index < 0) || (index > length()))
        throw new StringIndexOutOfBoundsException(index);
    if ((offset < 0) || (len < 0) || (offset > str.length - len))
        throw new StringIndexOutOfBoundsException(
            "offset " + offset + ", len " + len + ", str.length "
            + str.length);
    ensureCapacityInternal(count + len);//扩容
    System.arraycopy(value, index, value, index + len, count - index);//value数组从index位置处（包括）右移len格
    System.arraycopy(str, offset, value, index, len);//str从offset处起len长度个char字符复制入value数组内；
    count += len;
    return this;
}


//value从下标offset处起（包括）插入字符数组str
public AbstractStringBuilder insert(int offset, char[] str) {
    if ((offset < 0) || (offset > length()))
        throw new StringIndexOutOfBoundsException(offset);
    int len = str.length;
    ensureCapacityInternal(count + len);
    System.arraycopy(value, offset, value, offset + len, count - offset);
    // 复制str数组入value数组内
    System.arraycopy(str, 0, value, offset, len);
    count += len;
    return this;
}


public AbstractStringBuilder insert(int dstOffset, CharSequence s) {
    if (s == null)
        s = "null";
    if (s instanceof String)
        return this.insert(dstOffset, (String)s);
    return this.insert(dstOffset, s, 0, s.length());
}


//value从下标offset处起（包括）插入s的[start,end)部分字符
public AbstractStringBuilder insert(int dstOffset, CharSequence s,
                                     int start, int end) {
    if (s == null)
        s = "null";
    if ((dstOffset < 0) || (dstOffset > this.length()))
        throw new IndexOutOfBoundsException("dstOffset "+dstOffset);
    if ((start < 0) || (end < 0) || (start > end) || (end > s.length()))
        throw new IndexOutOfBoundsException(
            "start " + start + ", end " + end + ", s.length() "
            + s.length());
    int len = end - start;//插入数组长度
    ensureCapacityInternal(count + len);
    System.arraycopy(value, dstOffset, value, dstOffset + len,
                     count - dstOffset);//从dstOffset处右移len长度
    for (int i=start; i<end; i++)//[start,end)字符插入value数组内
        value[dstOffset++] = s.charAt(i);
    count += len;
    return this;
}

public AbstractStringBuilder insert(int offset, boolean b) {
    return insert(offset, String.valueOf(b));
}


public AbstractStringBuilder insert(int offset, char c) {
    ensureCapacityInternal(count + 1);
    System.arraycopy(value, offset, value, offset + 1, count - offset);
    value[offset] = c;
    count += 1;
    return this;
}


public AbstractStringBuilder insert(int offset, int i) {
    return insert(offset, String.valueOf(i));
}

//省略基础数据类型的插入，类比插入int类型
```


#### 字符位置获取 indexOf()/lastIndexOf()
```java
// 获取子串顺序下首次出现的位置
public int indexOf(String str) {
    return indexOf(str, 0);
}

// 获取子串顺序下从fromIndex开始（包括）首次出现的位置
public int indexOf(String str, int fromIndex) {
    //value数组在0-count范围内从下标fromIndex起（包括）首次出现str的下标
    return String.indexOf(value, 0, count, str, fromIndex);
}

public int lastIndexOf(String str) {
    return lastIndexOf(str, count);
}


public int lastIndexOf(String str, int fromIndex) {
    return String.lastIndexOf(value, 0, count, str, fromIndex);
}
```


#### 字符数组倒序排列 reverse()

```java
public AbstractStringBuilder reverse() {
    boolean hasSurrogates = false;
    int n = count - 1;// 最大下标
    // 折半交换
    // 如果存在奇数个字符，则跳过中间字符
    //巧妙的循环方式 
    for (int j = (n-1) >> 1; j >= 0; j--) {
        int k = n - j; 
        char cj = value[j];
        char ck = value[k];
        value[j] = ck;
        value[k] = cj;
        if (Character.isSurrogate(cj) ||
            Character.isSurrogate(ck)) {
            hasSurrogates = true;
        }
    }
    if (hasSurrogates) {
        reverseAllValidSurrogatePairs();
    }
    return this;
}


private void reverseAllValidSurrogatePairs() {
    for (int i = 0; i < count - 1; i++) {
        char c2 = value[i];
        if (Character.isLowSurrogate(c2)) {
            char c1 = value[i + 1];
            if (Character.isHighSurrogate(c1)) {
                value[i++] = c1;
                value[i] = c2;
            }
        }
    }
}
```









