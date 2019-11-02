---
title: Java线程池【未完】
date: 2018-04-19 15:51:16
tags: ['Java','线程池']
---

好几次被问到关于线程池的问题，却答不上来，有一种最熟悉的陌生人的感觉。

<!--more-->



# 什么是线程池

> **线程池**（英语：thread pool）：一种[线程](https://zh.wikipedia.org/wiki/%E7%BA%BF%E7%A8%8B)使用模式。线程过多会带来调度开销，进而影响缓存局部性和整体性能。而线程池维护着多个线程，等待着监督管理者分配可并发执行的任务。这避免了在处理短时间任务时创建与销毁线程的代价。线程池不仅能够保证内核的充分利用，还能防止过分调度。可用线程数量应该取决于可用的并发处理器、处理器内核、内存、网络sockets等的数量。 例如，线程数一般取cpu数量+2比较合适，线程数过多会导致额外的线程切换开销。



# 为什么使用线程池

- 复用线程，降低系统开销。

  众所周知，创建/销毁一个线程是需要消耗系统资源的，特别是频繁的创建/销毁。如果当创建一个线程的时间为T1，销毁为T2，线程执行任务的时间为T3，此时T1+T2>T3，那么对于程序来说，这是一种很不划算的方式来执行这项任务。而线程池可以将创建后的线程保存下来，当下次再使用时就不需要重新创建。

- 管理线程。

  创建众多的线程如果没有恰当的管理与调度，那么线程就会混乱。线程池提供了对线程的简单管理和一些执行策略，同时对于任务，有专门的任务队列进行管理。

- 控制并发。

  多线程的情况下，可能会出现对共享资源竞争而产生死锁的情况，此时线程池能够控制并发数，尽可能避免因竞争资源而导致死锁的情况





# ThreadPoolExecutor类

该类是最核心的一个类，`Executors`类中的大多简单工厂方法都是返回ThreadPoolExecutor的实例，只不过对应了不同构造参数，这个待会再讲。

先看下ThreadPoolExecutor类中的重要变量

## corePoolSize

```java
    /**
     * Core pool size is the minimum number of workers to keep alive
     * (and not allow to time out etc) unless allowCoreThreadTimeOut
     * is set, in which case the minimum is zero.
     */
	/**the number of threads to keep in the pool, even
     * if they are idle, unless {@code allowCoreThreadTimeOut} is set
     */
    private volatile int corePoolSize;
```

核心池大小。

从注释可以看出，Core pool size是线程池中（核心）线程的数量，这些线程即使是空闲的，也不会被销毁，除非

`allowCoreThreadTimeOut`这个字段被设置

这里额外扩展下`volatile`这个关键字

### volatile

> 当一个变量被声明为volatile类型后，编译期与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起重排序。volatile变量不会被缓存在寄存器或者对其他处理器看不到的地方，因此在读取volatile变量时总会返回最新写入的值。
>
> 但是，
>
> **volatile无法解决多线程下写入导致数据不一致的问题**，volatile只能确保可见性。
>
> 那什么是**可见性**呢？
>
> > 可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

## maximumPoolSize

```java
    /**
     * Maximum pool size. Note that the actual maximum is internally
     * bounded by CAPACITY.
     */
    private volatile int maximumPoolSize;
```



线程池允许的最大线程数。但实际上不能超过线程自定义的CAPACITY

```java
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
```



## keepAliveTime

```java
    /**
     * Timeout in nanoseconds for idle threads waiting for work.
     * Threads use this timeout when there are more than corePoolSize
     * present or if allowCoreThreadTimeOut. Otherwise they wait
     * forever for new work.
     */
    private volatile long keepAliveTime;
```

空闲线程的超时时间。

当线程数量大于corePoolSize或者allowCoreThreadTimeOut为true时，线程使用这个超时时间来终止自己。否则，这些线程将一直等待到有work为止。



## workQueue

```java
    /**
     * The queue used for holding tasks and handing off to worker
     * threads.  We do not require that workQueue.poll() returning
     * null necessarily means that workQueue.isEmpty(), so rely
     * solely on isEmpty to see if the queue is empty (which we must
     * do for example when deciding whether to transition from
     * SHUTDOWN to TIDYING).  This accommodates special-purpose
     * queues such as DelayQueues for which poll() is allowed to
     * return null even if it may later return non-null when delays
     * expire.
     */
    private final BlockingQueue<Runnable> workQueue;
```

工作队列。

这个队列用于执行任务之前保留任务。即用来存储等待执行的任务。

1. ArrayBlockingQueue：　　

   基于数组结构的有界队列，此队列按FIFO原则对任务进行排序。如果队列满了还有任务进来，则调用拒绝策略。

2. LinkedBlockingQueue：　 

   基于链表结构的无界队列，此队列按FIFO原则对任务进行排序。因为它是无界的，根本不会满，所以采用此队列后线程池将忽略拒绝策略（handler）参数；同时还将忽略最大线程数（maximumPoolSize）等参数。

3. SynchronousQueue：　　

    直接将任务提交给线程而不是将它加入到队列，实际上此队列是空的。每个插入的操作必须等到另一个调用移除的操作；如果新任务来了线程池没有任何可用线程处理的话，则调用拒绝策略。其实要是把maximumPoolSize设置成无界（Integer.MAX_VALUE）的，加上SynchronousQueue队列，就等同于Executors.newCachedThreadPool()。

4. PriorityBlockingQueue：　

   具有优先级的队列的有界队列，可以自定义优先级；默认是按自然排序，可能很多场合并不合适。











## threadFactory

```java
/**
 * Factory for new threads. All threads are created using this
 * factory (via method addWorker).  All callers must be prepared
 * for addWorker to fail, which may reflect a system or user's
 * policy limiting the number of threads.  Even though it is not
 * treated as an error, failure to create threads may result in
 * new tasks being rejected or existing ones remaining stuck in
 * the queue.
 *
 * We go further and preserve pool invariants even in the face of
 * errors such as OutOfMemoryError, that might be thrown while
 * trying to create threads.  Such errors are rather common due to
 * the need to allocate a native stack in Thread.start, and users
 * will want to perform clean pool shutdown to clean up.  There
 * will likely be enough memory available for the cleanup code to
 * complete without encountering yet another OutOfMemoryError.
 */
private volatile ThreadFactory threadFactory;
```

线程工厂。

所有的线程都从这个工厂中产生













# Executors类









# 参考链接

https://liuzho.github.io/2017/04/17/%E7%BA%BF%E7%A8%8B%E6%B1%A0%EF%BC%8C%E8%BF%99%E4%B8%80%E7%AF%87%E6%88%96%E8%AE%B8%E5%B0%B1%E5%A4%9F%E4%BA%86

https://zh.wikipedia.org/wiki/%E7%BA%BF%E7%A8%8B%E6%B1%A0

http://www.cnblogs.com/nayitian/p/3262031.html








> 大部分正义感都是虚伪的 聊胜于无   —————————————————————————沃德彭尤·胡某