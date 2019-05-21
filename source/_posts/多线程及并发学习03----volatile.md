---
title: 多线程及并发学习03----volatile 与 可见性
date: 2018-05-03 08:45:36
tags:
    - Java
    - 多线程
    - 并发
---

> 很早以前就接触了volatile， 但是并没有特别深入的去研究。只是模糊的知道可以用它来解决可见性问题，但是具体什么是可见性问题以及具体原理是什么却不甚了解。最近翻阅了许多文档，并结合自己的思考和实践对`volatile`有了比较深刻的认知。特此记录以防遗忘。

<!--more-->

## 可见性及其相关概念

- **可见性**是指一个线程对**共享变量**值的修改，能够及时地被其他线程看到。
- **共享变量**是指如果一个变量在多个线程的**工作内存**中都存在，那么这个变量就是这几个线程的共享变量。

### java内存模型

`java`内存模型（JMM）分为主内存和工作内存。一个程序用到的所有变量都存储在主内存中。而每个线程都有自己独立的工作内存，里面保存该线程所使用到的共享变量的副本（主内存中该变量的一份拷贝）。大致结构如下图：

![TIM截图20180503092017.png-99.1kB][1]

**`java`内存模型中有两条规定:**
- 线程对**共享变量**的所有操作都必须在自己的工作内存中进行，不能直接从主内存中读写。
- 不同线程之间无法直接访问其他线程工作内存中的**共享变量**，线程间**共享变量**值的传递需要通过主内存来完成。

**线程1对共享变量的修改要想被线程2及时看到，必须要经过如下两个步骤：**
- 把工作内存1中更新过的**共享变量**刷新到主存中
- 将内存中最新的**共享变量**的值更新到工作内存2中

可见性问题反映到代码中示例如下：
```java
public class Task implements Runnable {

    boolean running = true;
    int i = 0;

    @Override
    public void run() {
        while (running) {
            //System.out.println(...); 加入这行代码后可以跳出无限循环
            i();
        }
    }

    public int i(){
        return i++;
    }

    public static void main(String[] args) throws Exception {
        Task task = new Task();
        Thread th = new Thread(task);
        th.start();
        Thread.sleep(10);
        task.running = false;
        Thread.sleep(100);
        System.out.println(task.i);
        System.out.println("程序停止");
    }
}
```
初次看到这个示例代码时我以为能够正常结束，但实际运行结果却和自己想的大相径庭：程序会一直运行下去，无法停止。结合可见性原理可以大致分析原因：主线程对`running`
字段值的修改不会马上同步至`th`线程的工作内存中，因此造成了无限循环。

##  synchronized实现可见性

**`java`内存模型关于`synchronized`的两条规定**
- 线程解锁前，必须把共享变量的最新值刷新至主内存中。
- 线程加锁时，将清空工作内存中共享变量的值，从而使共享变量需要从主内存中重新读取最新的值。

**这两条规定保证了共享变量的可见性**
因此上面的代码只需改为如下所示即可：
```java
public class Task implements Runnable {

    boolean running = true;
    int i = 0;

    @Override
    public void run() {
        while (running) {
            i();
        }
    }

    public synchronized int i(){
        return i++;
    }

    public static void main(String[] args) throws Exception {
        Task task = new Task();
        Thread th = new Thread(task);
        th.start();
        Thread.sleep(10);
        task.running = false;
        Thread.sleep(100);
        System.out.println(task.i);
        System.out.println("程序停止");
    }
}
```
运行结果程序能正常停止。

## volatile实现可见性
&ensp;&ensp;具体原理可以看最后的参考资料，大致原理可以理解为：`volatile`变量在每次被线程访问时，都强迫从主存中重读该变量的值，而当该变量发生变化时，又会强迫线程将最新的值刷新到主内存。这样就实现了任何时刻，不同的线程总能看到该变量的最新值。
之前的示例更换如下：
```java
public class Task implements Runnable {

    boolean volatile running = true;
    int i = 0;

    @Override
    public void run() {
        while (running) {
            i();
        }
    }

    public int i(){
        return i++;
    }

    public static void main(String[] args) throws Exception {
        Task task = new Task();
        Thread th = new Thread(task);
        th.start();
        Thread.sleep(10);
        task.running = false;
        Thread.sleep(100);
        System.out.println(task.i);
        System.out.println("程序停止");
    }
}
```

## volatile适用场合

**要在多线程中安全的使用`volatile`变量，必须同时满足：**
- 对变量的写入操作不依赖当前值：如`number++`、`count *= 5`则不满足
- 该变量没有包含在具有其他变量的不变式中

[参考资料](https://www.imooc.com/video/6817)

  [1]: http://static.zybuluo.com/hewei0928/lxioc01s8qmytz19p9eptmjq/TIM%E6%88%AA%E5%9B%BE20180503092017.png