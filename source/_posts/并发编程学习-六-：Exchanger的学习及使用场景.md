---
title: 并发编程学习(六)：Exchanger的学习及使用场景
date: 2019-07-01 21:28:39
categories: [Java基础,并发编程]
tags: [Java基础,并发编程,Exchanger,线程交换器]
---

### 前言

在JUC包中，除了一些常用的或者说常见的并发工具类(ReentrantLock，CountDownLatch，CyclicBarrier，Semaphore)等，还有一个不常用的线程同步器类 —— Exchanger。<br>

Exchanger是适用在两个线程之间数据交换的并发工具类，它的作用是找到一个同步点，当两个线程都执行到了同步点(**exchange方法**)之后(*有一个没有执行到就一直等待，也可以设置等待超时时间*)，就将自身线程的数据与对方交换。

<!--more-->

<div style="text-align: center" center><a href="javascript:">Exchanger工具类UML</a></div>
![UML](exc.png)

#### Exchanger类结构

如上图UNML，Exchanger类中有两个内部类，一个Node，一个Participant。

Participant继承了ThreadLocal并且重写了其initialValue方法，返回一个Node对象。

```java
/** The corresponding thread local class */
static final class Participant extends ThreadLocal<Node> {
    public Node initialValue() { return new Node(); }
}
```

Node类封装了两个线程存储的数据对象：

```java
/**
 * Nodes hold partially exchanged data, plus other per-thread
 * bookkeeping. Padded via @sun.misc.Contended to reduce memory
 * contention.
 */
@sun.misc.Contended static final class Node {
    int index;              //  node 在 arena 数组下标
    int bound;              //  交换器的最后记录值 
    int collides;           //  记录的 CAS 失败数
    int hash;               //  伪随机的自旋数
    Object item;            //  这个线程的数据项
    volatile Object match;  //  另一个线程的数据项
    volatile Thread parked; //  当阻塞时，设置此线程，不阻塞的话会自旋
}

```

Exchanger源码分析

```java
@SuppressWarnings("unchecked")
public V exchange(V x) throws InterruptedException {
    Object v;
    Object item = (x == null) ? NULL_ITEM : x; // translate null args
    if ((arena != null || // 是null就执行后面的方法
         (v = slotExchange(item, false, 0L)) == null) &&
        // 如果执行slotExchange有结果就执行后面的，否则返回
        ((Thread.interrupted() || // 非中断则执行后面的方法
          (v = arenaExchange(item, false, 0L)) == null)))
        throw new InterruptedException();
    return (v == NULL_ITEM) ? null : (V)v;
}
```



`exchange`方法的步骤：

- 如果执行slotExchange有结果就执行后面的arenaExchange
- 如果solt被占用，就执行arenaExchange
- 返回的数据v是对方线程的数据项
- 总结即：如果A线程先调用，那么A的数据项存储的item中
- 则B线程的数据项存储在match中
- 当没有多线程并发操作 Exchange 的时候，使用 slotExchange 就足够了。 slot 是一个 node 对象。
- 当出现并发了，一个 slot 就不够了，就需要使用一个 node 数组 arena 操作了。

​	

#### Exchanger的使用

下面的例子模拟一个队列中数据的交换使用的场景：

- 线程A往队列中存入数据
- 线程B从队列中消耗数据
- 当线程A存满的时候
- 才交换给线程B
- 当线程B消耗完成之后才交换给线程A。
- 线程A、B的生产和消耗的速率有可能不同
- 对方线程调用exchange之前，另一个线程执行到exchange会阻塞

```java
/** 在对方线程调用exchange之前，另一个线程执行到exchange会阻塞 直到双方都调用exchange */
public class ExchangerStudy {

  private static ArrayBlockingQueue<String> initialFillQueue 
      = new ArrayBlockingQueue<>(5);
  private static ArrayBlockingQueue<String> initialEmptyQueue 
      = new ArrayBlockingQueue<>(5);
  private static Exchanger<ArrayBlockingQueue<String>> exchanger 
      = new Exchanger<>();

  /** 填充缓存队列的线程 */
  static class FillingRunnable implements Runnable {
    @Override
    public void run() {
      ArrayBlockingQueue<String> current = initialEmptyQueue;
      try {
        while (current != null) {
          String str = StrUtil.uuid();
          System.out.println("生产了一个序列：" + str + ">>>>>加入到交换区");
          Thread.sleep(2000);
          try {
            current.add(str);
          } catch (IllegalStateException e) {
            System.out.println("队列已满，换一个空的");
            current = exchanger.exchange(current);
          }
        }
      } catch (Exception e) {
        e.printStackTrace();
      }
    }
  }
  /** 填充缓存队列的线程 */
  static class EmptyingRunnable implements Runnable {
    @Override
    public void run() {
      ArrayBlockingQueue<String> current = initialFillQueue;
      try {
        while (current != null) {
          if (!current.isEmpty()) {
            String str = current.poll();
            System.out.println("消耗一个数列：" + str);
          } else {
            System.out.println("队列空了，换个满的");
            current = exchanger.exchange(current);
            System.out.println("换满的成功~~~~~~~~~~~~~~~~~~~~~~");
          }
        }
      } catch (Exception e) {
        e.printStackTrace();
      }
    }
  }

  public static void main(String[] args) {
    new Thread(new FillingRunnable()).start();
    new Thread(new EmptyingRunnable()).start();

  }
}

```



----

> [>>>>> 更详细的源码解析 - 掘金](https://juejin.im/post/5ae7554ff265da0b86360880)

![结尾](https://user-gold-cdn.xitu.io/2018/5/1/16317a536c642f7c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

