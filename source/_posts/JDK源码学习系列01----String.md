---
title: JDK源码学习系列01----String
date: 2017-12-11 13:02:31
tags: 
   - Java 
   - JDK源码

---

>## 写在最前面
&ensp;&ensp;这是我JDK源码学习系列的第一篇博文，我知道源码学习这条路很难坚持，但是我始终相信，不积跬步无以至千里。初步计划是每天早上晨会前看一个类的源码，虽然工作比较忙，但时间就像那个，挤挤总是有的嘛~~晚上回来写写博文，一是加深理解二来也加深记忆方便以后查阅。学习的步骤当然是先从自己已经用的非常熟练的类入手。

<!--more-->

## java.lang.String
```java
public final class String
    implements java.io.Serializable, 
               Comparable<String>, 
               CharSequence
```

`String` 是`final`修饰的类，不可以被继承，可以被序列化，实现了`Comparable`, `CharSequence`接口

## 1. 成员变量
```java
private final char value[];//final修饰， 说明String不可变
private int hash; 
private static final long serialVersionUID = -6849794470754667710L;
private static final ObjectStreamField[] serialPersistentFields = new ObjectStreamField[0];
```

`String`是以字符数组的形式实现的，有`final`关键字修饰，说明字符串一旦初始化就不可修改。这也是与其与`StringBuffer`和`StringBuilder`的区别。`hash`则用来存放计算后的哈希值。

## 2. 构造函数
### (1). 无参构造 
```java
public String() {
    this.value = "".value;
}
```
构造一个0字符的字符串，由于String是不可变的，没什么大用

### (2). String参数
```java
public String(String original) {
    this.value = original.value;
    this.hash = original.hash;
}
```

### (3). char[]参数
```java
public String(char value[]) {
    this.value = Arrays.copyOf(value, value.length);//复制， 也解释了this.value无法修改，而value可以改变的矛盾
}
```
`Arrays.copyOf`方法中新建了一个`char`数组，并将`value`中的值复制入新建的数组中，返回对新建数组的引用，`this.value`指向这个新建的数组


```java
//char数组下标offset起（包括）截取count个构建字符串
public String(char value[], int offset, int count) {
    if (offset < 0) {
        throw new StringIndexOutOfBoundsException(offset);
    }
    if (count <= 0) {
        if (count < 0) {
            throw new StringIndexOutOfBoundsException(count);
        }
        if (offset <= value.length) {
            this.value = "".value;
            return;
        }
    }
    // Note: offset or count might be near -1>>>1.
    if (offset > value.length - count) {
        throw new StringIndexOutOfBoundsException(offset + count);
    }
    this.value = Arrays.copyOfRange(value, offset, offset+count);
}
```
同上，调用`Arrays.copyOfRange`新建了一个`char`数组，进行对`value`进行截取复制。传回引用至`this.value`。

### (3). byte[]
```java
public String(byte bytes[], int offset, int length) {
    checkBounds(bytes, offset, length);
    this.value = StringCoding.decode(bytes, offset, length);
}

public String(byte bytes[]) {
    this(bytes, 0, bytes.length);
}
```

### (4). StringBuilder和StringBuffer参数
```java
public String(StringBuffer buffer) {
    synchronized(buffer) {
        this.value = Arrays.copyOf(buffer.getValue(), buffer.length());
    }
}
    
public String(StringBuilder builder) {
    this.value = Arrays.copyOf(builder.getValue(), builder.length());
}
```
原理类似于`char[]`参数的构造函数


## 3. 重要方法
### (1). length()
```java
public int length() {
    return value.length;
}
```
获取字符串中包含的字符数， 即字符数组的长度
### (2). isEmpty
```java
public boolean isEmpty() {
    return value.length == 0;
}
```
判断字符串是否为空，即判断value数组的长度为0即可 
### (3). charAt()
```java
public char charAt(int index) {
    if ((index < 0) || (index >= value.length)) {
        throw new StringIndexOutOfBoundsException(index);
    }
    return value[index];
}
```
返回`index`下标指向的字符

### (4) getChars()
```java
//str.getChars()调用此方法将str的字符数组从dstBegin下标处起始复制到dst内
void getChars(char dst[], int dstBegin) {
    System.arraycopy(value, 0, dst, dstBegin, value.length);
}
```
此方法`AbstractStringBuilder`类中用到


