---
title: LeetCode上的几个多线程编程题解
date: 2019-07-24 15:16:20
categories: [数据结构与算法,算法题解]
tags: [leetcode,算法题解,并发编程]
---

![](fm.png)



<!--more-->



#### 1、[按序打印](https://leetcode-cn.com/problems/print-in-order)

- 方法一：使用显示锁与condition

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 三个线程 按序打印
 */
public class Foo {

    public Foo() {

    }

    private ReentrantLock lock = new ReentrantLock();

    private Condition c1 = lock.newCondition();
    private Condition c2 = lock.newCondition();
    private Condition c3 = lock.newCondition();

    private int state = 1;

    public void first(Runnable printFirst) throws InterruptedException {
        try {
            lock.lock();
            if (state != 1) {
                c1.await();
            }
            state = 2;
            // printFirst.run() outputs "first". Do not change or remove this line.
            printFirst.run();
            c2.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void second(Runnable printSecond) throws InterruptedException {
        try {
            lock.lock();
            if (state != 2) {
                c2.await();
            }
            state = 3;
            // printSecond.run() outputs "second". Do not change or remove this line.
            printSecond.run();
            c3.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }

    }

    public void third(Runnable printThird) throws InterruptedException {
        try {
            lock.lock();
            if (state != 3) {
                c3.await();
            }
            state = 1;

            // printThird.run() outputs "third". Do not change or remove this line.
            printThird.run();
            c1.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

}

```

- 方法二：使用volatile状态变量控制顺序

```java
public class Foo2 {

    private volatile int state = 1;

    public void first(Runnable printFirst) throws InterruptedException {
        while (state != 1) {

        }
        // printFirst.run() outputs "first". Do not change or remove this line.
        printFirst.run();
        state = 2;
    }

    public void second(Runnable printSecond) throws InterruptedException {
        while (state != 2) {

        }
        // printSecond.run() outputs "second". Do not change or remove this line.
        printSecond.run();
        state = 3;
    }

    public void third(Runnable printThird) throws InterruptedException {
        while (state != 3) {

        }
        // printThird.run() outputs "third". Do not change or remove this line.
        printThird.run();
        state = 1;
    }
}

```

- 方法三：使用CountDownLatch控制顺序
```java
import java.util.concurrent.CountDownLatch;

public class Foo3 {

    private CountDownLatch countDownLatch2 = new CountDownLatch(1);
    private CountDownLatch countDownLatch3 = new CountDownLatch(1);

    public void first(Runnable printFirst) throws InterruptedException {
        countDownLatch3.await();
        countDownLatch2.await();
        // printFirst.run() outputs "first". Do not change or remove this line.
        printFirst.run();
        countDownLatch2.countDown();
    }

    public void second(Runnable printSecond) throws InterruptedException {

        countDownLatch3.await();
        countDownLatch2.await();
        // printSecond.run() outputs "second". Do not change or remove this line.
        printSecond.run();
        countDownLatch3.countDown();
    }

    public void third(Runnable printThird) throws InterruptedException {

        countDownLatch3.await();
        // printThird.run() outputs "third". Do not change or remove this line.
        printThird.run();
    }
}
```



#### 2、[交替打印FooBar](https://leetcode-cn.com/problems/print-foobar-alternately)

```java

import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/** @Author zhangjiaheng @Description 交替打印FooBar */
public class FooBar {
  private int n;

  public FooBar(int n) {
    this.n = n;
  }

  private ReentrantLock lock = new ReentrantLock();

  private Condition c1 = lock.newCondition();
  private Condition c2 = lock.newCondition();

  private volatile boolean flag = false;

  public void foo(Runnable printFoo) throws InterruptedException {

    for (int i = 0; i < n; i++) {
        try {
            lock.lock();
            if (flag) {
                c1.await();
            }
            // printFoo.run() outputs "foo". Do not change or remove this line.
            printFoo.run();
            flag = !flag;
            c2.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
  }

  public void bar(Runnable printBar) throws InterruptedException {

    for (int i = 0; i < n; i++) {
        try {
            lock.lock();
            if (!flag) {
                c2.await();
            }
            // printBar.run() outputs "bar". Do not change or remove this line.
            printBar.run();
            flag = !flag;
            c1.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
  }
}

```





#### 3、[打印0与奇偶数](https://leetcode-cn.com/problems/print-zero-even-odd)





#### 4、[H2O生成](https://leetcode-cn.com/problems/building-h2o)