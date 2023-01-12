---
layout: post
title:  "ThreadPoolExecutor 介绍"
subtitle: ""
date:   2019-03-22
background: '/img/imac_bg.png'
---

## 使用`Executors`创建线程池：
共有四种线程池：
#### 1. `CachedThreadPool` 可缓存线程池

```java
    /**
     * Creates a thread pool that creates new threads as needed, but
     * will reuse previously constructed threads when they are
     * available.  These pools will typically improve the performance
     * of programs that execute many short-lived asynchronous tasks.
     * Calls to {@code execute} will reuse previously constructed
     * threads if available. If no existing thread is available, a new
     * thread will be created and added to the pool. Threads that have
     * not been used for sixty seconds are terminated and removed from
     * the cache. Thus, a pool that remains idle for long enough will
     * not consume any resources. Note that pools with similar
     * properties but different details (for example, timeout parameters)
     * may be created using {@link ThreadPoolExecutor} constructors.
     *
     * @return the newly created thread pool
     */
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }

    /**
     * Creates a thread pool that creates new threads as needed, but
     * will reuse previously constructed threads when they are
     * available, and uses the provided
     * ThreadFactory to create new threads when needed.
     * @param threadFactory the factory to use when creating new threads
     * @return the newly created thread pool
     * @throws NullPointerException if threadFactory is null
     */
    public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      threadFactory);
    }
```
注意两个要点：
[1.`SynchronousQueue`是一个没有数据缓冲的`BlockingQueue`，生产者线程对其的插入操作`put`必须等待消费者的移除操作`take`，反过来也一样。](https://www.cnblogs.com/duanxz/p/3252267.html)
2.核心线程数为0，最大线程总数为int最大值。
综上：如果创建的线程阻塞，新的`task`添加进来会不断创建线程，最终耗尽资源。

#### 2. `FixedThreadPool` 定长线程池

```java
    /**
     * Creates a thread pool that reuses a fixed number of threads
     * operating off a shared unbounded queue.  At any point, at most
     * {@code nThreads} threads will be active processing tasks.
     * If additional tasks are submitted when all threads are active,
     * they will wait in the queue until a thread is available.
     * If any thread terminates due to a failure during execution
     * prior to shutdown, a new one will take its place if needed to
     * execute subsequent tasks.  The threads in the pool will exist
     * until it is explicitly {@link ExecutorService#shutdown shutdown}.
     *
     * @param nThreads the number of threads in the pool
     * @return the newly created thread pool
     * @throws IllegalArgumentException if {@code nThreads <= 0}
     */
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

    /**
     * Creates a thread pool that reuses a fixed number of threads
     * operating off a shared unbounded queue, using the provided
     * ThreadFactory to create new threads when needed.  At any point,
     * at most {@code nThreads} threads will be active processing
     * tasks.  If additional tasks are submitted when all threads are
     * active, they will wait in the queue until a thread is
     * available.  If any thread terminates due to a failure during
     * execution prior to shutdown, a new one will take its place if
     * needed to execute subsequent tasks.  The threads in the pool will
     * exist until it is explicitly {@link ExecutorService#shutdown
     * shutdown}.
     *
     * @param nThreads the number of threads in the pool
     * @param threadFactory the factory to use when creating new threads
     * @return the newly created thread pool
     * @throws NullPointerException if threadFactory is null
     * @throws IllegalArgumentException if {@code nThreads <= 0}
     */
    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```
注意：上述代码中，线程数量是有限制了，但是队列数量无限制，而根据以下代码：
```java
    /**
     * Creates a {@code LinkedBlockingQueue} with a capacity of
     * {@link Integer#MAX_VALUE}.
     */
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }
```
可能队列里堆积大量请求导致OOM。

#### 3. `SingleThreadPool` 只有一个线程的线程池。

这个只贴一个方法，重载方法没贴。

```java
    /**
     * Creates an Executor that uses a single worker thread operating
     * off an unbounded queue. (Note however that if this single
     * thread terminates due to a failure during execution prior to
     * shutdown, a new one will take its place if needed to execute
     * subsequent tasks.)  Tasks are guaranteed to execute
     * sequentially, and no more than one task will be active at any
     * given time. Unlike the otherwise equivalent
     * {@code newFixedThreadPool(1)} the returned executor is
     * guaranteed not to be reconfigurable to use additional threads.
     *
     * @return the newly created single-threaded Executor
     */
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```
注意：与2有同样问题。

#### 4. `ScheduledThreadPool` 按照固定频率执行的线程池。

```java
   /**
     * Creates a thread pool that can schedule commands to run after a
     * given delay, or to execute periodically.
     * @param corePoolSize the number of threads to keep in the pool,
     * even if they are idle
     * @return a newly created scheduled thread pool
     * @throws IllegalArgumentException if {@code corePoolSize < 0}
     */
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }


    /**
     * Creates a thread pool that can schedule commands to run after a
     * given delay, or to execute periodically.
     * @param corePoolSize the number of threads to keep in the pool,
     * even if they are idle
     * @param threadFactory the factory to use when the executor
     * creates a new thread
     * @return a newly created scheduled thread pool
     * @throws IllegalArgumentException if {@code corePoolSize < 0}
     * @throws NullPointerException if threadFactory is null
     */
    public static ScheduledExecutorService newScheduledThreadPool(
            int corePoolSize, ThreadFactory threadFactory) {
        return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
    }
```
注意：有上述所有问题。

*引用自：阿里巴巴Java开发手册：*
>强制】线程池不允许使用 `Executors` 去创建，而是通过 T`hreadPoolExecutor` 的方式，
这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。

## 使用 `new ThreadPoolExecutor`创建线程池

一般使用 `new ThreadPoolExecutor`方式，`Executor`的几个静态方法实现也是
`new ThreadPoolExecutor`的方式，不过是参数值不一样。
`ThreadPoolExecutor`构造方法如下：
```java
/**
     * Creates a new {@code ThreadPoolExecutor} with the given initial
     * parameters.
     *
     * @param corePoolSize the number of threads to keep in the pool, even
     *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
     * @param maximumPoolSize the maximum number of threads to allow in the
     *        pool
     * @param keepAliveTime when the number of threads is greater than
     *        the core, this is the maximum time that excess idle threads
     *        will wait for new tasks before terminating.
     * @param unit the time unit for the {@code keepAliveTime} argument
     * @param workQueue the queue to use for holding tasks before they are
     *        executed.  This queue will hold only the {@code Runnable}
     *        tasks submitted by the {@code execute} method.
     * @param threadFactory the factory to use when the executor
     *        creates a new thread
     * @param handler the handler to use when execution is blocked
     *        because the thread bounds and queue capacities are reached
     * @throws IllegalArgumentException if one of the following holds:<br>
     *         {@code corePoolSize < 0}<br>
     *         {@code keepAliveTime < 0}<br>
     *         {@code maximumPoolSize <= 0}<br>
     *         {@code maximumPoolSize < corePoolSize}
     * @throws NullPointerException if {@code workQueue}
     *         or {@code threadFactory} or {@code handler} is null
     */
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

1. `corePoolSize` - 核心线程数。
2. `maximumPoolSize` - 池中允许的最大线程数。核心 + 非核心
3. `keepAliveTime` - 当线程数大于核心线程数时，为空闲线程闲置时间。如果设置`allowCoreThreadTimeOut = true`，核心线程也会受此值影响。
4. `unit` - `keepAliveTime` 参数的时间单位。
5. `workQueue` - 执行前用于保持任务的队列。此队列仅保持由 `execute` 方法提交的`Runnable` 任务。
常用的`workQueue`类型：`SynchronousQueue`、`LinkedBlockingQueue`、`ArrayBlockingQueue`、`DelayQueue`。
    - `SynchronousQueue`
    - `LinkedBlockingQueue`
    - `ArrayBlockingQueue`
    - `DelayQueue`
6. `threadFactory` - 执行程序创建新线程时使用的工厂。
7. `handler` - 由于超出线程范围和队列容量而使执行被阻塞时所使用的处理程序。
    - `CallerRunsPolicy`  由调用线程来执行
    - `AbortPolicy`  默认的拒绝策略，直接抛出异常
    - `DiscardPolicy`  丢弃任务，不抛出异常
    ![丢掉](https://upload-images.jianshu.io/upload_images/13572633-c4b8e1d865ee80a8.gif?imageMogr2/auto-orient/strip)
    - `DiscardOldestPolicy`  丢弃队列前端任务，将新任务加到队列中。

使用`execute`执行`task`时，当线程数未到达核心线程数时，新建核心线程；当到达核心线程数量时新的task放到队列里；当队列满时候，新建非核心线程；当线程总数到达`maximumPoolSize` ，执行丢弃策略。

### 线程池配置 [转载自文章底部链接]

1.  DB连接池配置
- 最小连接数=（平均QPS* QPS平均RT +平均TPS* TPS平均RT）／业务机器数
- 最大连接数=（峰值QPS* QPS平均RT +峰值TPS* TPS平均RT）／业务机器数；
如果业务代码中没有另起多线程，那么也可以使用公式：最大连接数= 容器处理请求的线程池大小。

另外，如果需要，也可以在此公式的基础上预留一些buffer。

2.  Java线程池配置
**2.1    CPU密集型任务的线程池配置**
因为cpu执行速度非常快，往往切换线程上下文环境所耗费的时间比执行代码花费的时间更长，所以CPU密集型任务的线程数应该尽量保持和CPU核数一致，以减少线程切换带来的性能损耗。
**2.2   IO密集型任务的线程池配置**
IO密集型任务的主要性能瓶颈在于等待IO结果，当遇到任务中存在从DB中读取数据，从缓存中读取数据时，就可以认为是IO密集型任务。一般而言业务系统配置线程池基本上都会采用此模型配置

参考：[云栖社区：ThreadPoolExecutor](https://yq.aliyun.com/articles/592272)
