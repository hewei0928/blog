---
title: JDK源码学习系列01----String
date: 2017-11-20 13:02:31
tags: 
   - Java 
   - JDK
   - String
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

## 成员变量
```java
private final char value[];//final修饰， 说明String不可变
private int hash; 
private static final long serialVersionUID = -6849794470754667710L;
private static final ObjectStreamField[] serialPersistentFields = new ObjectStreamField[0];
```

`String`是以字符数组的形式实现的，有`final`关键字修饰，说明字符串一旦初始化就不可修改。这也是与其与`StringBuffer`和`StringBuilder`的区别。`hash`则用来存放计算后的哈希值。

## 构造函数
### 无参构造 
```java
public String() {
    this.value = "".value;
}
```
构造一个长度为0的字符串，由于String是不可变的，没什么大用。但是`String s =""`和`String s2 == new String()`有显著区别：前者对象存储在常量池，后者存储在堆中。因此是`s == s2`返回`false`

### String参数
```java
public String(String original) {
    this.value = original.value;
    this.hash = original.hash;
}
```

### char[]参数
```java
public String(char value[]) {
    this.value = Arrays.copyOf(value, value.length);//复制， 也解释了this.value无法修改，而value可以改变的矛盾
}
```
`Arrays.copyOf`方法中新建了一个`char`数组，并将`value`中的值复制入新建的数组中，返回对新建数组的引用，`this.value`指向这个新建的数组。参数数组改变时字符串不会变化。


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
    this.value = Arrays.copyOfRange(value, offset, offset+count);//利用Arrays.copyOfRange返回新数组对象
}
```
同上，调用`Arrays.copyOfRange`新建了一个`char`数组，进行对`value`进行截取复制。传回引用至`this.value`。

### byte[]
```java
public String(byte bytes[], int offset, int length) {
    checkBounds(bytes, offset, length);
    this.value = StringCoding.decode(bytes, offset, length);
}

public String(byte bytes[]) {
    this(bytes, 0, bytes.length);
}
```

### StringBuilder和StringBuffer参数
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


## 重要方法
### length
```java
public int length() {
    return value.length;
}
```
获取字符串中包含的字符数， 即字符数组的长度
### isEmpty
```java
public boolean isEmpty() {
    return value.length == 0;
}
```
判断字符串是否为空，即判断value数组的长度为0即可 
### charAt
```java
public char charAt(int index) {
    if ((index < 0) || (index >= value.length)) {
        throw new StringIndexOutOfBoundsException(index);
    }
    return value[index];
}
```
返回数组`index`下标处的字符

### codePointAt、codePointBefore、 codePointRange 暂时还未理解
```java
public int codePointAt(int index) {
    if ((index < 0) || (index >= value.length)) {
        throw new StringIndexOutOfBoundsException(index);
    }
    return Character.codePointAtImpl(value, index, value.length);
}


public int codePointBefore(int index) {
    int i = index - 1;
    if ((i < 0) || (i >= value.length)) {
        throw new StringIndexOutOfBoundsException(index);
    }
    return Character.codePointBeforeImpl(value, index, 0);
}


public int codePointCount(int beginIndex, int endIndex) {
    if (beginIndex < 0 || endIndex > value.length || beginIndex > endIndex) {
        throw new IndexOutOfBoundsException();
    }
    return Character.codePointCountImpl(value, beginIndex, endIndex - beginIndex);
}
```



### getChars
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

### getBytes
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

### equals、contentEquals 
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
            : (anotherString != null)// 调用的String对象一定不为null
            && (anotherString.value.length == value.length)
            && regionMatches(true, 0, anotherString, 0, value.length);// 循环比较每个字符toLowerCase,toUpperCase
}
```
`String`类的equals方法首先比较两个字符串是否为同一个对象，为否的话继续遍历比较两个字符串对象内的`char`数组是否完全相同。


```java
public boolean contentEquals(StringBuffer sb) {
    return contentEquals((CharSequence)sb);// 向上转型
}

public boolean contentEquals(CharSequence cs) {
    // Argument is a StringBuffer, StringBuilder
    // StringBuilder StringBuffer比较
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
`contentEquals` 效果与`equals`类似， 不过`equals`参数为`String`而`contentEquals`参数为`CharSequence`及其子类如`CharBuffer`, `Segment`, `String`, `StringBuffer`, `StringBuilder`

### comapreTo
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
比较两个字符串的大小，先顺序遍历两个字符串的`char`数组，如果有不相等的字符，则返回结果。否则则比较两个字符串的长度。

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

**比较两个字符是否大小写不敏感的相等需要`toUpperCase`,`toLowerCase`同时相等。例如下例两个不相等的字符`toUpperCase`，`toLowerCase`的比较结果可能不一致**
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

### regionMatches
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

### startWith、 endWith
```java
public boolean startsWith(String prefix) {
    return startsWith(prefix, 0);
}

