---
title: JDK源码学习系列08----Arrays
date: 2017-12-18 09:27:02
tags:
    - JDK
    - Java
    - Arrays
---

>Arrays类是JDK中专门用于进行数组操作的类。自己平时用到倒不是很多，在阅读集合类时发现大量用到了此类中的方法。因此决定开始详细的解析其源码，熟悉各个方法。

## 方法汇总

`Arrays`类中的方法大致可以分为以下几类：

| 方法名 | 用处 |
| ----- | :-----: |
| copyOf() | 复制指定的数组 |
| copyOfRange() | 将指定数组的某一范围内的数据复制到另一数组内 |
| binarySearch() | 使用二分搜索法查找数组内指定元素 |
| deeepEquals() | 比较两个数组是否深层次相等 |
| deepHashCode() | 基于指定数组的"深层内容"返回哈希值 |
| deepToString() | 返回指定数组"深层内容"的字符串表示形式 |
| equals() | 比较两个数组是否相等 |
| fill() | 将指定对象引用分配给数组的每个元素 |
| hashCode() | 基于数组的内容返回哈希值 |
| sort() | 按指定顺序对数组元素进行排序 |
| toString() | 返回指定数组内容的字符串表示形式 |
| asList() | 返回一个受指定数组支持的固定大小的列表 |

<!--more-->


## copyOf() 复制指定的数组，返回新数组

### System.arrayCopy()
```java
//
// 数组复制， 浅拷贝
public static native void arraycopy(Object src, //源数组
                                    int  srcPos, // 起始位置[0, length)
                                    Object dest, // 目标数组
                                    int destPos, // 起始位置[0, length)
                                    int length); // 复制长度 length <= src.length - srcPos, length <= dest.length - destPos
                                    

public static void main(String[] args) {
        int[] a = {1, 2};
        int[] b = {4, 5, 6, 7, 8, 6};
        System.arraycopy(a, 0, b, 3, 3);// ArrayIndexOutOfBoundsException
        System.out.println(Arrays.toString(b));
    }                                    
```


### copyOf
```
// 复制对象数组至新数组， newLength > original.length 则将多余未用null填充
public static <T> T[] copyOf(T[] original, int newLength) {
    return (T[]) copyOf(original, newLength, original.getClass());
}


public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked")
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}


// 以下为基本数据类型的数组拷贝， newLength > original.length 则将多余位填充默认值
public static byte[] copyOf(byte[] original, int newLength) {
    byte[] copy = new byte[newLength];
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}

public static short[] copyOf(short[] original, int newLength) {
    short[] copy = new short[newLength];
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}

public static int[] copyOf(int[] original, int newLength) {
    int[] copy = new int[newLength];
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}

public static long[] copyOf(long[] original, int newLength) {
    long[] copy = new long[newLength];
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}

public static char[] copyOf(char[] original, int newLength) {
    char[] copy = new char[newLength];
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}

public static float[] copyOf(float[] original, int newLength) {
    float[] copy = new float[newLength];
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}

public static double[] copyOf(double[] original, int newLength) {
    double[] copy = new double[newLength];
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}

public static boolean[] copyOf(boolean[] original, int newLength) {
    boolean[] copy = new boolean[newLength];
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}
```

## copyOfRange 复制数组的指定范围至新数组，返回新数组

