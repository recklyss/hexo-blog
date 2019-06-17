---
title: 并发编程学习(四)：理解ThreadPoolExecutor线程池
date: 2019-06-17 14:19:07
categories: [Java基础,并发编程]
tags: [Java基础,并发编程,ThreadPoolExecutor,线程池]
---
![线程池](xcc.png)

<!--more-->

### 前言：关于ThreadPoolExecutor
**ThreadPoolExecutor**即我们常说的线程池。《阿里巴巴Java手册》中对于线程池的使用规定如下：
> **3.【强制】线程资源必须通过线程池提供，不允许在应用中自行显示创建线程。**<br/>
> **说明：使用线程池的好处是减少线程在创建和销毁线程上所花的时间以及系统资源的开销，解决资源不足的问题。如果不使用线程池，有可能造成系统创建大量的同类线程而导致消耗完内存或者“过度切换”的问题​**


### 使用线程池的好处

#### 使用线程池创建线程可以
- 避免在应用中频繁的创建和销毁线程
- 使用线程池创建线程可以复用CPU资源
- 提高线程的可管理性

### 使用线程池的风险 
#### 线程饥饿死锁

线程池为“死锁”这一概念带来了一种新的可能：线程饥饿死锁。在线程池中，如果一个任务将另一个任务提交到同一个Executor，那么通常会引发死锁。第二个线程停留在工作队列中等待第一个提交的任务执行完成，但是第一个任务又无法执行完成，因为它在等待第二个任务执行完成。如下代码所示。
```aidl
public class MyThreadPoolDeadLock {
    static ExecutorService singlePool = Executors.newSingleThreadExecutor();
    static class MyTask implements Callable<String> {
        String name;
        public MyTask(String name) {
            this.name = name;
        }
        @Override
        public String call() throws Exception {
            Future<String> inner = singlePool.submit(new MyTask("inner"));
            return inner.get();
        }
    }
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Future<String> result = singlePool.submit(new MyTask("outer"));
        System.out.println(result.get());
    }
}
```
在更大的线程池中，如果所有线程都由于等待其他仍处于工作队列的任务而阻塞，那么会发生同样的问题，这种情况被称为线程饥饿死锁。

#### 内存溢出
除了Thread 对象所需的内存之外，每个线程都需要两个可能很大的执行调用堆栈。除此以外，JVM 可能会为每个 Java 线程创建一个本机线程，这些本机线程将消耗额外的系统资源。如果线程池的大小设置的不合理就会有可能导致内存溢出的风险。还有就是Java预置线程池FixedThreadPool 和 SingleThreadPool中的阻塞队列使用的无界队列，最多可以保存2147483647个任务，如果代码编写不严谨就会堆积大量请求导致内存溢出。

#### 线程泄漏
各种线程池都会导致一种问题就是线程泄漏。当从线程池取出一个线程去执行任务时，如果任务抛出RuntimeException 或一个Error而未捕获异常时，那么线程只会退出而线程池的大小将永远减少一个，当这种情况发生多次时，线程池最终就会为空并且因为没有可用的线程来处理任务。

### 如果要自己实现线程池需要关注哪些点

- 首先要有一个存放线程的容器并设置容量
- 还需要一个存放用户提交的任务的容器，阻塞队列，有界还是无界
- 线程池创建的时候需要将指定数量的线程启动
- 用户提交任务的时候如果线程池没有空闲的线程如何创建线程并放入线程池
- 线程数量远大于用户提交的任务数量需要有一个回收线程的机制
- 线程全部在执行任务的时候存放的任务需要等待还是怎样或者再新加入任务时要提供一个饱和策略



### ThreadPoolExecutor构造函数参数意义
![构造函数](gzhs.png)
ThreadPoolExecutor提供了四种构造函数，总共有如下几种参数，意义为：
- `int corePoolSize`: 核心线程数的大小，在线程池创建的时候就会创建这么多线程待命，用户提交任务之后立即开始执行任务
- `int maximumPoolSize`: 最大线程数的大小，即最多会创建这么多线程，当超过这个数目的时候可能会在执行完任务之后回收多于核心线程数的线程
- `long keepAliveTime`: 线程最大存活时间，是相对于核心线程数来讲的。没有超过核心线程数的会一直存活的。超过的才有存活时间的限制
- `TimeUnit unit`: 时间单位
- `BlockingQueue<Runnable> workQueue`: 阻塞队列，用于存放用户提交的任务。系统预置的线程池的阻塞队列一般都是无界的LinkBlockingQueue，但是建议使用有界队列，对于非常大或者无界的线程池，可以使用同步移交队列控制避免排队，直接将任务从生产者移交到工作者线程。
- `ThreadFactory threadFactory`: 线程工厂接口。只有一个newThread方法。便于用户根据业务需要实现自己的线程创建机制。
- `RejectedExecutionHandler handler`: 饱和策略。默认四种，在下面讲解。

