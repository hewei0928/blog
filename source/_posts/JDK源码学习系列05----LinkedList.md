---
layout: 'n'
title: JDK源码学习系列05----LinkedList
date: 2017-12-10 12:26:47
tags:
    - Java
    - LinkedList
    - List
    - JDK
---

>`LinkedList`是基于双向链表实现的，它也可以被当作堆栈、队列或双端队列进行操作。

## 一. LinkedList简介
```java
public class LinkedList<E>
extends AbstractSequentialList<E>
implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```
`LinkedList` 继承自`AbStractSequentiaList`,实现了`List`,`Deque`,`Cloneable`,`Serializable`接口

<!--more-->

`AbstractSequentialList`实现了get(),set(),add(),remove()等操作。`List`接口则包含那些列表操作。`Deque`接口则表示能将`LinkedList`当作双端队列使用。`LinkedList`的继承结构图如下所示：
![LinkedList.jpg-71.3kB][1]


  [1]: http://static.zybuluo.com/hewei0928/qh9uwsxu380mxpxzanlaso0l/LinkedList.jpg


## 二. 成员变量

```java
transient int size = 0;//链表中的数据（节点）个数

transient Node<E> first;//链表的头结点

transient Node<E> last;//链表的尾部节点

//LinkedList内部类
private static class Node<E> {
    E item;//存储具体的数据
    Node<E> next;//指向下一个节点
    Node<E> prev;//指向上一个节点

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

从内部类`Node`可以看出，`LinkedList`底层由双向链表实现。

## 三. 构造函数
```java
//初始化一个空链表
public LinkedList() {
}

public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```

## 四. 成员方法

### (1) 重要内部私有方法
```java
//使用对应参数作为头结点
private void linkFirst(E e) {
    final Node<E> f = first;
    //创建一个新节点，next指向头结点
    final Node<E> newNode = new Node<>(null, e, f);//从此处null可以看出底层不是循环链表
    first = newNode;
    if (f == null)//如果头结点为空，即链表长度为0
        last = newNode;
    else//原头结点不为空
        f.prev = newNode;//原头结点的pre指向新头结点
    size++;//长度+1
    modCount++;//修改次数+1
}

//使用对应参数作为尾部结点
void linkLast(E e) {
    //指针l指向原尾节点
    final Node<E> l = last;
    //创建一个新节点，其pre指向原尾节点
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;//将新节点设置为链表尾节点
    if (l == null)//如果原尾节点为空，即链表长度为0
        first = newNode;//
    else//原尾节点不为空
        l.next = newNode;//原尾节点的next指向新尾节点
    size++;
    modCount++;
}

//在succ节点前插入新节点
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;//目标节点的pre为空，则新插入节点设为链表头结点+
    else
        pred.next = newNode;
    size++;
    modCount++;
}

//取消链接非空的第一个节点f。
//移除头结点，并将头结点的next设为头结点
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}

//取消链接非空的尾 节点f。
//移除尾结点，并将尾结点的pre设为尾尾结点
private E unlinkLast(Node<E> l) {
    // assert l == last && l != null;
    final E element = l.item;
    final Node<E> prev = l.prev;
    l.item = null;
    l.prev = null; // help GC
    last = prev;
    if (prev == null)
        first = null;
    else
        prev.next = null;
    size--;
    modCount++;
    return element;
}

//去除某个特定节点
E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}
```

链表内部操作节点的几个方法。从linkedFirst，linkedLast方法即可看出`LinkedList`链表不是循环链表
  
### (2) get()，set() 方法

```java
//获取头结点
public E getFirst() {
    final Node<E> f = first;//f指向头结点
    if (f == null)
        throw new NoSuchElementException();
    return f.item;
}

//获取尾节点
public E getLast() {
    final Node<E> l = last;//l指向尾节点
    if (l == null)
        throw new NoSuchElementException();
    return l.item;
}

//根据下标获取对应节点
public E get(int index) {
    checkElementIndex(index); //[0, size-1]
    return node(index).item;
}

Node<E> node(int index) {
    // assert isElementIndex(index);

    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}

//替换下标的节点内容
public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}
```

### (3) add()，remove()
```java
//去除头结点
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}

//去除尾节点
public E removeLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return unlinkLast(l);
}

//去除特定元素节点
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}


public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}



/////////////add方法

public void addFirst(E e) {
    linkFirst(e);
}

public void addLast(E e) {
    linkLast(e);
}

public boolean add(E e) {
    linkLast(e);
    return true;
}


public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);//index为size时为在链表末尾添加集合
}


//在index元素前插入新集合
public boolean addAll(int index, Collection<? extends E> c) {
    checkPositionIndex(index);//[0,size]

    Object[] a = c.toArray();
    int numNew = a.length;
    if (numNew == 0)
        return false;

    Node<E> pred, succ;
    if (index == size) {//index为size时为在链表末尾添加集合
        succ = null;
        pred = last;
    } else {
        succ = node(index);//index: [0, size-1]
        pred = succ.prev;
    }

    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);//从此行可以看出集合插入位置
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        pred = newNode;
    }

    if (succ == null) {
        last = pred;
    } else {
        pred.next = succ;
        succ.prev = pred;
    }

    size += numNew;
    modCount++;
    return true;
}
```

clear方法:
```java
public void clear() {
    //将对置为空,方便gc回收垃圾
    for (Node<E> x = first; x != null; ) {
        Node<E> next = x.next;
        x.item = null;
        x.next = null;
        x.prev = null;
        x = next;
    }
    first = last = null;
    size = 0;
    modCount++;
}
```

### (4) indexOf()
```java
public int indexOf(Object o) {
    int index = 0;
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null)
                return index;
            index++;
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item))
                return index;
            index++;
        }
    }
    return -1;
}