```java
// 对象数组复制，长度超出部分用null填充
public static <T> T[] copyOfRange(T[] original, int from, int to) {
    return copyOfRange(original, from, to, (Class<? extends T[]>) original.getClass());
}

public static <T,U> T[] copyOfRange(U[] original, int from, int to, Class<? extends T[]> newType) {
    int newLength = to - from;
    if (newLength < 0)
        throw new IllegalArgumentException(from + " > " + to);
    @SuppressWarnings("unchecked")
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, from, copy, 0,
                     Math.min(original.length - from, newLength));
    return copy;
}


// 基本类型数组范围复制 复制下标from（从0开始） 至 下标to处（不包括）的数组。 to > original.length 超出部分用默认值填充。 新数组长为to - from
public static byte[] copyOfRange(byte[] original, int from, int to) {
    int newLength = to - from;
    if (newLength < 0)
        throw new IllegalArgumentException(from + " > " + to);
    byte[] copy = new byte[newLength];
    System.arraycopy(original, from, copy, 0,
                     Math.min(original.length - from, newLength));
    return copy;
}

public static short[] copyOfRange(short[] original, int from, int to) {
    int newLength = to - from;
    if (newLength < 0)
        throw new IllegalArgumentException(from + " > " + to);
    short[] copy = new short[newLength];
    System.arraycopy(original, from, copy, 0,
                     Math.min(original.length - from, newLength));
    return copy;
}

public static int[] copyOfRange(int[] original, int from, int to) {
    int newLength = to - from;
    if (newLength < 0)
        throw new IllegalArgumentException(from + " > " + to);
    int[] copy = new int[newLength];
    System.arraycopy(original, from, copy, 0,
                     Math.min(original.length - from, newLength));
    return copy;
}

public static long[] copyOfRange(long[] original, int from, int to) {
    int newLength = to - from;
    if (newLength < 0)
        throw new IllegalArgumentException(from + " > " + to);
    long[] copy = new long[newLength];
    System.arraycopy(original, from, copy, 0,
                     Math.min(original.length - from, newLength));
    return copy;
}

public static char[] copyOfRange(char[] original, int from, int to) {
    int newLength = to - from;
    if (newLength < 0)
        throw new IllegalArgumentException(from + " > " + to);
    char[] copy = new char[newLength];
    System.arraycopy(original, from, copy, 0,
                     Math.min(original.length - from, newLength));
    return copy;
}

public static float[] copyOfRange(float[] original, int from, int to) {
    int newLength = to - from;
    if (newLength < 0)
        throw new IllegalArgumentException(from + " > " + to);
    float[] copy = new float[newLength];
    System.arraycopy(original, from, copy, 0,
                     Math.min(original.length - from, newLength));
    return copy;
}

public static double[] copyOfRange(double[] original, int from, int to) {
    int newLength = to - from;
    if (newLength < 0)
        throw new IllegalArgumentException(from + " > " + to);
    double[] copy = new double[newLength];
    System.arraycopy(original, from, copy, 0,
                     Math.min(original.length - from, newLength));
    return copy;
}

public static boolean[] copyOfRange(boolean[] original, int from, int to) {
    int newLength = to - from;
    if (newLength < 0)
        throw new IllegalArgumentException(from + " > " + to);
    boolean[] copy = new boolean[newLength];
    System.arraycopy(original, from, copy, 0,
                     Math.min(original.length - from, newLength));
    return copy;
}
```
实例：

```java
public static void main(String[] args) {
    int[] b = {4, 5, 6, 7, 8, 6};
    int[] d = Arrays.copyOfRange(b, 4, 8);
    System.out.println(Arrays.toString(d)); // [8, 6, 0, 0]
    
    Integer[] Arr = new Integer[]{5, 2, 15, 52, 10};
    Integer[] arr1 = Arrays.copyOfRange(Arr, 2, 10);
    System.out.println(Arrays.toString(arr1)); // [15, 52, 10, null, null, null, null, null]
}
```