public boolean endsWith(String suffix) {
    return startsWith(suffix, value.length - suffix.value.length);
}

// 字符串数组偏移toffset位后是否以prefix开头
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
    // 从偏移处开始循环，循环prefix.value.length次
    while (--pc >= 0) {
        if (ta[to++] != pa[po++]) {
            return false;
        }
    }
    return true;
}
```

### hashCode
```java
public int hashCode() {
    int h = hash;//默认值为0
    //同一对象重复调用hashCode方法，这段代码只会运行一次
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

### indexOf
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

//返回字符串首次出现的下标，无符合的则返回-1
public int indexOf(String str) {
    return indexOf(str, 0);
}

public int indexOf(String str, int fromIndex) {
    return String.indexOf(value, 0, count, str, fromIndex);
}
```
### subString
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
    //当需要截取整个字符串时返回this,不会创建新对象
    return ((beginIndex == 0) && (endIndex == value.length)) ? this
            : new String(value, beginIndex, subLen);
}

//返回结果可以直接向下转型为String
public CharSequence subSequence(int beginIndex, int endIndex) {
    return this.substring(beginIndex, endIndex);
}
```

### concat
```java
public String concat(String str) {
    int otherLen = str.length();
    if (otherLen == 0) {
        return this;
    }
    int len = value.length;
    //将value数组复制入新数组内，新数组的长度len + otherLen
    char buf[] = Arrays.copyOf(value, len + otherLen);
    //将str中的char数组复制到buf中，下标从len开始；
    str.getChars(buf, len);
    return new String(buf, true);
}

void getChars(char dst[], int dstBegin) {
    System.arraycopy(value, 0, dst, dstBegin, value.length);
}
```
分析以上代码可知`concat`方法的作用为在字符串后拼接字符串，返回的字符串及字符串数组为新对象

### replace

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
                char c = val[i];// 将原char赋值入新数组
                buf[i] = (c == oldChar) ? newChar : c;//替换目标字符
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

### split
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

1. 
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

2. 
```java
(ch < Character.MIN_HIGH_SURROGATE ||
 ch > Character.MAX_LOW_SURROGATE)
```
regex参数超过2个字符并且为合法的正则表达式 ?????


### contains
```java
public boolean contains(CharSequence s) {
    return indexOf(s.toString()) > -1;
}

public int indexOf(String str) {
    return indexOf(str, 0);
}

public int indexOf(String str, int fromIndex) {
    return indexOf(value, 0, value.length,
            str.value, 0, str.value.length, fromIndex);
}

// 在source数组指定范围内查找target数组的指定范围
//source源数组，sourceOffset数组查找偏移下标，sourceCount源数组长度
//target要查找数组，targetOffset查找数组偏移下标，  targetCount查找数组长度
static int indexOf(char[] source, int sourceOffset, int sourceCount,
        char[] target, int targetOffset, int targetCount,
        int fromIndex) {
    if (fromIndex >= sourceCount) {
        return (targetCount == 0 ? sourceCount : -1);
    }
    if (fromIndex < 0) {
        fromIndex = 0;
    }
    if (targetCount == 0) {
        return fromIndex;
    }

    char first = target[targetOffset];
    int max = sourceOffset + (sourceCount - targetCount);

    for (int i = sourceOffset + fromIndex; i <= max; i++) {
        /* Look for first character. */
        if (source[i] != first) {
            while (++i <= max && source[i] != first);
        }

        /* Found first character, now look at the rest of v2 */
        if (i <= max) {
            int j = i + 1;
            int end = j + targetCount - 1;
            for (int k = targetOffset + 1; j < end && source[j]
                    == target[k]; j++, k++);

            if (j == end) {
                /* Found whole string. */
                return i - sourceOffset;
            }
        }
    }
    return -1;
}
```
判断字符串内是否存在某字符串片段

### trim
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
然后返回之间的子字符串。首位均未出现空格时返回原字符串对象

## 不可变String
&ensp;&ensp;`String`对象是不可变的。根据`String`源码，类中每一个看似会修改`String`值的方法，实际上都是创建了一个新的`String`对象，而最初的`String`对象丝毫未动：
```java
public static void main(String[] args) {
    String a = new String("aaaaa");
    String b = a.toUpperCase();
    System.out.println(a);
}
```


## 重载“+” 与 StringBuilder

### 重载“+”
&ensp;&ensp;“+”在用于`String`类时，会被赋予特殊的含义。
```java
public static void main(String[] args) {
    String a = "aaa";
    String b = "bbb";
    String c = a + b;
}
```
利用**javap -c**命令反编译上述代码的class文件，得到以下结果：
![微信截图_20180523103952.png-56.2kB][1]

可以看到生成变量c时，首先生成了一个`StringBuilder`类型变量，然后调用了两次`append`方法，在用`toString`方法返回`String`对象。只有当拼接语句中出现存储在堆中的对象时编译器才会对“+”进行重载，如果拼接语句中出现的对象全部存储在常量池，则编译器会直接对其拼接，并将结果存入常量池：
```java
public static void main(String[] args) {
    String a = "aaa";
    Integer b = 1;
    String c = "a" + "b";
    String d = "a" + a;
    String e = "a" + 1;
    String f = "a" + b;
}
```

反编译后结果如下：

![微信截图_20180523110522.png-78.6kB][2]

`String c = "a" + "b"`和`String e = "a" + 1`分别对应`String ab`和`String a1` 字符串直接在常量池中相加，所得的字符串也存储在常量池中。因此如果运行
```java
String g = "a" + "b";
String h = "a" + 1;
System.out.println(c == g);
System.out.println(h == e);
```
返回的结果都是`true`。g,c,e,h都指向存储在常量池中。

`String d = "a" + a`和`String f = "a" + b`则对“+”进行了重载。生成新`String`变量存储在堆中。因此如果运行：
```java
String i = "a" + a;
String j = "a" + b;
System.out.println(d == i);//false
System.out.println(f == j);//false
```

### StringBuilder的使用
这个现象让我明白以前对的字符串拼接操作的缺陷。例如
```java
public String a(String[] strings){
    String result = "";
    for (String s : strings) {
        result += s;
    }
    return result;
}

public String b(String[] strings){
    StringBuilder result = new StringBuilder();
    for (String s : strings) {
        result.append(s);
    }
    return result.toString();
}
```
对这两行代码进行反编译：
![image_1ce5n3fnuifh1tg61dap11fd12tps.png-76.4kB][3]
![image_1ce5n3vvvc8v2likg0fnh20t19.png-66.8kB][4]
可以发现a方法的循环体中，每循环一次都会创建一个`StringBuilder`对象。这对于内存是一种浪费。因此当使用循环去拼接字符串时，最好先创建`StringBuilder`对象，然后使用`append`去实现字符串拼接。

### toString方法中的递归

```java
public class InfiniteResoursion {

    @Override
    public String toString() {
        return "内存地址为" + this;
    }

    public static void main(String[] args) {
        List<InfiniteResoursion> infiniteResoursions = new ArrayList<>();

        for(int i = 0; i < 10; i++){
            infiniteResoursions.add(new InfiniteResoursion());
        }

        System.out.println(infiniteResoursions);
    }
}
```
`InfiniteResoursion`的`toString`方法看似是打印出对象的内存地址，但是当实际运行时，却会报`java.lang.StackOverflowError`。这是因为当运行`"内存地址为" + this`时，编译器会自动调用`this.toString`将`this`转换为字符串，于是就发生了递归。如果真的想打印内存地址，则要调用`Object.toString`：
```
@Override
public String toString() {
    return "内存地址为" + super.toString();
}
```

  [1]: http://static.zybuluo.com/hewei0928/pq31gchv5my2qh6wzdrcxck5/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180523103952.png
  [2]: http://static.zybuluo.com/hewei0928/h3o8fx6mv8n3ub3g417vcnjw/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20180523110522.png
  [3]: http://static.zybuluo.com/hewei0928/8g79bcgzds9lj8olwswk9akp/image_1ce5n3fnuifh1tg61dap11fd12tps.png
  [4]: http://static.zybuluo.com/hewei0928/95js4b98a1ggesb413hkbzxu/image_1ce5n3vvvc8v2likg0fnh20t19.png

## 总结

&ensp;&ensp;源码阅读系列博文第一篇，跟了一遍String源码，但是却并不尽兴，对于内存层面的理解远没有到位。所以还得继续努力。

&ensp;&ensp;既然选择远方，便只顾风雨兼程。不忘初心，方得始终。




