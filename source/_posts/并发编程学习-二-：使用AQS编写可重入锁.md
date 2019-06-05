---
title: 并发编程学习(二)：使用AQS编写可重入锁
date: 2019-05-25 14:07:35
categories: [并发编程]
tags: [Java基础,并发编程,可重入锁,AQS]
---
![AQS中文文档介绍](aqs.png)
<div style="width:100%;text-align: center;"><a href="http://www.matools.com/api/java8">AQS中文文档介绍</a></div>

<!--more-->

----------
### 前言
** [上一章](/blog/20190517/编写一个简易的可重入锁-一/) ** 我使用实现`Lock`接口的方式并结合`Synchronized`关键字实现了自己的可重入锁，学习并了解了可重入锁的原理机制。这一章我在学习了AQS之后结合AQS实现自己的显示可重入锁。

### 什么是AQS
如上所述，Java8中文文档中描述的，AQS即`AbstractQueuedSynchronizer`。它提供了一个框架，用于实现依赖先进先出（FIFO）等待队列的阻塞锁和相关同步器（信号量，事件等）。该类被设计为大多数类型的同步器的有用依据，这些同步器依赖于单个原子int值来表示状态。 子类必须定义改变此状态的受保护方法，以及根据该对象被获取或释放来定义该状态的含义。 给定这些，这个类中的其他方法执行所有排队和阻塞机制。 子类可以保持其他状态字段，但只以原子方式更新int使用方法操纵值getState() ， setState(int)和compareAndSetState(int, int)被跟踪相对于同步。**子类应定义为非公共内部助手类，用于实现其封闭类的同步属性。 AbstractQueuedSynchronizer类不实现任何同步接口。 相反，它定义了一些方法，如acquireInterruptibly(int) ，可以通过具体的锁和相关同步器来调用适当履行其公共方法。**

其实AQS类是一个使用了模板方法模式的抽象框架类。它将核心实现封装在模板方法中，提供给程序员去实现具体的加锁和释放的机制，以便于实现一些特殊功能的锁，比如JDK提供的可重入锁和可重入读写锁等等。

### 如何使用AQS
**AQS在使用的时候主要需要重写以下方法**
- `isHeldExclusively()`：该线程是否正在独占资源。只有用到condition才需要去实现它。
- `tryAcquire(int)`：独占方式。尝试获取资源，成功则返回true，失败则返回false。
- `tryRelease(int)`：独占方式。尝试释放资源，成功则返回true，失败则返回false。
- `tryAcquireShared(int)`：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- `tryReleaseShared(int)`：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

### 使用AQS实现自己的可重入独占非公平锁的伪代码如下

#### 加锁步骤伪代码
线程调用`lock`方法加锁，直接调用`sync.acquire(1);`，具体实现在`tryAcquire`
- 首先线程进入想要获取锁
- 拿到当前线程的引用
- 判断加锁状态，如果是未加锁状态
  - 使用`compareAndSetState`自旋原子操作加锁
  - 设置当前线程
  - 返回true加锁成功
- 如果是加锁状态
  - 判断是否是当前线程重入
  - 如果是当前线程重入，state加1，并返回true加锁成功
- 最后如果都不是就返回false加锁失败

#### 释放锁步骤伪代码
线程调用`unLock`方法加锁，直接调用`sync.release(1);`，具体实现在`tryRelease`
- 首先线程进入方法想要释放锁
- 判断如果不是当前线程，就抛出异常
- 如果是当前线程，state就减1（arg一般为1），表示释放一次
- 当state释放到0时，设置拥有锁的线程为null，然后返回true

**具体的代码实现如下**

```

/**
 * @Author zhangjiaheng
 * @Description 使用AQS重写一个可重入锁
 **/
public class MyReentrantLockByAQS implements Lock {

    private Sync sync = new Sync();

    // 内部类Sync ReentrantLock使用的内部抽象类 并派生两个子类实现两种(公平/非公平)锁
    private class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int arg) {
            Thread t = Thread.currentThread();
            // 如果第一个线程进来 可以拿到锁 则返回true

            // 如果第二个线程进来 如果不等于当前线程 返回false 否则更新当前线程值

            int state = getState();
            if (state == 0) {
                while (compareAndSetState(0, 1)) {
                    setExclusiveOwnerThread(t);
                    return true;
                }
            } else if (t == getExclusiveOwnerThread()) {
                // 当前线程再进来
                setState(getState() + 1);
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int arg) {
            // 锁的获取和释放时一一对应的
            // 调用此方法的线程肯定是当前线程
            if (Thread.currentThread() != getExclusiveOwnerThread()) {
                throw new RuntimeException();
            }
            int c = getState() - arg;
            boolean flag = false;
            if (c == 0) {
                setExclusiveOwnerThread(null);
                flag = true;
            }
            setState(c);
            return flag;
        }

        public Condition newCondition() {
            return newCondition();
        }
    }

    @Override
    public void lock() {
        sync.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }
}
```