### 几种默认的饱和策略
当有界队列被填满后，用户创建的任务无法再添加到线程池中保存，饱和策略开始发挥作用。如果某个任务被提交到已关闭的Executors时，饱和策略也会被执行。饱和策略的实现需要实现接口`RejectedExecutionHandler`。
![四种默认的饱和策略](bhcl.png)
如上，在ThreadPoolExecutor类中有四个内部类实现了`RejectedExecutionHandler`接口。分别是:
```aidl
public static class AbortPolicy implements RejectedExecutionHandler {...}
public static class DiscardPolicy implements RejectedExecutionHandler {...}
public static class DiscardOldestPolicy implements RejectedExecutionHandler {...}
public static class CallerRunsPolicy implements RejectedExecutionHandler {...}
```

#### AbortPolicy
“中止”策略是默认的饱和策略，该策略将会抛出一个异常`RejectedExecutionException`，调用者可以捕获这个异常然后编写自己的业务代码。

#### DiscardPolicy
“抛弃”策略会在新提交的任务无法保存在队列中等待执行时将其抛弃掉。

#### DiscardOldestPolicy
同“抛弃”策略，这种策略会将即将执行的那个任务抛弃掉，即抛弃最老的任务然后尝试提交新的任务。如果工作队列使用的是优先队列，那么会导致优先级最高的任务被抛弃，**慎用**！

#### CallerRunsPolicy
“调用者执行”策略即在队列满的时候由调用者去执行该任务。不会在线程池的某个线程中执行新的任务。

>《阿里巴巴Java开发手册》中强调使用线程池的时候尽量使用ThreadPoolExecutor，目的在于让程序员更加明确线程池的工作机制，实际业务中不可能在任务满时将任务抛弃掉，所以实现自己的饱和策略是有必要的。

### Java预置线程池及其使用场景
如图是Executors类中的所有方法
![预置线程池构造](yzxcgz.png)
#### Executors.newCachedThreadPool()
```aidl
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
    }
```
无限容量的线程池(最大为2147483647)，调用ThreadPoolExecutor构造传入的核心线程数为0。适合场景为创建执行时间短效快速的线程任务，线程在执行完成之后直接被回收。阻塞队列使用SynchronousQueue，这是一个不保存数据的队列，因为该线程池有任务提交就会创建线程去执行，所以不需要保存

#### Executors.newFixedThreadPool(nThreads)
```aidl
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```
创建固定数量的线程池。调用ThreadPoolExecutor的构造函数传入的核心线程数等于最大线程数。该线程池中的阻塞队列也使用的是无界的LinkedBlockingQueue。

#### Executors.newSingleThreadExecutor()：
```aidl
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```
每次都只有一个线程去执行任务，用户提交的任务都会排队阻塞在阻塞队列中等待上一个任务执行完之后执行下一个。适用场景为后面任务依赖前面任务的情况。该线程池中的阻塞队列也使用的是无界的LinkedBlockingQueue。使用这个线程池需要小心<a href="#线程饥饿死锁">线程饥饿死锁</a>

#### Executors.newWorkStealingPool()
```aidl
    public static ExecutorService newWorkStealingPool() {
        return new ForkJoinPool
            (Runtime.getRuntime().availableProcessors(),
             ForkJoinPool.defaultForkJoinWorkerThreadFactory,
             null, true);
    }
```
获取当前可用的线程数量进行创建作为并行级别，通过源码可以看出底层调用的是ForkJoinPool线程池，newWorkStealingPool适合使用在很耗时的操作，但是newWorkStealingPool不是ThreadPoolExecutor的扩展，它是新的线程池类ForkJoinPool的扩展，但是都是在统一的一个Executors类中实现，由于能够合理的使用CPU进行对任务操作（并行操作），所以适合使用在很耗时的任务中。


#### Executors.newScheduledThreadPool()
```aidl
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
    ↑↑↑
    ↓↓↓
    ...
    public class ScheduledThreadPoolExecutor
            extends ThreadPoolExecutor
            implements ScheduledExecutorService {
        ...
        public ScheduledThreadPoolExecutor(int corePoolSize) {
            super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
                  new DelayedWorkQueue());
        }
        ...
    }
```
设定延迟时间，定期执行。通过源码可以看出底层调用的是一个ScheduledThreadPoolExecutor，然后传入线程数量。同newWorkStealingPool一样也不是直接使用ThreadPoolExecutor进行扩展。可以延时启动，定时启动的线程池，适用于需要多个后台线程执行周期任务的场景。

### 优雅的关闭线程池

#### shutdown
设置线程池状态为关闭，但是只会关闭已经执行完成的线程，对于还未执行完成的线程，会等待执行完成再关闭。

当我们使用shuwdown方法关闭线程池时，一定要确保任务里不会有永久阻塞等待的逻辑，否则线程池就关闭不了。

#### shutdownNow
立马关闭线程池，线程池里的任务不再执行。

如果我们调用shutdownNow方法时，线程处于从队列里读取任务而阻塞中，则会导致抛出InterruptedException异常

