---
title: JDK源码学习系列02----AbstractStringBuilder
date: 2017-11-30 13:02:31
tags: 
   - Java 
   - JDK源码

---

## 前言
>因为看`StringBuffer` 和 `StringBuilder` 的源码时发现两者都继承了`AbstractStringBuilder`，并且很多方法都是直接`super`的父类`AbstractStringBuilder`的方法，所以还是决定先看`AbstractStringBuilder`的源码，然后再看`StringBuffer` 和 `StringBuilder`.

<!--more-->

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

##方法汇总

### 1. 成员变量
&ensp;&ensp;`AbstractStringBuilder`和`String`一样，在其内部都是以字符数组的形式实现的。也就是`String,StringBuffer`以及`StringBuilder`在其内部都是以字符数组的形式实现的。

```java
char value[]; //用于存储字符串
int count; //表示字符串的实际长度
```

### 2. 构造函数
&ensp;&ensp;`AbstractStringBuilder`的构造函数中传入的`capacity`是指`char`数组容量，实际长度是以count表示的。
```java
AbstractStringBuilder() {
}

AbstractStringBuilder(int capacity) {
    value = new char[capacity];
}
```

### 3. 容量和长度
```java
/**
* 返回字符串长度
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

### 4. `AbstractStringBuilder`的扩容
```java
public void ensureCapacity(int minimumCapacity) {
    if (minimumCapacity > 0)
        ensureCapacityInternal(minimumCapacity);
}

    
private void ensureCapacityInternal(int minimumCapacity) {
    // overflow-conscious code
    if (minimumCapacity - value.length > 0) {
        //将char数组进行扩容，并将原数据赋值入新数组内
        value = Arrays.copyOf(value,
                newCapacity(minimumCapacity));
    }
}

    

/**
* 计算扩容的长度
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

### 5. 字符串减少存储空间
```java
// 把数组长度削减到最小
public void trimToSize() {
    if (count < value.length) {
        value = Arrays.copyOf(value, count);
    }
}
```

### 6. 字符数组设置新长度
```java
public void setLength(int newLength) {
    if (newLength < 0)
        throw new StringIndexOutOfBoundsException(newLength);
    ensureCapacityInternal(newLength);

    if (count < newLength) {//设置长度大于原来长度时，后面部分补以空白 
        Arrays.fill(value, count, newLength, '\0');
    }

    // 更新字符数量
    count = newLength;
}
```

### 7. `charAt()`
```java
//根据下标获取对应字符
public char charAt(int index) {
    if ((index < 0) || (index >= count))
        throw new StringIndexOutOfBoundsException(index);
    return value[index];
}

// 获取子串，把它拷贝到目标字符数组指定起始位置
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


//字符数组中指定下标字符替换 index:[0~value)
public void setCharAt(int index, char ch) {
    if ((index < 0) || (index >= count))
        throw new StringIndexOutOfBoundsException(index);
    value[index] = ch;
}
```


### 8. 比较重要的append方法

各类`append`方法

```java
public AbstractStringBuilder append(Object obj) {
    return append(String.valueOf(obj));//如果obj==null则返回"null"
}

public AbstractStringBuilder append(String str) {
    if (str == null) //如果为空则添加"null"
        return appendNull();
    int len = str.length();
    ensureCapacityInternal(count + len);//value数组扩容
    str.getChars(0, len, value, count);//将str中的char数组添加至value数组末尾
    count += len;//改变字符串实际长度
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

//感觉这个方法和AbstractStringBuilder类中其他方法的代码风格格格不入，换人了？
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
    if (i == Integer.MIN_VALUE) {
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


/**
* append null
*/
private AbstractStringBuilder appendNull() {
    int c = count;
    ensureCapacityInternal(c + 4);
    final char[] value = this.value;
    value[c++] = 'n';
    value[c++] = 'u';
    value[c++] = 'l';
    value[c++] = 'l';
    count = c;
    return this;
}
```


### 9. delete方法

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

### 10. 字符串替换
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

### 11. subString()
```java
public String substring(int start) {
    return substring(start, count);
}

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
        throw new StringIndexOutOfBoundsException(end);
    if (start > end)
        throw new StringIndexOutOfBoundsException(end - start);
    return new String(value, start, end - start);//构造函数中调用Arrays.copyOfRange([))进行数组截取
}
```

### 12. insert()
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
    str.getChars(value, offset);//从offset处起（包括）将str内的char数组复制入value数组内
    count += len;//实际字符数增加
    return this;
}


//value数组在下标index出（包括）插入str数组从offset位置起（包括）len长度的字符
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

//省略基础数据类型的插入，类比Object
```


### 13. indexOf()
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


### 14. reverse()
字符数组倒序排列
```java
public AbstractStringBuilder reverse() {
    boolean hasSurrogates = false;
    int n = count - 1;
    // 折半交换
    // 如果存在奇数个字符，则跳过中间字符
    for (int j = (n-1) >> 1; j >= 0; j--) {
        int k = n - j; //巧妙的循环方式
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









