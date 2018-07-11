
```
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    /**
     * LinkedBlockingQueue内部的节点类
     */
    static class Node<E> {
        E item;

        Node<E> next;

        Node(E x) { item = x; }
    } 
    
    //队列的容量，默认为Integer.MAX_VALUE
    private final int capacity;

    //队列的元素数量，初始值为0
    private final AtomicInteger count = new AtomicInteger();

    //头节点
    transient Node<E> head;

    //尾部节点
    private transient Node<E> last;

    //这个重入锁用于take, poll等获取数据的方法，即用于出队的锁
    private final ReentrantLock takeLock = new ReentrantLock();

    //用于take, poll等获取数据的方法的阻塞条件，即用于入队的锁
    private final Condition notEmpty = takeLock.newCondition();

    //这个重入锁用于put, offer等添加数据的方法
    private final ReentrantLock putLock = new ReentrantLock();

    /用于put, offer等添加数据的方法的阻塞条件
    private final Condition notFull = putLock.newCondition();
    
    //激活调用notEmpty.await()阻塞后放入notEmpty条件队列中的线程
    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }
    
    //激活调用notFull.await()阻塞后放入notEmpty条件队列中的线程
    private void signalNotFull() {
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            notFull.signal();
        } finally {
            putLock.unlock();
        }
    }
    
    //向队列的链表结构的尾部插入数据
    private void enqueue(Node<E> node) {
        // assert putLock.isHeldByCurrentThread();
        // assert last.next == null;
        last = last.next = node;
    }
    
    //取出链表头部的节点数据，并释放资源
    private E dequeue() {
        // assert takeLock.isHeldByCurrentThread();
        // assert head.item == null;
        Node<E> h = head;
        Node<E> first = h.next;
        h.next = h; // help GC
        head = first;
        E x = first.item;
        first.item = null;
        return x;
    }
    
    //可以看到默认的构造函数容量大小为Integer.MAX_VALUE
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }

    
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }
    
    //返回队列中的元素数量，这里直接通过原子变量获取
    public int size() {
        return count.get();
    }
    
    public void put(E e) throws InterruptedException {
        //不允许添加null
        if (e == null) throw new NullPointerException();
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        int c = -1;
        //生成新的节点
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        //获取可中断的锁
        //因为后面调用了条件变量的await()方法，而await()方法会在中断标志设置后抛出InterruptedException异常后退出，所以还不如在加锁时候先看中断标志是不是被设置了，如果设置了直接抛出InterruptedException异常，就不用再去获取锁了
        putLock.lockInterruptibly();
        try {
            /*
             * 下面的原文注释很好的解释了为什么这里count是线程安全的
             * Note that count is used in wait guard even though it is
             * not protected by lock. This works because count can
             * only decrease at this point (all other puts are shut
             * out by lock), and we (or some other waiting put) are
             * signalled if it ever changes from capacity. Similarly
             * for all other uses of count in other wait guards.
             */
            //如果队列已经满了，则阻塞当前线程
            while (count.get() == capacity) {
                notFull.await();
            }
            //调用notFull的signal方法退出阻塞状态后，执行添加数据的操作
            enqueue(node);
            c = count.getAndIncrement();
            //如果添加数据后还队列还没有满，则继续调用notFull的signal方法唤醒其他等待在入队的线程
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        //c=0说明队列里有一个元素，这时候唤醒出队线程
        if (c == 0)
            signalNotEmpty();
    }
    
    //带超时时间的offer方法
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        if (e == null) throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                //如果超时时间过了队列仍然是满的话就直接返回false
                if (nanos <= 0L)
                    return false;
                //否则调用awaitNanos等待，超时会返回<= 0L
                nanos = notFull.awaitNanos(nanos);
            }
            //在超时时间内返回则添加元素
            enqueue(new Node<E>(e));
            c = count.getAndIncrement();
            //如果添加数据后还队列还没有满，则继续调用notFull的signal方法唤醒其他等待在入队的线程
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        //c=0说明队列里有一个元素，这时候唤醒出队线程
        if (c == 0)
            signalNotEmpty();
        return true;
    }

    //不带超时时间的offer方法
    public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        final AtomicInteger count = this.count;
        //如果队列已经满了的话就直接返回false，后面的逻辑基本和上面相同
        if (count.get() == capacity)
            return false;
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            if (count.get() < capacity) {
                enqueue(node);
                c = count.getAndIncrement();
                if (c + 1 < capacity)
                    notFull.signal();
            }
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return c >= 0;
    }
    
    //take方法用于从队列中取数据，队列为空是会阻塞线程
    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        //获取可中断锁
        takeLock.lockInterruptibly();
        try {
            //如果队列为空，则会阻塞线程
            while (count.get() == 0) {
                notEmpty.await();
            }
            x = dequeue();
            c = count.getAndDecrement();
            //如果c > 1说明队列中还有元素，则调用notEmpty的signal方法唤醒出队线程
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        //如果c == capacity就是说队列中有一个空位，唤醒入队线程
        if (c == capacity)
            signalNotFull();
        return x;
    }
    
    //带超时时间的poll方法用于从队列中取数据，不会阻塞当前线程
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        E x = null;
        int c = -1;
        long nanos = unit.toNanos(timeout);
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        //获取可中断锁
        takeLock.lockInterruptibly();
        try {
            //如果队列位空，那就循环等待
            while (count.get() == 0) {
                if (nanos <= 0L)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
            //在超时时间内返回，则调用dequeue获取队列中的数据
            x = dequeue();
            c = count.getAndDecrement();
            //如果c > 1说明队列中还有元素，则调用notEmpty的signal方法唤醒出队线程
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        //如果c == capacity就是说队列中有一个空位，唤醒入队线程
        if (c == capacity)
            signalNotFull();
        return x;
    }
    
    //不带超时时间的poll方法
    public E poll() {
        final AtomicInteger count = this.count;
        //如果队列为空则直接返回null，后面逻辑同上面的超时poll方法类似
        if (count.get() == 0)
            return null;
        E x = null;
        int c = -1;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            if (count.get() > 0) {
                x = dequeue();
                c = count.getAndDecrement();
                if (c > 1)
                    notEmpty.signal();
            }
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
    
    //peek方法不会阻塞当前线程
    public E peek() {
        //如果队列为空，则直接返回null
        if (count.get() == 0)
            return null;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            //如果队列不为空，则直接返回链表头部的节点元素数据
            return (count.get() > 0) ? head.next.item : null;
        } finally {
            takeLock.unlock();
        }
    }
    
    //删除队列中的元素
    public boolean remove(Object o) {
        //如果删除的元素为null，则直接返回false
        if (o == null) return false;
        //双重加锁，即入队和出队同时锁定，这样能保证在删除操作时队列元素保持不变
        fullyLock();
        try {
            //循环遍历链表，删除元素
            for (Node<E> trail = head, p = trail.next;
                 p != null;
                 trail = p, p = p.next) {
                if (o.equals(p.item)) {
                    unlink(p, trail);
                    return true;
                }
            }
            return false;
        } finally {
            fullyUnlock();
        }
    }
    
    //双重加锁
    void fullyLock() {
        putLock.lock();
        takeLock.lock();
    }

    //双重解锁
    void fullyUnlock() {
        takeLock.unlock();
        putLock.unlock();
    }

    ...
}
```

- LinkedBlockingQueue是一个基于链表的队列，并且是一个先进先出的队列。
- LinkedBlockingQueue如果不指定队列的容量，默认容量大小为Integer.MAX_VALUE，有可能造成占用内存过大的情况
- LinkedBlockingQueue内部对入队和出队操作采用了不同的锁，这样入队和出队操作可以并发进行。但同时只能有一个线程可以进行入队或出队操作。
- LinkedBlockingQueue内部采用的是可重入独占的非公平锁，并且通过重入锁的条件变量来进行出队和入队的同步
- LinkedBlockingQueue通过操作原子变量count来获取当前队列的元素个数
