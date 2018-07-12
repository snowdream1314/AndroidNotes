
```
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {
    //可以看到DelayQueue中的元素必须是Delayed的子类，必须实现getDelay方法
    
    //通过重入锁实现同步
    private final transient ReentrantLock lock = new ReentrantLock();
    //元素存储在一个优先级队列PriorityQueue中，PriorityQueue是基于数组实现的
    private final PriorityQueue<E> q = new PriorityQueue<E>();
    
    private Thread leader;

    //条件变量
    private final Condition available = lock.newCondition();
    
    //默认构造函数啥也不干
    public DelayQueue() {}
    
    public boolean add(E e) {
        return offer(e);
    }
    
    public void put(E e) {
        offer(e);
    }
    
    public boolean offer(E e, long timeout, TimeUnit unit) {
        return offer(e);
    }
    
    //上面几个方法最后调用的都是这个offer方法，offer方法不会挂起线程，所以上面的几个方法都不会挂起线程
    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        //获取全局独占锁
        lock.lock();
        try {
            //将元素存入PriorityQueue，PriorityQueue中会按照优先级排序，刚刚放入的元素并不一定就在队列的第一个位置
            //这里e也不能为null
            q.offer(e);
            //q.peek()获取的是队列的第一个元素
            //如果刚刚放进去的元素就在第一个位置，唤醒正在等待的线程
            if (q.peek() == e) {
                leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }
    
    //poll方法也不会挂起线程
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            E first = q.peek();
            //队列的第一个元素为null或者延时时间还没到，就说明现在队列里还没有需要执行的任务，返回null；否则直接调用q.poll()方法获取队列的第一个元素
            //这里为什么不直接返回first呢？因为q.poll()方法除了返回并删除队列的第一个元素，还会重新移动队列；而peek方法只是简单返回第一个元素而已，是用于判断队列是否有符合条件的元素的
            return (first == null || first.getDelay(NANOSECONDS) > 0)
                ? null
                : q.poll();
        } finally {
            lock.unlock();
        }
    }
    
    //take方法可能会挂起当前线程
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        //获取的是可中断的锁，如果已经被中断了就会抛出InterruptedException异常，而不需要等到调用await方法才抛异常了
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                //当前队列为空，挂起当前线程，进入available条件的等待队列
                if (first == null)
                    available.await();
                else {
                    //队列不为空的话，就看首个元素是否到了执行的时间
                    long delay = first.getDelay(NANOSECONDS);
                    //如果时间到了，就直接通过q.poll()返回
                    if (delay <= 0L)
                        return q.poll();
                    //执行到这里，说明队列里没有满足条件的元素，那就清空first的引用
                    first = null; // don't retain ref while waiting
                    //有线程的话就挂起等待，用await挂起需要其他线程调用signal唤醒
                    if (leader != null)
                        available.await();
                    else {
                        //将当前线程挂起
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            //将当前线程挂起，调用awaitNanos挂起，等待delay时间后会被系统唤醒
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
    
    //带超时时间的poll也会挂起线程
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        //获取可中断的锁，原因同上
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                //如果队列为空
                if (first == null) {
                    //时间也到了，那只能返回null了，获取数据失败
                    if (nanos <= 0L)
                        return null;
                    //时间没到就挂起等待下
                    else
                        nanos = available.awaitNanos(nanos);
                } else {//队列不为空
                    long delay = first.getDelay(NANOSECONDS);
                    //队列首位元素时间到了就调用q.poll()返回元素
                    if (delay <= 0L)
                        return q.poll();
                    //poll方法超时时间到了，但还没有符合的元素，就返回null，获取数据失败
                    if (nanos <= 0L)
                        return null;
                    //后面基本和上面take方法类似
                    first = null; // don't retain ref while waiting
                    if (nanos < delay || leader != null)
                        nanos = available.awaitNanos(nanos);
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            long timeLeft = available.awaitNanos(delay);
                            nanos -= delay - timeLeft;
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
    
    //DelayQueue的peek方法调用的是PriorityQueue的peek方法
    public E peek() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //如果队列不为空则返回队列首位元素，但不移动队列的元素也不删除元素
            return q.peek();
        } finally {
            lock.unlock();
        }
    }

    //获取队列的元素数量
    public int size() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return q.size();
        } finally {
            lock.unlock();
        }
    }
}
```

- 从上面的源码我们可以看出DelayQueue是基于优先级PriorityQueue实现的，而PriorityQueue的默认构造方法设置容量为11，所以DelayQueue是有界的
- DelayQueue中的元素都必须实现Delayed接口的getDelay方法，以便可以定时执行任务
- DelayQueue中的元素不一定会按照添加的顺序，而是根据元素的优先级排序，元素可以通过实现Comparable接口来定制排列的顺序
- DelayQueue的add、put、offer和poll方法不会挂起线程，而take和带有超时时间的poll方法可能会挂起当前线程
- DelayQueue通过全局独占锁来实现同步，这意味着同时只能有一个入队或是出队操作