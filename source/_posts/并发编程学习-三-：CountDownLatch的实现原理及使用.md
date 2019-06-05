---
title: 并发编程学习(三)：CountDownLatch的实现原理及使用
date: 2019-06-02 21:36:53
categories: [并发编程]
tags: [Java基础,并发编程,AQS,CountDownLatch]
---
![什么是CountDownLatch？](cdl.png)

<!--more-->

### 什么是`CountDownLatch`？

  在本篇博客的封面，我放了一个截图，上面对于`CountDownLatch`的翻译是这样的：*闭锁，倒计时门闩*。其实顾名思义，`CountDownLatch`实际上就是一个计数器：**计数-计数完成后做一些事**。其实这个东西可以类比为一个水坝：当水还没有装满水库的时候水坝是关闭的，当水装满之后开闸放水，水库中的水"一起"涌出水库。

  拥有同样功能的还有`CyclicBarrier`这个类，但是这个类相对较复杂，并且相对于`CountDownLatch`还可以重复使用，实际上前者一般被叫做线程计数器，后者被叫做循环屏障，还是有很大区别的。这个 **在后面再进行源码学习**。

### `CountDownLatch`是如何实现的？

  同`ReentrantLock`类似，内部也是有一个实现了`AbstractQueueSynchronizer`的内部类。内部类做了父类的共享式的显示锁的方法实现，维护一个初始为N的状态`state`，每次有线程调用之后阻塞，然后`state`减1，直到减为0之后所有阻塞的线程重新开始执行。

#### 首先是内部类Sync的实现
  构造器接收一个int参数初始化state的值。`tryAcquireShared()`方法不会对state做改变，当state不为0的时候返回-1即失败，当state等于0其返回1，表示计数器已经计数完成，`await()`方法不再阻塞。`tryReleaseShared()`方法会使用原子操作当`countDown()`被调用的时候释放一个state的占用，即state-1。

```
private static final class Sync extends AbstractQueuedSynchronizer {
      private static final long serialVersionUID = 4982264981922014374L;
      Sync(int count) {
          setState(count);
      }
      int getCount() {
          return getState();
      }
      protected int tryAcquireShared(int acquires) {
          return (getState() == 0) ? 1 : -1;
      }
      protected boolean tryReleaseShared(int releases) {
          // Decrement count; signal when transition to zero
          for (;;) {
              int c = getState();
              if (c == 0)
                  return false;
              int nextc = c-1;
              if (compareAndSetState(c, nextc))
                  return nextc == 0;
          }
      }
}
```

#### CountDownLatch的countDown方法
  countDown方法主要作用就是使state-1

```
public void countDown() {
    sync.releaseShared(1);
}
```

  AQS中的`releaseShared()`方法的实现，如果释放成功执行`doReleaseShared();`

```
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

#### CountDownLatch的await方法
  await方法会等待当前state值是否是0，如果不是的话就一直阻塞。直到state为0。

```
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

  AQS中的`acquireSharedInterruptibly()`方法实现如下，在AQS的实现中，判断当前线程是否中断，是的话抛出中断异常，否则判断当前线程是否继续需要阻塞，即调用`tryAcquireShared()`。是的话进入`doAcquireSharedInterruptibly()`方法，不断的判断`int r = tryAcquireShared(arg);`，state如果一直不等于0，r就一直是负数，就会继续进入循环。
```
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
/**
 * Acquires in shared interruptible mode.
 * @param arg the acquire argument
 */
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

其实以上代码的整体流程非常简单，即初始化`CountDownLatch`的state=N，每次调用countDown时state-1，减到0的时候停止阻塞，继续向下执行。

### 我可以用`CountDownLatch`来做什么事情？
#### 使用`CountDownLatch`模拟并发场景
- 可以使用`CountDownLatch`，创建多个线程并等待线程全部就绪之后唤醒所有线程。可以用这种方式测试代码的可用性，或者测试单例类等；

我在自己学习过程中也有写过类似的测试类 - [github](https://github.com/Fatezhang/Concurrent/tree/master/src/main/java/com/mime/concurrent/CountDownLatchStudy)

#### 使用`CountDownLatch`等待依赖线程执行
- `CountDownLatch`用来等待其他依赖服务都启动好之后在进行自身线程的任务处理

### 总结
  `CountDownLatch`是面试的时候多线程这块很容易被问到的点，实际上会考察这几个方面：
  - 1、内部实现原理 **——** 使用内部类继承AQS实现；
  - 2、需要注意的方面 **——** 计数器为0时，await后面的方法才会执行，否则一直阻塞，countDown方法尽量写在finally代码块中，避免出现异常导致死锁；
  - 3、使用场景 **——** 监控一些依赖服务启动完成之后执行代码，或者造“水坝”，即模拟大量并发场景等。
