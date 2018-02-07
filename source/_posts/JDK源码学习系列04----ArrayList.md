---
title: JDK源码学习系列04----ArrayList
tags: 
    - Java 
    - JDK源码 
    - List 
    - ArrayList 
    - Collection

---

>集合一直是java中的重点，今天开始慢慢学习它的源码。

## 一. ArrayList
```java
public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```
`ArrayList`实现了`List`, `RandomAccess`, `Clonable`, `SeriaLizable`, 继承了`AbstractList`。
`ArrayList`的类结构图如下：
<!--more-->
![ArrayList.jpg-57.3kB][1]


`ArrayList`集合属性最终实现的`Iterable`接口
`List`接口定义了列表必须实现的方法，`RandomAccess`是一个标记接口，接口内没有任何内容。（日后去理解）
`Cloneable`接口标识该类可以调用`Object.clone`方法返回对象的浅拷贝（日后详细讨论）。

## 二. 成员变量
```java
private static final long serialVersionUID = 8683452581122892189L;

private static final int DEFAULT_CAPACITY = 10;//默认容量为10

private static final Object[] EMPTY_ELEMENTDATA = {};//空ArrayList

private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;// 数组最大容量

transient Object[] elementData;//实际数据存储。 transient表示反序列化

private int size;//数组中实际存储的数据数, 不是容量。

protected transient int modCount = 0;//继承自AbstractList记录列表结构性变化的次数
```
从成员变量可以看出，`ArrayList`底层基于数组实现，也就是常说的动态数组， 因此`ArrayList`内的元素可以重复。

## 三. 构造函数

```java
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

//构造默认ArrayList,此为jdk1.8中的写法, 后续在初次调用add方法时会对数组进行扩容处理
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

//而在在JDK1.6中，其构造函数为
public ArrayList() {    
    this(10); 
}

//把整个集合初始化给ArrayList.注意：该集合内的数据类型必须与ArrayList一致!!!!  
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        //c.toArray()可能不会正确地返回一个 Object[]数组，那么使用Arrays.copyOf()方法
        if (elementData.getClass() != Object[].class)
        //Arrays.copyOf()返回一个 Object[].class类型的，大小为size，元素为elementData[0,...,size-1]
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```
jdk1.8中默认构造函数初始化时创建空数组，在第一次调用add方法时再对数组进行扩容。

ArrayList(Collection<? extends E> c)中toArray()方法bug测试如下：
```java
public class MyList<E> extends ArrayList<E> {
    // toArray() 的同名方法 返回String[]也可进行重写
    @Override
    public String[] toArray() {
        return new String[]{"1", "2", "3"};
    }
}


public void test3() {
    List<String> ss = new LinkedList<String>();             // LinkedList toArray() 返回的本身就是 Object[]
    ss.add("123");
    Object[] objs = ss.toArray();
    objs[0] = new Object();

    // 此处说明了：c.toArray might (incorrectly) not return Object[] (see 6260652)
    ss = new MyList<String>();
    objs = ss.toArray();
    System.out.println(objs.getClass());        // class [Ljava.lang.String;
    objs[0] = new Object();                         // java.lang.ArrayStoreException: java.lang.Object
}
```
当我们调用 List的 toArray() 时，我们实际上调用的是其具体实现类 MyList 中重写的 String[] toArray() 方法。因此，返回数组并不是 Object[] 而是 String[]，而向下转型是不安全的，因此会抛出异常。

## 四. 重要成员方法

### (1) trimToSize()
```java
//方法修整此ArrayList实例的是列表的当前大小的容量。应用程序可以使用此操作，以尽量减少一个ArrayList实例的存储。
public void trimToSize() {
    modCount++;//记录list结构性变化次数，当用iterator进行遍历时，会检查modCount,一旦发现遍历时值发生了变化便会报错
    if (size < elementData.length) {//实际数据数小于数组长度，即有多余无用空间。
        elementData = (size == 0)
          ? EMPTY_ELEMENTDATA
          : Arrays.copyOf(elementData, size);
    }
}
```
当list调用add方法时会进行扩容处理，此方法会将内容为空的数组项去除。节省空间。


### (2) 容量扩充

```java
//这个扩容方法，内部没有调用，判断ArrayList是否为空 
public void ensureCapacity(int minCapacity) {
    int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
        // any size if not default element table
        ? 0
        // larger than default for default empty table. It's already
        // supposed to be at default size.
        : DEFAULT_CAPACITY;

    if (minCapacity > minExpand) {
        ensureExplicitCapacity(minCapacity);
    }
}


//内部扩容方法
private void ensureCapacityInternal(int minCapacity) {
    //如果列表内数组为空，即为首次add时的扩容，将数组长度设为10
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    
    //判断是否需要扩容
    ensureExplicitCapacity(minCapacity);
}



private void ensureExplicitCapacity(int minCapacity) {
    ////modCount这个参数运用到了 fail-fast 机制
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}



private void grow(int minCapacity) {
    // 计算新数组长度
    int oldCapacity = elementData.length;
    //new = old * 1.5 默认数组扩容量
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    //将原数组数据复制入新数组中
    elementData = Arrays.copyOf(elementData, newCapacity);
}


//将数组容量扩围原先的1.5倍
private static int hugeCapacity(int minCapacity) {
    //超出最大容量 x7fffffff 抛出错误
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

列表内数组元素个数获取：
```java
public int size() {
    return size;//数组实际元素个数，而不是容量(数组长度)
}

