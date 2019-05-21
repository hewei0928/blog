---
title: 多线程及并发学习05-synchornized的用法
date: 2018-05-04 13:42:41
tags:
    - Java
    - 多线程
---


`synchornized`关键字用于控制同步，它修饰的对象主要有以下几种：
- 修饰一个代码块，称为同步语块，其作用是获取某个对象或类的锁，直到同步语块结束释放锁
- 修饰一个方法，被修饰的方法称为同步方法，其作用是获取`this`对象的锁，直至方法运行结束释放锁
- 修饰一个静态方法，其作用是获取类锁，直至方法运行结束释放锁

<!--more-->

## 修饰代码块

### synchornized(this) 线程调用对象自身的锁，其他试图获取该对象锁的线程被阻塞。
```java
public class Sync {

    public static void main(String[] args) {
        SyncThread syncThread = new SyncThread();
        Thread thread1 = new Thread(syncThread, "SyncThread1");
        Thread thread2 = new Thread(syncThread, "SyncThread2");
        thread1.start();
        thread2.start();
    }

}



/**
 * 同步线程
 */
class SyncThread implements Runnable {
    private int count;

    public SyncThread() {
        count = 0;
    }

    public  void run() {
        synchronized(this) {
            for (int i = 0; i < 5; i++) {
                try {
                    System.out.println(Thread.currentThread().getName() + ":" + (count++));
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public int getCount() {
        return count;
    }
}
```

运行结果如下，线程1运行时，需要获取同一对象锁的线程2被阻塞

![TIM截图20180504142208.png-53kB][1]

<br/>
**如果两个线程获取的不是同一对象的锁，则不会发生阻塞：**
```java
public class Sync {

    public static void main(String[] args) {
        SyncThread syncThread = new SyncThread();
        SyncThread syncThread1 = new SyncThread();
        Thread thread1 = new Thread(syncThread, "SyncThread1");
        Thread thread2 = new Thread(syncThread1, "SyncThread2");
        thread1.start();
        thread2.start();
    }

}
```
运行结果如下，两个线程并没有发生阻塞
![TIM截图20180504142809.png-27.2kB][2]

**其他线程任然可以运行该线程的非`synchornized`方法， 因为调用那些方法的线程，并没有尝试去获取该对象的锁：**
```java
public class Sync {

    public static void main(String[] args) {
        SyncThread syncThread = new SyncThread();
        Thread thread1 = new Thread(syncThread, "SyncThread1");
//        Thread thread2 = new Thread(syncThread, "SyncThread2");
        thread1.start();
//        thread2.start();
        syncThread.getCount();
    }

}



/**
 * 同步线程
 */
class SyncThread implements Runnable {
    private int count;

    public SyncThread() {
        count = 0;
    }

    public  void run() {
        synchronized(this) {
            for (int i = 0; i < 5; i++) {
                try {
                    System.out.println(Thread.currentThread().getName() + ":" + (count++));
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public void getCount() {
        for (int i = 0; i < 5; i++) {
            try {
                System.out.println(Thread.currentThread().getName() + ":" + (count++));
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```
运行结果如下，并未发生阻塞：

![TIM截图20180504153935.png-38.4kB][3]

### synchornized(object) 线程获得某对象的锁，其他试图获取该对象锁的线程被阻塞。
同上一条。

### synchornized(xxx.class) 线程获得该类的锁，其他试图获取该类锁的线程被阻塞。
将上个例子改为如下：
```java
class SyncThread implements Runnable {
    private  int count;

    public SyncThread() {
        count = 0;
    }

    public  void run() {
        synchronized(SyncThread.class) {
            for (int i = 0; i < 5; i++) {
                try {
                    System.out.println(Thread.currentThread().getName() + ":" + (count++));
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public int getCount() {
        return count;
    }
}
```
运行结果如下：
![TIM截图20180504143827.png-35.3kB][4]

对于要获取同一对象锁的线程1和2会发生阻塞。

## 修饰方法

### 修饰非静态方法， 线程获取对象的锁，其他尝试获取该对象锁的线程都会被阻塞
情况同对象锁同步代码块。

```java

public class Sync {

    public static void main(String[] args) {
        c c = new c();
        SyncThread syncThread = new SyncThread(c);
        Thread thread1 = new Thread(syncThread, "SyncThread1");
//        Thread thread2 = new Thread(syncThread, "SyncThread2");
        thread1.start();
        c.d(0);
    }

}



/**
 * 同步线程
 */
class SyncThread implements Runnable {
    private int count;

    private c x;

    public SyncThread(c x) {
        count = 0;
        this.x = x;
    }

    public  void run() {
        synchronized(x) {
            x.c(count++);
        }
    }


}

class c {

    public void c(int count) {
        for (int i = 0; i < 5; i++) {
            try {
                System.out.println(Thread.currentThread().getName() + ":" + (count));
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public synchronized void d(int count) {
        for (int i = 0; i < 5; i++) {
            try {
                System.out.println(Thread.currentThread().getName() + ":" + (count++));
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

运行结果显示会发生阻塞，因为`main`线程和`SyncThread1`尝试去获取同一对象的锁，必然有一个线程发生阻塞。

### 修饰静态方法，线程获取类的锁，其他尝试获取类的锁的线程被阻塞
情况同类锁同步代码块


  [1]: http://static.zybuluo.com/hewei0928/lw9gulqpri63h9fzp4kqle89/TIM%E6%88%AA%E5%9B%BE20180504142208.png
  [2]: http://static.zybuluo.com/hewei0928/msohiir9nwvt6ju34ulsg0xd/TIM%E6%88%AA%E5%9B%BE20180504142809.png
  [3]: http://static.zybuluo.com/hewei0928/u1vbb0af5s98oiz8ugbbw7of/TIM%E6%88%AA%E5%9B%BE20180504153935.png
  [4]: http://static.zybuluo.com/hewei0928/5miqbmu7tw6bs5ncg8wj1jij/TIM%E6%88%AA%E5%9B%BE20180504143827.png