```java
//将string中value字符数组srcBegin（包括）到srcEnd(不包括)内的字符复制到dst中，下标由dstBegin起始
public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {
    if (srcBegin < 0) {
        throw new StringIndexOutOfBoundsException(srcBegin);
    }
    if (srcEnd > value.length) {
        throw new StringIndexOutOfBoundsException(srcEnd);
    }
    if (srcBegin > srcEnd) {
        throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
    }
    System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
}
```

### (5). getBytes()
```java
//根据指定的decode编码返回某字符串在该编码下的byte数组表示
public byte[] getBytes(String charsetName)
        throws UnsupportedEncodingException {
    if (charsetName == null) throw new NullPointerException();
    return StringCoding.encode(charsetName, value, 0, value.length);
}

//字符串转为byte数组
public byte[] getBytes() {
    return StringCoding.encode(value, 0, value.length);
}
```

### (6). equals()
```java
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String)anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i])
                    return false;
                i++;
            }
            return true;
        }
    }
    return false;
}



public boolean equalsIgnoreCase(String anotherString) {
    return (this == anotherString) ? true
            : (anotherString != null)
            && (anotherString.value.length == value.length)
            && regionMatches(true, 0, anotherString, 0, value.length);
}
```
`String`类的equals方法首先比较两个字符串是否为同一个对象，为否的话继续遍历比较两个字符串对象内的`char`数组是否完全相同。

```java
public boolean contentEquals(StringBuffer sb) {
    return contentEquals((CharSequence)sb);
}

public boolean contentEquals(CharSequence cs) {
    // Argument is a StringBuffer, StringBuilder
    if (cs instanceof AbstractStringBuilder) {
        if (cs instanceof StringBuffer) {
            synchronized(cs) {
               return nonSyncContentEquals((AbstractStringBuilder)cs);
            }
        } else {
            return nonSyncContentEquals((AbstractStringBuilder)cs);
        }
    }
    // Argument is a String
    if (cs instanceof String) {
        return equals(cs);
    }
    // Argument is a generic CharSequence
    char v1[] = value;
    int n = v1.length;
    if (n != cs.length()) {
        return false;
    }
    for (int i = 0; i < n; i++) {
        if (v1[i] != cs.charAt(i)) {
            return false;
        }
    }
    return true;
}


private boolean nonSyncContentEquals(AbstractStringBuilder sb) {
    char v1[] = value;
    char v2[] = sb.getValue();
    int n = v1.length;
    if (n != sb.length()) {
        return false;
    }
    for (int i = 0; i < n; i++) {
        if (v1[i] != v2[i]) {
            return false;
        }
    }
    return true;
}
```

### (7). ComapreTo()
```java
public int compareTo(String anotherString) {
    int len1 = value.length;
    int len2 = anotherString.value.length;
    int lim = Math.min(len1, len2);
    char v1[] = value;
    char v2[] = anotherString.value;

    int k = 0;
    while (k < lim) {
        char c1 = v1[k];
        char c2 = v2[k];
        if (c1 != c2) {
            return c1 - c2;
        }
        k++;
    }
    return len1 - len2;
}

```
比较两个字符串的大小，顺序遍历两个字符串的`char`数组，如果有不相等的字符，则返回结果。否则则比较两个字符串的长度。

大小写不敏感的比较
```java
public static final Comparator<String> CASE_INSENSITIVE_ORDER
                                     = new CaseInsensitiveComparator();
private static class CaseInsensitiveComparator
        implements Comparator<String>, java.io.Serializable {

    private static final long serialVersionUID = 8575799808933029326L;

    public int compare(String s1, String s2) {
        int n1 = s1.length();
        int n2 = s2.length();
        int min = Math.min(n1, n2);
        for (int i = 0; i < min; i++) {
            char c1 = s1.charAt(i);
            char c2 = s2.charAt(i);
            if (c1 != c2) {
                c1 = Character.toUpperCase(c1);
                c2 = Character.toUpperCase(c2);
                if (c1 != c2) {
                    c1 = Character.toLowerCase(c1);
                    c2 = Character.toLowerCase(c2);
                    if (c1 != c2) {
                        // No overflow because of numeric promotion
                        return c1 - c2;
                    }
                }
            }
        }
        return n1 - n2;
    }

    /** Replaces the de-serialized object. */
    private Object readResolve() { return CASE_INSENSITIVE_ORDER; }
}

public int compareToIgnoreCase(String str) {
    return CASE_INSENSITIVE_ORDER.compare(this, str);
}
```