public boolean isEmpty() {
    return size == 0;
}
```

### (3) contains(), index()
```java
public boolean contains(Object o) {
    return indexOf(o) >= 0;//当找不到对应元素时返回-1
}

//顺序遍历查找元素
public int indexOf(Object o) {
    if (o == null) {//如果o为null，则在数组中寻找null
        for (int i = 0; i < size; i++)//注意循环至size,而不是elementData.length
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;//返回下标
    }
    return -1;//未找到匹配项返回-1
}

//倒序遍历查找元素，从此方法也可看出元素可重复
public int lastIndexOf(Object o) {
    if (o == null) {
        for (int i = size-1; i >= 0; i--)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = size-1; i >= 0; i--)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

### (4) toArray()
```java
public Object[] toArray() {
    return Arrays.copyOf(elementData, size);
}

public <T> T[] toArray(T[] a) {
    if (a.length < size)
        //若传入的数组长度小于size（ArrayList实际长度）,则返回一个长度为size新的数组, 且将进行类型转换
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());
    //若传入数组长度大于或等于elementData.length，则把elementData复制进传入数组 
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)//若传入的a的长度大于原本数组，则最大下标后面补null 
        a[size] = null;
    return a;
}
```

### (5) get,set
```java
E elementData(int index) {
    return (E) elementData[index];
}

public E get(int index) {

    rangeCheck(index);//index <= size 

    return elementData(index);
}

//直接替换index位置元素，返回原下标内的值
public E set(int index, E element) {
    rangeCheck(index);// index <= size

    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
}
```

### (6) add()
```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);
    elementData[size++] = e;//size++的同时在最后位置复制
    return true;
}
```

首次调用add方法, ensureCapacityInternal(1)会将数组长度扩容值10。之后当size为10时再次进行扩容至容量为15,当size为15时再进行扩容至22。以此类推。

```java
//在index下标的数据前插入element
public void add(int index, E element) {
    rangeCheckForAdd(index);//下标检查

    ensureCapacityInternal(size + 1);  //数组扩容
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);//观察此方法即可的值此add具体作用
    elementData[index] = element;
    size++;
}

//将另一集合添加至此列表尾部
public boolean addAll(Collection<? extends E> c) {
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
    System.arraycopy(a, 0, elementData, size, numNew);
    size += numNew;
    return numNew != 0;//此处返回值设置十分巧妙
}


//将另一个集合从指定位置插入当前列表
public boolean addAll(int index, Collection<? extends E> c) {
    rangeCheckForAdd(index);

    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  //扩容

    int numMoved = size - index;//此值小于等于0
    if (numMoved > 0)
        //如果size > index 则对原数组中数据进行后移操作。如果相等则无需改步
        System.arraycopy(elementData, index, elementData, index + numNew,
                         numMoved);

    //在扩容后数组的指定位置插入新数组，当size==index时即为在尾部添加，加入上条if判断进行些许优化
    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    return numNew != 0;
}
```

### (7) remove()

```java
public E remove(int index) {
    rangeCheck(index);//只检测了index<size

    modCount++;
    E oldValue = elementData(index);//需要移除的元素

    int numMoved = size - index - 1;
    if (numMoved > 0)//numMoved == 0 即为移除最后一位，无需进行数组复制操作
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; //将最后一位设置为null, 同时size减1

    return oldValue;//返回被去除的值
}
```
看了这个remove方法的源码，终于对之前对`list`进行for循环删除时的bug有了解答。当时碰到的bug如下所示：
```
List<String> list = new ArrayList<String>();
list.add("1");
list.add("2");
list.add("3");
list.add("3");
list.add("4");
list.add("5");
System.out.println("原来的list:" + list);
for(int i = 0; i < list.size(); i++){
    String s = list.get(i);
    if ("3".equals(s)){
        list.remove(i);
    }
}
System.out.println("修改后的list:: " + list);
```
输出结果如下图：
![360桌面截图20171214145911.jpg-34.1kB][2]
可以发现连续的'3'并没有被删除，那是因为当运行到i=2时，调用remove()方法，remove方法中的`System.arraycopy`将下标在2后的元素全部前移一位，之后i变为3，继续循环，跳过了原来下标为3的元素。
因而换种写法即可解决该问题：
```java
List<String> list = new ArrayList<String>();
list.add("1");
list.add("2");
list.add("3");
list.add("3");
list.add("4");
list.add("5");
System.out.println("原来的list:" + list);
for(int i = list.size()-1; i >= 0; i--){
    String s = list.get(i);
    if ("3".equals(s)){
        list.remove(i);
    }
}
System.out.println("修改后的list:: " + list);
```
倒序遍历删除时，即便`System.arraycopy`将要删除元素后的所有元素前移，也不会对遍历下标产生影响。


其他remove方法
```java
//类内部快速remove方法
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null;
}

public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)//遍历至size
            if (elementData[index] == null) {
                fastRemove(index);//顺序遍历去除首个null
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}


public void clear() {
    modCount++;

    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;

    size = 0;
}
```
看到这其实又产生了新的疑问，测试：
```java
List<String> list = new ArrayList<String>();
list.add("1");
list.add("2");
list.add("3");
list.add("3");
list.add("4");
list.add("5");
System.out.println("原来的list:" + list);
for (String s : list)
{
    if ("3".equals(s))
    {
        list.remove(s);
    }
}
System.out.println("修改后的list:: " + list);
```
结果如下：
![360桌面截图20171214151509.jpg-82.9kB][3]
观察源码中可以看出remove()方法中并没有利用`modCount`来进行检测并抛出`ConcurrentModificationException`。这个发现让我困惑了很久，从前只是知道用这种遍历抛错，但源码的写法并不符合我的预期。利用`javap`查看对应类的字节码：
![360桌面截图20171214152039.jpg-213.8kB][4]
这下一切就一目了然了，问题不是出在remove上，而是for (String s : list)的foreach循环其实是对实际的Iterable、hasNext、next方法的简写。接下来要看的就是`Iterator`。


## 四. Iterator
&ensp;&ensp;在ArrayList中，还提供了 Iterator 和 ListIterator 这两种迭代器。ListIterator 与普通的 Iterator 相比，它增加了增、删、设定元素、向前向后遍历的操作。

### (1) Iterator()
```java
private class Itr implements Iterator<E> {
    int cursor;       // 下一个返回元素的索引，默认为0
    int lastRet = -1; // 返回的最后一个元素的索引
    int expectedModCount = modCount;//记录itr对象创建时的modCount值，此值在itr中方法仍需调用时不能改变

    public boolean hasNext() {
        return cursor != size;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        checkForComodification();//检查是否在创建了itr对象后list结构是否有改变
        int i = cursor;
        if (i >= size)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i + 1;//索引+1
        return (E) elementData[lastRet = i];
    }

    public void remove() {
        if (lastRet < 0)//此判断表明调用remove前必须先调用next
            throw new IllegalStateException();
        checkForComodification();

        try {
            ArrayList.this.remove(lastRet);
            cursor = lastRet;//删除元素后下标前移一位
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }


    //JDK1.8新添加方法，日后研究
    @Override
    @SuppressWarnings("unchecked")
    public void forEachRemaining(Consumer<? super E> consumer) {
        Objects.requireNonNull(consumer);
        final int size = ArrayList.this.size;
        int i = cursor;
        if (i >= size) {
            return;
        }
        final Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length) {
            throw new ConcurrentModificationException();
        }
        while (i != size && modCount == expectedModCount) {
            consumer.accept((E) elementData[i++]);
        }
        // update once at end of iteration to reduce heap write traffic
        cursor = i;
        lastRet = i - 1;
        checkForComodification();
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}

```

### (2) listIterator
```java
private class ListItr extends Itr implements ListIterator<E> {
    ListItr(int index) {
        super();
        cursor = index;
    }

    public boolean hasPrevious() {
        return cursor != 0;
    }

    public int nextIndex() {
        return cursor;
    }

    public int previousIndex() {
        return cursor - 1;
    }

    @SuppressWarnings("unchecked")
    public E previous() {
        checkForComodification();
        int i = cursor - 1;
        if (i < 0)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i;
        return (E) elementData[lastRet = i];
    }

    public void set(E e) {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            ArrayList.this.set(lastRet, e);
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

    public void add(E e) {
        checkForComodification();

        try {
            int i = cursor;
            ArrayList.this.add(i, e);
            cursor = i + 1;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
}
```
ListItr继承自Itr并实现了ListIterator接口，既可以实现向前遍历，又可以实现向后遍历，并且除了删除操作外，还可以添加数据、更改数据。 

## 六. 总结

 - `ArrayList`底层由数组实现。
 - `ArrayList`非线程安全，只在单线程中使用。  


  [1]: http://static.zybuluo.com/hewei0928/14sq2fg2i9zv3lauqwzx0nig/ArrayList.jpg
  [2]: http://static.zybuluo.com/hewei0928/n0t95izlkq0senv0m7rxrpii/360%E6%A1%8C%E9%9D%A2%E6%88%AA%E5%9B%BE20171214145911.jpg
  [3]: http://static.zybuluo.com/hewei0928/ymgfp2xtsuno8ic2grq7kyfs/360%E6%A1%8C%E9%9D%A2%E6%88%AA%E5%9B%BE20171214151509.jpg
  [4]: http://static.zybuluo.com/hewei0928/azl6veouqgo6iuh462zw6p0z/360%E6%A1%8C%E9%9D%A2%E6%88%AA%E5%9B%BE20171214152039.jpg