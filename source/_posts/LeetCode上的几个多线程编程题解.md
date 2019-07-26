---
title: LeetCode上的几个多线程编程题解
date: 2019-07-24 15:16:20
categories: [数据结构与算法,算法题解]
tags: [leetcode,算法题解,并发编程]
---

![](fm.png)



<!--more-->

### [以下所有题目的题解 - github](https://github.com/Fatezhang/DataStructureAndAlgorithm/tree/master/src/main/java/Alogrithm/Alogrithm)

#### 1、[按序打印](https://leetcode-cn.com/problems/print-in-order)

- 方法一：使用volatile变量控制顺序

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

- 方法二：使用CountDownLatch控制顺序（只适用于执行一次。。。可以使用循环栅栏改一下~）

  
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

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;
import java.util.function.IntConsumer;

/**
 * @Author zhangjiaheng @Description https://leetcode-cn.com/problems/print-zero-even-odd
 * 3个线程交替打印奇偶数和0
 */
public class ZeroEvenOdd {
  private int n;

  public ZeroEvenOdd(int n) {
    this.n = n;
  }

  private ReentrantLock lock = new ReentrantLock();

  private Condition c1 = lock.newCondition();
  private Condition c2 = lock.newCondition();
  private Condition c3 = lock.newCondition();

  /** 0-打印0 1-打印奇数 2-打印偶数 */
  private volatile int flag = 0;

  public void zero(IntConsumer printNumber) throws InterruptedException {
    try {
      lock.lock();
      for (int i = 1; i <= n; i++) {
        while (flag != 0) {
          c1.await();
        }
        printNumber.accept(0);
        if ((i & 1) == 1) {
          flag = 1;
          c3.signal();
        } else {
          flag = 2;
          c2.signal();
        }
      }

    } catch (Exception e) {
      e.printStackTrace();
    } finally {
      lock.unlock();
    }
  }

  public void even(IntConsumer printNumber) throws InterruptedException {
    try {
      lock.lock();
      for (int i = 2; i <= n; i += 2) {
        while (flag != 2) {
          c2.await();
        }
        printNumber.accept(i);
        flag = 0;
        c1.signal();
      }

    } catch (Exception e) {
      e.printStackTrace();
    } finally {
      lock.unlock();
    }
  }

  public void odd(IntConsumer printNumber) throws InterruptedException {
    try {
      lock.lock();
      for (int i = 1; i <= n; i += 2) {
        while (flag != 1) {
          c3.await();
        }
        printNumber.accept(i);
        flag = 0;
        c1.signal();
      }

    } catch (Exception e) {
      e.printStackTrace();
    } finally {
      lock.unlock();
    }
  }

