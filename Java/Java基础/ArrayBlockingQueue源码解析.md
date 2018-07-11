```
public class ArrayBlockingQueue<E> extends AbstractQueue<E> implements BlockingQueue<E>, Serializable {
    private static final long serialVersionUID = -817911632652898426L;
    //可以看到ArrayBlockingQueue是通过数组来实现的队列效果
    final Object[] items;
    //记录队首元素的下标
    int takeIndex;
    //记录队尾元素的下标
    int putIndex;
    //记录队列中的元素个数
    int count;
    //通过ReentrantLock来实现同步
    final ReentrantLock lock;
    //有2个条件对象，分别表示队列不为空和队列不满的情况
    private final Condition notEmpty;
    private final Condition notFull;
    transient Itrs itrs;
    
    //默认的构造函数必须传入队列大小，所以是有届队列，默认不实现公平锁
    public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }

    //可以通过fair为true来实现公平锁
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
    
    //offer方法用于向队列中添加数据
    public boolean offer(E e) {
        //可以看出添加的数据不支持null值
        Objects.requireNonNull(e);
        final ReentrantLock lock = this.lock;
        //通过重入锁来实现同步
        lock.lock();
        try {
            //这里可以看到，如果队列已经满了的话直接就返回false，不会阻塞调用这个offer方法的线程
            if (count == items.length)
                return false;
            else {
                //如果队列没有满，就调用enqueue方法将元素添加到队列中
                enqueue(e);
                return true;
            }
        } finally {
            lock.unlock();
        }
    }
    
    //这个offer方法跟上面的offer方法最大的不同是多了个等待的时间
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        Objects.requireNonNull(e);
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        //获取可中断锁
        lock.lockInterruptibly();
        try {
            while (count == items.length) {
                //如果等待时间过了队列还是满的话就直接返回false，添加元素失败
                if (nanos <= 0L)
                    return false;
                //等待设置的时间
                nanos = notFull.awaitNanos(nanos);
            }
            //如果等待时间过了，队列有空间的话就会调用enqueue方法将元素添加到队列
            enqueue(e);
            return true;
        } finally {
            lock.unlock();
        }
    }
    
    //put方法和offer方法不一样的地方在于，如果队列是满的话，它就会把调用put方法的线程阻塞，直到队列里有空间
    public void put(E e) throws InterruptedException {
        Objects.requireNonNull(e);
        final ReentrantLock lock = this.lock;
        //因为后面调用了条件变量的await()方法，而await()方法会在中断标志设置后抛出InterruptedException异常后退出，所以还不如在加锁时候先看中断标志是不是被设置了，如果设置了直接抛出InterruptedException异常，就不用再去获取锁了
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                //如果队列满的话就阻塞等待，直到notFull的signal方法被调用，也就是队列里有空间了
                notFull.await();
            //队列里有空间了执行添加操作
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
    
    //poll方法用于从队列中取数据，不会阻塞当前线程
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //这里可以看到如果队列为空的话会直接返回null，否则调用dequeue方法取数据
            return (count == 0) ? null : dequeue();
        } finally {
            lock.unlock();
        }
    }
    
    //这个poll的重载方法也是加了个等待的时间，和上面offer的重载类似
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0) {
                if (nanos <= 0L)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
    
    //take方法也是用于取队列中的数据，但是和poll方法不同的是它有可能会阻塞当前的线程
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            //当队列为空时，就会阻塞当前线程
            while (count == 0)
                notEmpty.await();
            //直到队列中有数据了，调用dequeue方法将数据返回
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
    
    //将数据添加到队列中的具体方法
    private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = x;
        //可以看出是通过循环数组实现的队列，当数组满了时下标就变成0了
        if (++putIndex == items.length) putIndex = 0;
        count++;
        //激活因为notEmpty条件而阻塞的线程，比如上面的调用take方法的线程
        notEmpty.signal();
    }

    //将数据从队列中取出的方法
    private E dequeue() {
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        //将对应的数组下标位置设置为null释放资源
        items[takeIndex] = null;
        if (++takeIndex == items.length) takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        //激活因为notFull条件而阻塞的线程，比如上面的调用put方法的线程
        notFull.signal();
        return x;
    }
    
    //获取队列的元素个数，加了锁，所以结果是准确的
    public int size() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```
- ArrayBlockingQueue是一个用数组实现的有界阻塞队列，通过全局独占锁来实现出队和入队操作，同时只能有一个线程进行入队或出队操作
- ArrayBlockingQueue的offer、poll通过简单的加锁进行入队出队操作，并且不会阻塞线程；而put、take则通过重入锁的条件对象实现队列满则等待、队列空则等待，会阻塞当前线程
- ArrayBlockingQueue能通过size方法获取准确的队列元素个数