public int lastIndexOf(Object o) {
    int index = size;
    if (o == null) {
        for (Node<E> x = last; x != null; x = x.prev) {
            index--;
            if (x.item == null)
                return index;
        }
    } else {
        for (Node<E> x = last; x != null; x = x.prev) {
            index--;
            if (o.equals(x.item))
                return index;
        }
    }
    return -1;
}
```

### (5) 队列操作
```java
//出队（从前端），获得第一个元素，不存在会返回null，不会删除元素（节点）
public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}

//出队(从前端)，获得首个节点的数据， 如果首节点为空则报错，不会删除节点
public E element() {
    return getFirst();
}

//出队(从前端)， 从链表前端移除元素（可以为空），并返回该节点的数据
public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}


//出队(从前端)， 从链表前端移除元素（为空则报错），并返回该节点的数据
public E remove() {
    return removeFirst();
}

//入队（从后端），永远返回true
public boolean offer(E e) {
    return add(e);
}

//入栈(从前端)， 永远返回true
public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}

//入队（从后端），永远返回true
public boolean offerLast(E e) {
    addLast(e);
    return true;
}

//出队（从前端），获得第一个元素，不存在会返回null，不会删除元素（节点）
public E peekFirst() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}
 
//出队（从后端），获得最后一个元素，不存在会返回null，不会删除元素（节点）
public E peekLast() {
    final Node<E> l = last;
    return (l == null) ? null : l.item;
}

//出队（从前端），获得第一个元素，不存在会返回null，会删除元素（节点）
public E pollFirst() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}

//出队（从后端），获得最后一个元素，不存在会返回null，会删除元素（节点）
public E pollLast() {
    final Node<E> l = last;
    return (l == null) ? null : unlinkLast(l);
}

//入栈，从前面添加
public void push(E e) {
    addFirst(e);
}

//出栈，返回栈顶元素，从前面移除（会删除）
public E pop() {
    return removeFirst();
}

```

## 五. 迭代器操作

### (1) LinkedList的Iterator遍历
```java
List<String> s = new LinkedList<>();
s.iterator();
```
debug即可发现此方法内部顺序为：
`AbstractSequentialList.iterator()`-->`AbstractList.listIterator()`-->
`LinkedList.listIterator(int index)`;

```java
public ListIterator<E> listIterator(int index) {
    checkPositionIndex(index);
    return new ListItr(index);
}
```

### (2) ListItr

```java
private class ListItr implements ListIterator<E> {
    private Node<E> lastReturned;//上一次操作返回的节点
    private Node<E> next;//指针后的节点
    private int nextIndex;//指针后的节点下标
    private int expectedModCount = modCount;//标记linkedList结构变化

    ListItr(int index) {
        // assert isPositionIndex(index);
        //调用iterator时，index为0
        next = (index == size) ? null : node(index);
        nextIndex = index;
    }

    public boolean hasNext() {
        return nextIndex < size;
    }

    public E next() {
        checkForComodification();//list结构变化检测
        if (!hasNext())
            throw new NoSuchElementException();

        lastReturned = next;
        next = next.next;
        nextIndex++;
        return lastReturned.item;
    }

    public boolean hasPrevious() {
        return nextIndex > 0;
    }

    public E previous() {
        checkForComodification();
        if (!hasPrevious())
            throw new NoSuchElementException();

        lastReturned = next = (next == null) ? last : next.prev;
        nextIndex--;
        return lastReturned.item;
    }

    public int nextIndex() {
        return nextIndex;
    }

    public int previousIndex() {
        return nextIndex - 1;
    }

    public void remove() {
        checkForComodification();
        if (lastReturned == null)
            throw new IllegalStateException();

        Node<E> lastNext = lastReturned.next;
        unlink(lastReturned);
        if (next == lastReturned)
            next = lastNext;
        else
            nextIndex--;
        lastReturned = null;
        expectedModCount++;
    }

    public void set(E e) {
        if (lastReturned == null)
            throw new IllegalStateException();
        checkForComodification();
        lastReturned.item = e;
    }

    public void add(E e) {
        checkForComodification();
        lastReturned = null;
        if (next == null)
            linkLast(e);
        else
            linkBefore(e, next);
        nextIndex++;
        expectedModCount++;
    }

    public void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (modCount == expectedModCount && nextIndex < size) {
            action.accept(next.item);
            lastReturned = next;
            next = next.next;
            nextIndex++;
        }
        checkForComodification();
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

## 六. 总结
- LinkedList是以双链表的形式实现的。
- LinkedList即可以作为链表，还可以作为队列和栈。
- LinkedList是 非 线程安全的。