比较两个字符是否大小写不敏感的相等需要`toUpperCase`,`toLowerCase`同时相等
```java
char ch1 = (char) 73; //LATIN CAPITAL LETTER I
char ch2 = (char) 304; //LATIN CAPITAL LETTER I WITH DOT ABOVE
System.out.println(ch1==ch2);
System.out.println(Character.toUpperCase(ch1)==Character.toUpperCase(ch2));
System.out.println(Character.toLowerCase(ch1)==Character.toLowerCase(ch2));
```
输出：
```java
false
false
true
```

### (8) regionMatches()
比较两个字符串中指定区域的子串是否相等
```java
//toffset和ooffset分别指出当前字符串中的子串起始位置和要与之比较的字符串中的子串起始地址；len 指出比较长度。
public boolean regionMatches(int toffset, String other, int ooffset,
        int len) {
    char ta[] = value;
    int to = toffset;
    char pa[] = other.value;
    int po = ooffset;
    // Note: toffset, ooffset, or len might be near -1>>>1.
    if ((ooffset < 0) || (toffset < 0)
            || (toffset > (long)value.length - len)
            || (ooffset > (long)other.value.length - len)) {
        return false;
    }
    while (len-- > 0) {
        if (ta[to++] != pa[po++]) {
            return false;
        }
    }
    return true;
}


public boolean regionMatches(boolean ignoreCase, int toffset,
        String other, int ooffset, int len) {
    char ta[] = value;
    int to = toffset;
    char pa[] = other.value;
    int po = ooffset;
    // Note: toffset, ooffset, or len might be near -1>>>1.
    if ((ooffset < 0) || (toffset < 0)
            || (toffset > (long)value.length - len)
            || (ooffset > (long)other.value.length - len)) {
        return false;
    }
    while (len-- > 0) {
        char c1 = ta[to++];
        char c2 = pa[po++];
        if (c1 == c2) {
            continue;
        }
        if (ignoreCase) {
            // If characters don't match but case may be ignored,
            // try converting both characters to uppercase.
            // If the results match, then the comparison scan should
            // continue.
            char u1 = Character.toUpperCase(c1);
            char u2 = Character.toUpperCase(c2);
            if (u1 == u2) {
                continue;
            }
            // Unfortunately, conversion to uppercase does not work properly
            // for the Georgian alphabet, which has strange rules about case
            // conversion.  So we need to make one last check before
            // exiting.
            if (Character.toLowerCase(u1) == Character.toLowerCase(u2)) {
                continue;
            }
        }
        return false;
    }
    return true;
}
```

测试：
```java
String s1= “tsinghua” 
String s2=“it is TsingHua”； 
s1.regionMatches（0，s2，6，7）;
s1.regionMatches（true，0，s2，6，7);
```

### (9) startWith(), endWith()
```java
public boolean startsWith(String prefix) {
    return startsWith(prefix, 0);
}

public boolean endsWith(String suffix) {
    return startsWith(suffix, value.length - suffix.value.length);
}

public boolean startsWith(String prefix, int toffset) {
    char ta[] = value;
    int to = toffset;
    char pa[] = prefix.value;
    int po = 0;
    int pc = prefix.value.length;
    // Note: toffset might be near -1>>>1.
    if ((toffset < 0) || (toffset > value.length - pc)) {
        return false;
    }
    while (--pc >= 0) {
        if (ta[to++] != pa[po++]) {
            return false;
        }
    }
    return true;
}
```

### (10). hashCode()
```java
public int hashCode() {
    int h = hash;//默认值为0
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```
首次调用时会根据`char`字符数组计算哈希值:
hash = s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
以31为权是因为31是一个奇质数，所以31*i=32*i-i=(i5)-i，这种位移与减法结合的计算相比一般的运算快很多。

