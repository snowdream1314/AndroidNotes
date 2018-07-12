#### Android线程池的真正实现是ThreadPoolExecutor

```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
}
```
- corePoolSize：表示核心线程数。默认情况下核心线程会在线程池中一直存活，即使处于闲置状态。如果将ThreadPoolExecutor的allowCoreThreadTimeOut属性设置为true，那核心线程也会有超时策略，时间间隔由keepAliveTime指定
- maximumPoolSize：线程池的最大线程数。当活动的线程数达到这个数值后，后续的新任务就会被阻塞
- keepAliveTime：线程超时策略的超时时间，超过这个时间，非核心线程会被回收，核心线程要看allowCoreThreadTimeOut属性
- unit：keepAliveTime的单位
- workQueue：线程池的任务队列，如LinkedBlockingQueue,SynchronousQueue等
- ThreadFactory：线程工厂，为线程池创建新线程。ThreadFactory是一个接口，只有一个newThread方法

##### 执行过程
- 如果线程池中的线程数未达到核心线程数，就创建核心线程执行新的任务
- 如果线程池中的线程数已经达到核心线程数，就将任务插入到任务队列中
- 如果无法将任务插入队列，说明队列已经满了，这个时候如果线程数未达到线程池的最大线程数，就创建非核心线程执行任务
- 如果任务队列满了，线程数已经达到最大值，就拒绝执行任务。ThreadPoolExecutor会调用RejectedExecutionHandler的rejectedExecution方法通知调用者

#### 常见的四种线程池
##### FixedThreadPool
- 创建方法

```
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
}
```
- 可以看到默认构造函数需要传入核心线程数，且核心线程数和线程总数相同，也就是只有核心线程，同时没有超时时间，任务队列为LinkedBlockingQueue
- 由于只有核心线程，且没有超时策略，FixedThreadPool中的线程会一直存活，除非线程池被关闭。因此能快速响应外界的请求
- 所有的核心线程都处于活动状态时，新的任务都会处于等待状态

##### CachedThreadPool
- 创建方法

```
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
}
```
- CachedThreadPool只有非核心线程，且数量为Integer.MAX_VALUE，超时时间为60秒。
- CachedThreadPool采用的任务队列为SynchronousQueue，因此只要有新的任务来就会立刻被执行。
- 由于只有非核心线程，且有超时策略，当没有任务时，CachedThreadPool实际上一个线程也没有。
- CachedThreadPool比较适合执行大量耗时较少的任务

##### SingleThreadExecutor
- 创建方法

```
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
}
```
- SingleThreadExecutor只有一个核心线程，且没有超时策略，任务队列为LinkedBlockingQueue。
- SingleThreadExecutor确保所有的任务都在同一个线程中按顺序执行，这样这些任务就不需要处理线程同步的问题

##### ScheduledThreadPool
- 创建方法

```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
}

public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
}
```
- ScheduledThreadPool的核心线程数是固定的，必须在创建时指定。非核心线程数是没有限制的。当非核心线程空闲时会被立即回收。
- ScheduledThreadPool的任务队列为DelayedWorkQueue，因此可以用来执行一些定时任务和具有周期重复的任务

#### 总结
- 以上的四种常用的线程池虽然是系统默认提供的，但是我们在使用时还是要仔细选择。
- 比如FixedThreadPool和SingleThreadExecutor中的任务队列为LinkedBlockingQueue，且没有指定队列的默认容量，而LinkedBlockingQueue的默认容量为Integer.MAX_VALUE，因此有可能造成队列任务过多而占用过多内存甚至发生OOM的情况
- ScheduledThreadPool和CachedThreadPool的问题则是线程最大的数量为Integer.MAX_VALUE，可能导致创建大量的线程，引发OOM
- 以上都是需要注意的地方。另外还是推荐用ThreadPoolExecutor根据实际需要手动创建线程池