## binarySearch
```java
//a - 要搜索的数组, key - 要搜索的值
// 如果它包含在数组中，则返回搜索键的索引；否则返回 (-(插入点) - 1)。
public static int binarySearch(long[] a, long key) {
    return binarySearch0(a, 0, a.length, key);
}

//a - 要搜索的数组, key - 要搜索的值, 数组内的搜索下标[fromIndex, toIndex)
publ// 如果它包含在数组中，则返回搜索键的索引；否则返回 (-(插入点) - 1)。ic static int binarySearch(long[] a, int fromIndex, int toIndex,
                               long key) {
    rangeCheck(a.length, fromIndex, toIndex);
    return binarySearch0(a, fromIndex, toIndex, key);
}

// Like public version, but without range checks.
private static int binarySearch0(long[] a, int fromIndex, int toIndex,
                                 long key) {
    int low = fromIndex;
    int high = toIndex - 1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        long midVal = a[mid];

        if (midVal < key)
            low = mid + 1;
        else if (midVal > key)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found.
}


public static int binarySearch(int[] a, int key) {
    return binarySearch0(a, 0, a.length, key);
}

public static int binarySearch(int[] a, int fromIndex, int toIndex,
                               int key) {
    rangeCheck(a.length, fromIndex, toIndex);
    return binarySearch0(a, fromIndex, toIndex, key);
}

// Like public version, but without range checks.
private static int binarySearch0(int[] a, int fromIndex, int toIndex,
                                 int key) {
    int low = fromIndex;
    int high = toIndex - 1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        int midVal = a[mid];

        if (midVal < key)
            low = mid + 1;
        else if (midVal > key)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found.
}

public static int binarySearch(short[] a, short key) {
    return binarySearch0(a, 0, a.length, key);
}

public static int binarySearch(short[] a, int fromIndex, int toIndex,
                               short key) {
    rangeCheck(a.length, fromIndex, toIndex);
    return binarySearch0(a, fromIndex, toIndex, key);
}

// Like public version, but without range checks.
private static int binarySearch0(short[] a, int fromIndex, int toIndex,
                                 short key) {
    int low = fromIndex;
    int high = toIndex - 1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        short midVal = a[mid];

        if (midVal < key)
            low = mid + 1;
        else if (midVal > key)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found.
}

public static int binarySearch(char[] a, char key) {
    return binarySearch0(a, 0, a.length, key);
}

public static int binarySearch(char[] a, int fromIndex, int toIndex,
                               char key) {
    rangeCheck(a.length, fromIndex, toIndex);
    return binarySearch0(a, fromIndex, toIndex, key);
}

// Like public version, but without range checks.
private static int binarySearch0(char[] a, int fromIndex, int toIndex,
                                 char key) {
    int low = fromIndex;
    int high = toIndex - 1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        char midVal = a[mid];

        if (midVal < key)
            low = mid + 1;
        else if (midVal > key)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found.
}

public static int binarySearch(byte[] a, byte key) {
    return binarySearch0(a, 0, a.length, key);
}

public static int binarySearch(byte[] a, int fromIndex, int toIndex,
                               byte key) {
    rangeCheck(a.length, fromIndex, toIndex);
    return binarySearch0(a, fromIndex, toIndex, key);
}

// Like public version, but without range checks.
private static int binarySearch0(byte[] a, int fromIndex, int toIndex,
                                 byte key) {
    int low = fromIndex;
    int high = toIndex - 1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        byte midVal = a[mid];

        if (midVal < key)
            low = mid + 1;
        else if (midVal > key)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found.
}

public static int binarySearch(double[] a, double key) {
    return binarySearch0(a, 0, a.length, key);
}

public static int binarySearch(double[] a, int fromIndex, int toIndex,
                               double key) {
    rangeCheck(a.length, fromIndex, toIndex);
    return binarySearch0(a, fromIndex, toIndex, key);
}

// Like public version, but without range checks.
private static int binarySearch0(double[] a, int fromIndex, int toIndex,
                                 double key) {
    int low = fromIndex;
    int high = toIndex - 1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        double midVal = a[mid];

        if (midVal < key)
            low = mid + 1;  // Neither val is NaN, thisVal is smaller
        else if (midVal > key)
            high = mid - 1; // Neither val is NaN, thisVal is larger
        else {
            long midBits = Double.doubleToLongBits(midVal);
            long keyBits = Double.doubleToLongBits(key);
            if (midBits == keyBits)     // Values are equal
                return mid;             // Key found
            else if (midBits < keyBits) // (-0.0, 0.0) or (!NaN, NaN)
                low = mid + 1;
            else                        // (0.0, -0.0) or (NaN, !NaN)
                high = mid - 1;
        }
    }
    return -(low + 1);  // key not found.
}

public static int binarySearch(float[] a, float key) {
    return binarySearch0(a, 0, a.length, key);
}

public static int binarySearch(float[] a, int fromIndex, int toIndex,
                               float key) {
    rangeCheck(a.length, fromIndex, toIndex);
    return binarySearch0(a, fromIndex, toIndex, key);
}

// Like public version, but without range checks.
private static int binarySearch0(float[] a, int fromIndex, int toIndex,
                                 float key) {
    int low = fromIndex;
    int high = toIndex - 1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        float midVal = a[mid];

        if (midVal < key)
            low = mid + 1;  // Neither val is NaN, thisVal is smaller
        else if (midVal > key)
            high = mid - 1; // Neither val is NaN, thisVal is larger
        else {
            int midBits = Float.floatToIntBits(midVal);
            int keyBits = Float.floatToIntBits(key);
            if (midBits == keyBits)     // Values are equal
                return mid;             // Key found
            else if (midBits < keyBits) // (-0.0, 0.0) or (!NaN, NaN)
                low = mid + 1;
            else                        // (0.0, -0.0) or (NaN, !NaN)
                high = mid - 1;
        }
    }
    return -(low + 1);  // key not found.
}

public static int binarySearch(Object[] a, Object key) {
    return binarySearch0(a, 0, a.length, key);
}

public static int binarySearch(Object[] a, int fromIndex, int toIndex,
                               Object key) {
    rangeCheck(a.length, fromIndex, toIndex);
    return binarySearch0(a, fromIndex, toIndex, key);
}

// Like public version, but without range checks.
private static int binarySearch0(Object[] a, int fromIndex, int toIndex,
                                 Object key) {
    int low = fromIndex;
    int high = toIndex - 1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        @SuppressWarnings("rawtypes")
        Comparable midVal = (Comparable)a[mid];
        @SuppressWarnings("unchecked")
        int cmp = midVal.compareTo(key);

        if (cmp < 0)
            low = mid + 1;
        else if (cmp > 0)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found.
}

public static <T> int binarySearch(T[] a, T key, Comparator<? super T> c) {
    return binarySearch0(a, 0, a.length, key, c);
}

public static <T> int binarySearch(T[] a, int fromIndex, int toIndex,
                                   T key, Comparator<? super T> c) {
    rangeCheck(a.length, fromIndex, toIndex);
    return binarySearch0(a, fromIndex, toIndex, key, c);
}

// Like public version, but without range checks.
private static <T> int binarySearch0(T[] a, int fromIndex, int toIndex,
                                     T key, Comparator<? super T> c) {
    if (c == null) {
        return binarySearch0(a, fromIndex, toIndex, key);
    }
    int low = fromIndex;
    int high = toIndex - 1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        T midVal = a[mid];
        int cmp = c.compare(midVal, key);
        if (cmp < 0)
            low = mid + 1;
        else if (cmp > 0)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found.
}
```

