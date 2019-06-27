---
title: 并发编程学习(五)：Semaphore源码学习及使用案例
date: 2019-06-23 20:27:19
categories: [Java基础,并发编程]
tags: [Java基础,并发编程,Semaphore,信号量]
---

![](fm.png)



<!--more-->

### Semaphore同步工具类之信号量介绍
#### 什么是Semaphore
- Semaphore是JUC包中的一个很简单的工具类，用来实现多线程下对于资源的同一时刻的访问线程数限制
- Semaphore中存在一个【许可】的概念，即访问资源之前，先要获得许可，如果当前许可数量为0，那么线程阻塞，直到获得许可
- Semaphore内部使用AQS实现，由抽象内部类Sync继承了AQS。因为Semaphore天生就是共享的场景，所以其内部实际上类似于共享锁的实现。
- Semaphore机制是提供给线程抢占式获取许可，所以他可以实现公平或者非公平，类似于ReentrantLock。
- Semaphore提供两个构造方法，用来传入许可数量以及公平或者非公平：
  ```java
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }
    public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
  ```

#### Semaphore的使用场景
- 限流：并发环境(例如有1000个线程)下只允许100个线程访问数据库某资源
- 亦例如实际的，停车场只有10个车位，目前有15个汽车要来停车，多出的5个需要等其他车辆离开之后才能进行停车

### Semaphore源码解读
分为公平与非公平
#### 获取许可的非公平的实现 
在抽象类Sync中实现了非公平的消耗“许可”的方法。
```java
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
        	compareAndSetState(available, remaining))
        return remaining;
    }
}
```

- 首先获取当前许可数量

- 判断消耗许可之后的剩余数量是否>=0

- 是的话执行`compareAndSetState(available, remaining)`设置许可之后返回

- 否则返回的负数会使得其在`doAcquireSharedInterruptibly`中等待许可并挂起，直到被唤醒(这步骤在AQS中实现，如下)
	

  
  ```java
  public final void acquireSharedInterruptibly(int arg)
      throws InterruptedException {
      //如果线程被中断了，抛出异常
      if (Thread.interrupted())
      	throw new InterruptedException();
      //获取许可失败，将线程加入到等待队列中
      if (tryAcquireShared(arg) < 0)
      	doAcquireSharedInterruptibly(arg);
  }
  ```
	
#### 获取许可的公平实现

首先会在获取许可之前，判断`hasQueuedPredecessors()`，是否有线程在等待队列中等待许可，有的话直接返回-1，这个底层实现在AQS中已经实现好了。接下来剩下的操作就和非公平的基本一致了。

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = 2014338818796000944L;

    FairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        for (;;) {
            if (hasQueuedPredecessors())
                return -1;
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
}
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    // 判断头节点不等于尾节点并且（头节点的下一节点为空或者其为当前线程）
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

#### 许可的释放

许可的释放对于公平和非公平的实现都是一致的，定义在Sync类中。因为是共享式的，释放的时候没有像ReentrantLock一样去判断是否是当前线程来释放许可。释放许可也是采用原子操作将需要释放的许可加回去就完成了。

一旦线程调用`releaseShared`释放许可成功，就会同时调用`doReleaseShared`方法，其中会对阻塞的线程进行环型，下面是`tryReleaseShared`的源码。

```java
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        // 拿到当前的许可数量
        int current = getState();
        // 加上还回来的许可
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        // 原子操作 归还许可
        if (compareAndSetState(current, next))
            return true;
    }
}
```

#### 减少许可数量以及将剩余许可数量都取走

Semaphore还提供了几个额外的操作许可的方法

- 减少许可数量

  ```java
  final void reducePermits(int reductions) {
      for (;;) {
          int current = getState();
          int next = current - reductions;
          if (next > current) // underflow
              throw new Error("Permit count underflow");
          if (compareAndSetState(current, next))
              return;
      }
  }
  ```

- 取走剩余全部许可

  ```java
  final int drainPermits() {
      for (;;) {
          int current = getState();
          if (current == 0 || compareAndSetState(current, 0))
              return current;
      }
  }
  ```

  

### 实际使用信号量的代码实例

如下：使用信号量做了一个限流的功能。

在1000个线程并发访问的情况下，每次限制只有100个线程能够获取到资源

```java
public class SemaphoreStudy {

  // 许可的数量
  public static final int N = 100;
  // 线程数量
  public static final int M = 1000;
  // 获取许可失败的次数
  private static final AtomicInteger F = new AtomicInteger();
  // 获取许可成功的次数
  private static final AtomicInteger S = new AtomicInteger();
  // 声明许可
  private static Semaphore store = new Semaphore(N);

  public static void main(String[] args) throws BrokenBarrierException, InterruptedException {
    // 使用栅栏模拟1000并发
    CyclicBarrier BARRIER = new CyclicBarrier(M + 1);
    // 使用线程池创建线程
    ExecutorService pool = Executors.newCachedThreadPool();

    for (int i = 0; i < M; i++) {
      pool.execute(
          () -> {
            try {
              BARRIER.await();
            } catch (Exception e) {
              e.printStackTrace();
            }
            getData();
          });
    }
    System.out.println("等待2秒执行并发1000线程");
    Thread.sleep(2000);
    // 等待两秒后打开栅栏 并发获取数据开始执行
    BARRIER.await();
    pool.shutdown();
  }

  /** 模拟获取数据或者业务处理 */
  public static void getData() {
    while (!store.tryAcquire()) {
      int a = 5000 + new Random().nextInt(1000);
      System.out.println("没有可用资源，等待一小会儿: " + a + "，目前：" + F.incrementAndGet());
      try {
        Thread.sleep(a);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    }

    System.out.println("成功拿到资源");
    try {
      Thread.sleep(5000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    store.release();
    System.out.println("释放资源，现在：" + S.incrementAndGet());
  }
}

```