### (11). indexOf()
```java
public int indexOf(int ch, int fromIndex) {
    final int max = value.length;
    if (fromIndex < 0) {//下标小于0则取为0
        fromIndex = 0;
    } else if (fromIndex >= max) {
        // Note: fromIndex might be near -1>>>1.
        return -1;
    }

    if (ch < Character.MIN_SUPPLEMENTARY_CODE_POINT) {
        final char[] value = this.value;
        for (int i = fromIndex; i < max; i++) {
            if (value[i] == ch) {
                return i;
            }
        }
        return -1;
    } else {
        return indexOfSupplementary(ch, fromIndex);
    }
}

public int indexOf(int ch) {
    return indexOf(ch, 0);
}
```
### (12). subString()
```java
public String substring(int beginIndex) {
    if (beginIndex < 0) {
        throw new StringIndexOutOfBoundsException(beginIndex);
    }
    int subLen = value.length - beginIndex;
    if (subLen < 0) {
        throw new StringIndexOutOfBoundsException(subLen);
    }
    //当参数为0时，返回this; 不为0时调用构造函数创建新的String对象
    return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
}
```
当参数为0时，返回this，不会新建`String`；另由构造函数内调用的`Arrays.copyOfRange()`方法可知截取后的字符串包括下标为beginIndex的字符。

```java
public String substring(int beginIndex, int endIndex) {
    if (beginIndex < 0) {
        throw new StringIndexOutOfBoundsException(beginIndex);
    }
    if (endIndex > value.length) {
        throw new StringIndexOutOfBoundsException(endIndex);
    }
    int subLen = endIndex - beginIndex;
    if (subLen < 0) {
        throw new StringIndexOutOfBoundsException(subLen);
    }
    return ((beginIndex == 0) && (endIndex == value.length)) ? this
            : new String(value, beginIndex, subLen);
}


public CharSequence subSequence(int beginIndex, int endIndex) {
    return this.substring(beginIndex, endIndex);
}
```

### (13). concat()
```java
public String concat(String str) {
    int otherLen = str.length();
    if (otherLen == 0) {
        return this;
    }
    int len = value.length;
    //将value数组复制入新数组内，新数组的长度为value.length和len + otherLen的较小值。
    char buf[] = Arrays.copyOf(value, len + otherLen);
    str.getChars(buf, len);//将str中的char数组复制到buf中，下标从len开始；
    return new String(buf, true);
}
```
分析以上代码可知`concat`方法的作用为在字符串后拼接字符串。

### (14) replace()

该`replace`方法参数是`char`,源码思路是找到第一个出现的oldChar，如果oldChar的下标小于len的下标，把oldChar前面的存到临时变量buf中，把oldChar后面的所有oldChar都变为newChar.
```java
public String replace(char oldChar, char newChar) {
    if (oldChar != newChar) {//替换字符不变不进行操作，返回this，不会创建对象
        int len = value.length;
        int i = -1;
        char[] val = value; /* avoid getfield opcode */

        while (++i < len) {//!!!学习源码编程风格
            if (val[i] == oldChar) {//找到第一个要替换的字符的下标
                break;
            }
        }
        //字符串内无要替换的字符，则返回this，不会创建新对象
        if (i < len) {
            char buf[] = new char[len];
            for (int j = 0; j < i; j++) {
                buf[j] = val[j];//替换字符前的字符复制入新字符数组中
            }
            //遍历要替换字符首次出现的下标到结尾，替换并复制入新char数组
            while (i < len) {
                char c = val[i];
                buf[i] = (c == oldChar) ? newChar : c;
                i++;
            }
            return new String(buf, true);
        }
    }
    return this;
}
```

借用正则表达式的`replace`:
```java
public String replaceFirst(String regex, String replacement) {
    return Pattern.compile(regex).matcher(this).replaceFirst(replacement);
}


public String replaceAll(String regex, String replacement) {
    return Pattern.compile(regex).matcher(this).replaceAll(replacement);
}


public String replace(CharSequence target, CharSequence replacement) {
    return Pattern.compile(target.toString(), Pattern.LITERAL).matcher(
            this).replaceAll(Matcher.quoteReplacement(replacement.toString()));
}
```

其他用到了正则的String方法
```java
public boolean matches(String regex) {
    return Pattern.matches(regex, this);
}
```

