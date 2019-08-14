---
title: 并发编程学习(七)：Fork/Join框架原理及demo
date: 2019-08-14 16:48:56
categories: [Java基础,并发编程]
tags: [Java基础,并发编程,Fork/Join框架,ForkJoinPool,线程池]
---

![fm.jpg](fm.jpg)

<!--more-->

### 概念

Fork/Join框架是jdk1.7引入的一个基于“分治”思想的多线程框架。它的功能是将一个大任务切分(**fork**)成多个相同逻辑的小任务，分而治之，当子任务全都执行完成之后，将结果合并(**join**)起来，最终成为整体任务的执行结果。原理可以抽象成下图表示：

![Fork/Join](fj.png)

### Fork/Join相关代码原理及思想

###### Fork/Join执行步骤

1. 进行任务分割：将任务分割成小任务，然后这个小任务有可能还需要继续分割，直到足够小。

2. 执行并合并结果：分割的子任务分别放在双端队列里，然后几个启动线程分别从双端队列里获取任务执行。子任务执行完的结果都统一放在一个队列里，启动一个线程从队列里拿数据，然后合并这些数据。


Fork/Join使用两个类完成以上步骤：

- **ForkJoinTask**：
  - Fork/Join提供了两个子类：RecursiveAction：用于没有返回结果的任务；RecursiveTask ：用于有返回结果的任务
- **ForkJoinPool** ：`public class ForkJoinPool extends AbstractExecutorService{ ... }`ForkJoinTask需要通过ForkJoinPool来执行，任务分割出的子任务会添加到当前工作线程所维护的双端队列中，进入队列的头部。当一个工作线程的队列里暂时没有任务时，它会随机从其他工作线程的队列的尾部获取一个任务。这种算法成为**工作窃取算法(work-stealing)**

###### 工作窃取算法(work-stealing)

- Fork/Join框架内部实现了一个类似于LinkedBlockingDeque的双端队列用作工作线程的任务队列**WorkQueue**。使用`ForkJoinWorkerThread`保存工作线程，`ForkJoinPool.WorkQueue`就在其内部。

- Fork/Join每个工作线程在运行中产生了新的任务(通常是调用fork方法)的时候，将任务加入WorkQueue尾部，并且工作线程每次取出任务执行也是从队尾取出执行，即LIFO

- 每个工作线程在处理自己的工作队列同时，会尝试窃取一个任务（或是来自于刚刚提交到 pool 的任务，或是来自于其他工作线程的工作队列），窃取的任务位于其他线程的工作队列的队首，也就是说工作线程在窃取其他工作线程的任务时，使用的是 FIFO 方式。

- 在遇到 join() 时，如果需要 join 的任务尚未完成，则会先处理其他任务，并等待其完成。

- 在既没有自己的任务，也没有可以窃取的任务时，进入休眠。

### Fork/Join demo演示

> 使用Fork/Join完成大量有序数字的加和

```java
public class MyCalculateTask extends RecursiveTask<Integer> {

  private static final int THREADSHOLD = 50;

  private int start;
  private int end;
  private List<String> list;

  public MyCalculateTask(int start, int end, List<String> list) {
    this.start = start;
    this.end = end;
    this.list = list;
  }

  @Override
  protected Integer compute() {
    int sum = 0;
    if (end - start < THREADSHOLD) {
      // 当两数字之间差值小于指定值 就不再查分成小任务 
      String so = "";
      for (int i = start; i < end; i++) {
        sum += i;
        so += list.get(i) + ",";
      }
      System.out.println(Thread.currentThread().getName() + "处理 " + so + " 的数据");
    } else {
      int mid = (start + end) / 2;
      // 一分为二 拆分任务
      final MyCalculateTask left = new MyCalculateTask(start, mid, list);
      final MyCalculateTask right = new MyCalculateTask(mid, end, list);
      left.fork();
      right.fork();
      sum += left.join();
      sum += right.join();
    }

    return sum;
  }

  public static void main(String[] args) throws InterruptedException, ExecutionException {
    int sum = 0;
    ForkJoinPool pool = new ForkJoinPool();
    int count = 400;
    List<String> list = new ArrayList<>(count);
    for (int i = 0; i < count; i++) {
      list.add("i-" + i);
      sum += i;
    }
    MyCalculateTask task = new MyCalculateTask(0, count, list);
    final ForkJoinTask<Integer> submit = pool.submit(task);
    System.out.println("sum = " + sum + " --- submit.get() = " + submit.get());
    pool.awaitTermination(5, TimeUnit.SECONDS);
    pool.shutdown();
  }
}
```

### 总结

Fork/Join框架可以帮助我们完成很多这种大任务可以拆分成小任务执行的场景，不过上面的方法并不是最佳执行调用方式

```
left.fork();  
right.fork();
替换为
invokeAll(left, right);
```

因为对于Fork/Join模式，假如Pool里面线程数量是固定的，那么调用子任务的fork方法相当于A先分工给B，然后A当监工不干活，B去完成A交代的任务。所以上面的模式相当于浪费了一个线程。那么如果使用invokeAll相当于A分工给B后，A和B都去完成工作。这样可以更好的利用线程池，缩短执行的时间。



> 参考：http://ifeve.com/talk-concurrency-forkjoin/