实例：
```java
public static void main(String[] args) {
    int[] b = {4, 5, 6, 7, 8, 6};
    Arrays.sort(b);
    System.out.println(Arrays.binarySearch(b,6));// 2
    System.out.println(Arrays.binarySearch(b,1, 2, 6));// -3
    System.out.println(Arrays.binarySearch(b,3, 4, 6));// 3
    System.out.println(Arrays.toString(b));// [4, 5, 6, 6, 7, 8]
}
```

## deeepEquals()  比较两个数组是否深层次相等
当数组为一维时与`equals`方法相同， 而当数组为多维数组时需要使用`deeepEquals`
```java
public static boolean deepEquals(Object[] a1, Object[] a2) {
    if (a1 == a2)
        return true;
    if (a1 == null || a2==null)
        return false;
    int length = a1.length;
    if (a2.length != length)
        return false;

    for (int i = 0; i < length; i++) {
        Object e1 = a1[i];
        Object e2 = a2[i];

        if (e1 == e2)
            continue;
        if (e1 == null)
            return false;

        // Figure out whether the two elements are equal
        boolean eq = deepEquals0(e1, e2);

        if (!eq)
            return false;
    }
    return true;
}

static boolean deepEquals0(Object e1, Object e2) {
    assert e1 != null;
    boolean eq;
    if (e1 instanceof Object[] && e2 instanceof Object[])// int[][] a instanceof Object[] --> true
        eq = deepEquals ((Object[]) e1, (Object[]) e2);
    else if (e1 instanceof byte[] && e2 instanceof byte[])
        eq = equals((byte[]) e1, (byte[]) e2);
    else if (e1 instanceof short[] && e2 instanceof short[])
        eq = equals((short[]) e1, (short[]) e2);
    else if (e1 instanceof int[] && e2 instanceof int[])
        eq = equals((int[]) e1, (int[]) e2);
    else if (e1 instanceof long[] && e2 instanceof long[])
        eq = equals((long[]) e1, (long[]) e2);
    else if (e1 instanceof char[] && e2 instanceof char[])
        eq = equals((char[]) e1, (char[]) e2);
    else if (e1 instanceof float[] && e2 instanceof float[])
        eq = equals((float[]) e1, (float[]) e2);
    else if (e1 instanceof double[] && e2 instanceof double[])
        eq = equals((double[]) e1, (double[]) e2);
    else if (e1 instanceof boolean[] && e2 instanceof boolean[])
        eq = equals((boolean[]) e1, (boolean[]) e2);
    else
        eq = e1.equals(e2);
    return eq;
}
```

