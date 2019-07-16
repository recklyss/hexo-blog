---
title: Synchronized使用时一个不起眼的注意点
date: 2019-07-16 17:12:47
categories: [Java基础,并发编程]
tags: [Java基础,并发编程,Synchronized]
---

#### Synchronized 前情提要

Synchronized是Java中用来进行方法或者代码同步的一个内置锁机制。这种内置锁机制可以保证代码执行的原子性、可见性，但是并不能屏蔽代码的重排序。Synchronized可以修饰方法、对象以及代码块，并可以保证被修饰的方法或者代码块，在同一个时刻只能有一个线程能够访问得到。

- 修饰静态方法：锁的是当前类的class对象，修饰方法时Synchronized没有表现在字节码指令中，而是在class文件的方法表中将该方法的access_flags值置为1。表示该方法是同步方法，并使用调用该方法的对象或该方法所属的 Class 在 JVM 的内部对象表示 Klass 作为锁对象。
- 修饰普通方法：锁的是当前实例对象，修饰方法时同上。
- 修饰代码块：锁的是Synchronized()中的对象，编译后的字节码会在代码块前后插入monitorenter 和monitorexit。JVM需要每一个monitorenter都有一个monitorexit与之对应，任何对象都有一个monitor与之相对应，当一个monitor被持有，即线程执行到monitorenter时，对象将处于锁定状态。

Synchronized是Java内置的重量级锁，在jdk1.6之后引入了自旋锁、轻量级锁、适应性自旋、锁粗化、锁消除、偏向锁等技术来减少Synchronized的性能开销。

<!--more-->

#### 切入正题

以上知识点想必刚开始学习并发编程的程序员都会先学习以上知识，但是很多程序员在使用Synchronized的时候有可能会发现，我明明加锁了，但是方法却并没有同步执行，这到底是什么原因？先看下如下代码：

```java
private static Integer cn = 0;
private static final int size = 20;
public static void main(String[] args) {
    for (int j = 0; j < size; j++) {
        new Thread(
            () -> {
                for (int i = 0; i < 5; i++) {
                    synchronized (cn) {
                        cn++;
                    }
                }
            })
            .start();
    }
}
```

以上代码启动了20个线程，对Integer变量cn进行自增。很多人在写Synchronized的时候都有可能出现这种问题。这样的写法是错误的！

因为`cn++`这句代码的原理是将cn指向一个cn+1的新的Integer对象！

修改成如下，然后看看输出：

```java
private static Integer cn = 0;
private static final int size = 20;
private static CountDownLatch cd = new CountDownLatch(size);
public static void main(String[] args) {
    for (int j = 0; j < size; j++) {
        int finalJ = j;
        new Thread(
            () -> {
                try {
                    cd.countDown();
                    cd.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                for (int i = 0; i < 5; i++) {
                    synchronized (cn) {
                        cn++;
                        System.out.println(
                            "cn" + finalJ + " = " + cn + "\t\t\t" + System.identityHashCode(cn));
                    }
                }
            })
            .start();
    }
}
```

**以上代码输出如下 >>**

![输出](cons.png)

每次输出的Integer对象的HashCode值并不相同。所以，每次锁的并不是同一个对象！既然不是同一个对象，那么这个方法在多线程访问的时候肯定就不是线程安全的！对于如上这种例子我们当然可以使用原子变量`AtomicInteger`来实现更高级的同步机制去解决这个问题，但是其他场景下呢？

不仅仅是Integer对象哦！所有的对象都有可能会有这些问题存在！当你在锁这个对象的时候，一定要保证加锁的对象在线程中不被修改成另一个对象！否则就是一个**假的**同步代码块！

----