### (15) split()
```java
public String[] split(String regex) {
    return split(regex, 0);
}


public String[] split(String regex, int limit) {

    char ch = 0;
    if (((regex.value.length == 1 &&
         ".$|()[{^?*+\\".indexOf(ch = regex.charAt(0)) == -1) ||
         (regex.length() == 2 &&
          regex.charAt(0) == '\\' &&
          (((ch = regex.charAt(1))-'0')|('9'-ch)) < 0 &&
          ((ch-'a')|('z'-ch)) < 0 &&
          ((ch-'A')|('Z'-ch)) < 0)) &&
        (ch < Character.MIN_HIGH_SURROGATE ||
         ch > Character.MAX_LOW_SURROGATE))
    {
        int off = 0;
        int next = 0;
        boolean limited = limit > 0;
        ArrayList<String> list = new ArrayList<>();
        while ((next = indexOf(ch, off)) != -1) {
            if (!limited || list.size() < limit - 1) {
                //截取每一段子字符串存入list
                list.add(substring(off, next));
                //修改截取的起始索引位置
                off = next + 1;
            } else {    // last one
                //assert (list.size() == limit - 1);
                list.add(substring(off, value.length));
                off = value.length;
                break;
            }
        }
        // If no match was found, return this
        // off为零，当前字符串与给定的regex参数不匹配，那么直接返回一个字符串数组  
        // 并且该字符串数组只包含一个元素且该元素为当前字符串对象
        if (off == 0)
            return new String[]{this};

        // Add remaining segment
        if (!limited || list.size() < limit)
            list.add(substring(off, value.length));

        // Construct result
        int resultSize = list.size();
        if (limit == 0) {
            while (resultSize > 0 && list.get(resultSize - 1).length() == 0) {
                resultSize--;
            }
        }
        String[] result = new String[resultSize];
        return list.subList(0, resultSize).toArray(result);
    }
    return Pattern.compile(regex).split(this, limit);
}
```

分段解析：
```java
if (((regex.value.length == 1 &&
     ".$|()[{^?*+\\".indexOf(ch = regex.charAt(0)) == -1) ||
     (regex.length() == 2 &&
      regex.charAt(0) == '\\' &&
      (((ch = regex.charAt(1))-'0')|('9'-ch)) < 0 &&
      ((ch-'a')|('z'-ch)) < 0 &&
      ((ch-'A')|('Z'-ch)) < 0)) &&
    (ch < Character.MIN_HIGH_SURROGATE ||
     ch > Character.MAX_LOW_SURROGATE))
```
- 1. 
```java
((regex.value.length == 1 &&
         ".$|()[{^?*+\\".indexOf(ch = regex.charAt(0)) == -1) ||
         (regex.length() == 2 &&
          regex.charAt(0) == '\\' &&
          (((ch = regex.charAt(1))-'0')|('9'-ch)) < 0 &&
          ((ch-'a')|('z'-ch)) < 0 &&
          ((ch-'A')|('Z'-ch)) < 0))
```
    1. regex只有一位，且不为列出的特殊字符；
    2. regex有两位，第一位为转义字符且第二个字符为非ASCII码字母或非ASCII码数字
    以上两条至少一条为true
    
- 2 
```java
(ch < Character.MIN_HIGH_SURROGATE ||
 ch > Character.MAX_LOW_SURROGATE)
```
regex参数超过2个字符并且为合法的正则表达式 ?????



### （15） trim()
```java
public String trim() {
    int len = value.length;
    int st = 0;
    char[] val = value;    /* avoid getfield opcode */

    while ((st < len) && (val[st] <= ' ')) {
        st++;
    }
    while ((st < len) && (val[len - 1] <= ' ')) {
        len--;
    }
    return ((st > 0) || (len < value.length)) ? substring(st, len) : this;
}
```
这个trim()是去掉首尾的空格，而实现方式也非常简单，分别找到第一个非空格字符的下标，与最后一个非空格字符的下标
然后返回之间的子字符串。注意这里由于应用了substring方法，所以len变量的控制要小心

## 总结

&ensp;&ensp;源码阅读系列博文第一篇，跟了一遍String源码，但是却并不尽兴，对于内存层面的理解却远没有到位。所以还得继续努力。

&ensp;&ensp;既然选择远方，便只顾风雨兼程。不忘初心，方得始终。