## equals()  比较两个数组是否相等
先直接`==`判断，再判断是否一个数组为空，再比较数组长度，最后循环比较各个下标处的数据是否`equals`
```java
public static boolean equals(long[] a, long[] a2) {
    if (a==a2)
        return true;
    if (a==null || a2==null)
        return false;

    int length = a.length;
    if (a2.length != length)
        return false;

    for (int i=0; i<length; i++)
        if (a[i] != a2[i])
            return false;

    return true;
}

public static boolean equals(int[] a, int[] a2) {
    if (a==a2)
        return true;
    if (a==null || a2==null)
        return false;

    int length = a.length;
    if (a2.length != length)
        return false;

    for (int i=0; i<length; i++)
        if (a[i] != a2[i])
            return false;

    return true;
}

public static boolean equals(short[] a, short a2[]) {
    if (a==a2)
        return true;
    if (a==null || a2==null)
        return false;

    int length = a.length;
    if (a2.length != length)
        return false;

    for (int i=0; i<length; i++)
        if (a[i] != a2[i])
            return false;

    return true;
}

public static boolean equals(char[] a, char[] a2) {
    if (a==a2)
        return true;
    if (a==null || a2==null)
        return false;

    int length = a.length;
    if (a2.length != length)
        return false;

    for (int i=0; i<length; i++)
        if (a[i] != a2[i])
            return false;

    return true;
}

public static boolean equals(byte[] a, byte[] a2) {
    if (a==a2)
        return true;
    if (a==null || a2==null)
        return false;

    int length = a.length;
    if (a2.length != length)
        return false;

    for (int i=0; i<length; i++)
        if (a[i] != a2[i])
            return false;

    return true;
}

public static boolean equals(boolean[] a, boolean[] a2) {
    if (a==a2)
        return true;
    if (a==null || a2==null)
        return false;

    int length = a.length;
    if (a2.length != length)
        return false;

    for (int i=0; i<length; i++)
        if (a[i] != a2[i])
            return false;

    return true;
}

public static boolean equals(double[] a, double[] a2) {
    if (a==a2)
        return true;
    if (a==null || a2==null)
        return false;

    int length = a.length;
    if (a2.length != length)
        return false;

    for (int i=0; i<length; i++)
        if (Double.doubleToLongBits(a[i])!=Double.doubleToLongBits(a2[i]))
            return false;

    return true;
}

public static boolean equals(float[] a, float[] a2) {
    if (a==a2)
        return true;
    if (a==null || a2==null)
        return false;

    int length = a.length;
    if (a2.length != length)
        return false;

    for (int i=0; i<length; i++)
        if (Float.floatToIntBits(a[i])!=Float.floatToIntBits(a2[i]))
            return false;

    return true;
}

public static boolean equals(Object[] a, Object[] a2) {
    if (a==a2)
        return true;
    if (a==null || a2==null)
        return false;

    int length = a.length;
    if (a2.length != length)
        return false;

    for (int i=0; i<length; i++) {
        Object o1 = a[i];
        Object o2 = a2[i];
        if (!(o1==null ? o2==null : o1.equals(o2)))
            return false;
    }
    
    return true;
}
```
## fill()  将指定对象引用分配给数组的每个元素
```java
// 将数组用特定值填满
public static void fill(long[] a, long val) {
    for (int i = 0, len = a.length; i < len; i++)
        a[i] = val;
}

// 将数组下标[fromIndex, toIndex)的值用特定值填满
public static void fill(long[] a, int fromIndex, int toIndex, long val) {
    rangeCheck(a.length, fromIndex, toIndex);
    for (int i = fromIndex; i < toIndex; i++)
        a[i] = val;
}

public static void fill(int[] a, int val) {
    for (int i = 0, len = a.length; i < len; i++)
        a[i] = val;
}

public static void fill(int[] a, int fromIndex, int toIndex, int val) {
    rangeCheck(a.length, fromIndex, toIndex);
    for (int i = fromIndex; i < toIndex; i++)
        a[i] = val;
}

public static void fill(short[] a, short val) {
    for (int i = 0, len = a.length; i < len; i++)
        a[i] = val;
}

public static void fill(short[] a, int fromIndex, int toIndex, short val) {
    rangeCheck(a.length, fromIndex, toIndex);
    for (int i = fromIndex; i < toIndex; i++)
        a[i] = val;
}

public static void fill(char[] a, char val) {
    for (int i = 0, len = a.length; i < len; i++)
        a[i] = val;
}

public static void fill(char[] a, int fromIndex, int toIndex, char val) {
    rangeCheck(a.length, fromIndex, toIndex);
    for (int i = fromIndex; i < toIndex; i++)
        a[i] = val;
}

public static void fill(byte[] a, byte val) {
    for (int i = 0, len = a.length; i < len; i++)
        a[i] = val;
}

public static void fill(byte[] a, int fromIndex, int toIndex, byte val) {
    rangeCheck(a.length, fromIndex, toIndex);
    for (int i = fromIndex; i < toIndex; i++)
        a[i] = val;
}

public static void fill(boolean[] a, boolean val) {
    for (int i = 0, len = a.length; i < len; i++)
        a[i] = val;
}

public static void fill(boolean[] a, int fromIndex, int toIndex,
                        boolean val) {
    rangeCheck(a.length, fromIndex, toIndex);
    for (int i = fromIndex; i < toIndex; i++)
        a[i] = val;
}

public static void fill(double[] a, double val) {
    for (int i = 0, len = a.length; i < len; i++)
        a[i] = val;
}

public static void fill(double[] a, int fromIndex, int toIndex,double val){
    rangeCheck(a.length, fromIndex, toIndex);
    for (int i = fromIndex; i < toIndex; i++)
        a[i] = val;
}

public static void fill(float[] a, float val) {
    for (int i = 0, len = a.length; i < len; i++)
        a[i] = val;
}

public static void fill(float[] a, int fromIndex, int toIndex, float val) {
    rangeCheck(a.length, fromIndex, toIndex);
    for (int i = fromIndex; i < toIndex; i++)
        a[i] = val;
}

public static void fill(Object[] a, Object val) {
    for (int i = 0, len = a.length; i < len; i++)
        a[i] = val;
}

public static void fill(Object[] a, int fromIndex, int toIndex, Object val) {
    rangeCheck(a.length, fromIndex, toIndex);
    for (int i = fromIndex; i < toIndex; i++)
        a[i] = val;
}
```

## sort()  按指定顺序对数组元素进行排序
## toString()  返回指定数组内容的字符串表示形式

## asList()  返回一个受指定数组支持的固定大小的列表
```java
public static void main(String[] args) {
    Integer[] a = {1, 2};
    int[] b = {1};
    List d = Arrays.sList(a, b);
    System.out.println(Arrays.toString((Integer[])d.get(0)));//[1, 2]
}
```