  public static void main(String[] args) {
    ZeroEvenOdd zeroEvenOdd = new ZeroEvenOdd(5);
    ThreadPoolExecutor pools =
        new ThreadPoolExecutor(
            3, 3, 1, TimeUnit.MINUTES, new ArrayBlockingQueue<>(1), r -> new Thread(r, "某线程"));
    pools.execute(
        () -> {
          try {
            zeroEvenOdd.zero(System.out::print);
          } catch (InterruptedException e) {
            e.printStackTrace();
          }
        });

    pools.execute(
        () -> {
          try {
            zeroEvenOdd.even(System.out::print);
          } catch (InterruptedException e) {
            e.printStackTrace();
          }
        });
    pools.execute(
        () -> {
          try {
            zeroEvenOdd.odd(System.out::print);
          } catch (InterruptedException e) {
            e.printStackTrace();
          }
        });
  }
}
```





#### 4、[H2O生成](https://leetcode-cn.com/problems/building-h2o)

- 方法一：使用显示锁和condition

```java
  import java.util.concurrent.locks.Condition;
  import java.util.concurrent.locks.ReentrantLock;
  
  /** @Author zhangjiaheng @Description 水分子生成 */
  public class H2O {
    public H2O() {}
  
    private ReentrantLock lock = new ReentrantLock();
  
    private Condition H = lock.newCondition();
    private Condition O = lock.newCondition();
  
    private volatile int hCount = 0;
  
    public void hydrogen(Runnable releaseHydrogen) throws InterruptedException {
      try {
        lock.lock();
        while (hCount == 2) {
          H.await();
        }
        hCount++;
        releaseHydrogen.run();
        if (hCount == 2) {
          O.signal();
        } else {
          H.signal();
        }
      } catch (Exception e) {
        e.printStackTrace();
      } finally {
        lock.unlock();
      }
    }
  
    public void oxygen(Runnable releaseOxygen) throws InterruptedException {
      try {
        lock.lock();
        while (hCount != 2) {
          O.await();
        }
        hCount = 0;
        releaseOxygen.run();
        H.signal();
      } catch (Exception e) {
        e.printStackTrace();
      } finally {
        lock.unlock();
      }
    }
  
    public static void main(String[] args) {
      String s = "HHOOOHHHH";
      final H2O o = new H2O();
      for (char c : s.toCharArray()) {
        if (c == 'H') {
          new Thread(
                  () -> {
                    try {
                      o.hydrogen(() -> System.out.print("H"));
                    } catch (InterruptedException e) {
                      e.printStackTrace();
                    }
                  })
              .start();
        } else {
          new Thread(
                  () -> {
                    try {
                      o.oxygen(() -> System.out.print("O"));
                    } catch (InterruptedException e) {
                      e.printStackTrace();
                    }
                  })
              .start();
        }
      }
    }
  }
```

  

- 方法二：使用信号量控制通知线程

```java
import java.util.Random;
import java.util.concurrent.*;

/** @Author zhangjiaheng @Description 使用信号量控制水分子生成 */
public class H2O_2 {

  private Semaphore hAcquire, oAcquire, hRelease, oRelease;

  public H2O_2() {
    // H 原子线程 请求信号
    hAcquire = new Semaphore(2);
    // O 原子线程 请求信号
    oAcquire = new Semaphore(1);
    // H 原子线程 释放信号
    hRelease = new Semaphore(0);
    // O 原子线程 释放信号
    oRelease = new Semaphore(0);
  }

  public void hydrogen(Runnable releaseHydrogen) throws InterruptedException {
    hAcquire.acquire(); // H线程开始请求
    hRelease.release(); // 通知一个H线程即将释放 因为一个H线程释放最多只有一个O线程等待其释放
    oRelease.acquire(); // 等待O线程释放 一个O线程释放就可以通过
    releaseHydrogen.run();
    hAcquire.release(); // 唤醒H线程请求
  }

  public void oxygen(Runnable releaseOxygen) throws InterruptedException {
    oAcquire.acquire(); // O线程开始请求
    oRelease.release(2); // 通知一个O线程即将释放 因为一个O线程释放 会有两个H线程等待其释放
    hRelease.acquire(2); // 等待H线程释放 要等待两次释放 才可以通过
    releaseOxygen.run();
    oAcquire.release(); // 唤醒O线程请求
  }

  public static void main(String[] args) {
    String s = "HHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHOOHHOOOOOHOOOHHHOHHHHOOOHHHOOOHOHOHOHHOHOOHHHOOHOOOHHOOOOHOHHHHOOOOOHHHOOOHOHOHOOOHHOHOOHHOHHHHHHHHHHHHHHHHHHHHHHHHHHH";
    final H2O_2 o = new H2O_2();
    ThreadPoolExecutor pool =
        new ThreadPoolExecutor(
            300,
            300,
            10,
            TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(100),
                (ThreadFactory) Thread::new);
    for (char c : s.toCharArray()) {
      if (c == 'H') {
        pool.execute(
            () -> {
              try {
                Thread.sleep(new Random().nextInt(10));
                o.hydrogen(() -> System.out.print("H"));
              } catch (InterruptedException e) {
                e.printStackTrace();
              }
            });
      } else {
        pool.execute(
            () -> {
              try {
                Thread.sleep(new Random().nextInt(10));
                o.oxygen(() -> System.out.print("O"));
              } catch (InterruptedException e) {
                e.printStackTrace();
              }
            });
      }
    }
  }
}
```

