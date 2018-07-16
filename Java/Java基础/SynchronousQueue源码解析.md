
```
public class SynchronousQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {
    
    //Transferer是一个抽象类，SynchronousQueue内部有2个Transferer的子类，分别是TransferQueue和TransferStack
    //
    private transient volatile Transferer<E> transferer;

    //默认构造方法的线程等待队列是不保证顺序的
    public SynchronousQueue() {
        this(false);
    }

    //如果fair为true，那SynchronousQueue所采用的是能保证先进先出的TransferQueue，也就是先被挂起的线程会先返回
    public SynchronousQueue(boolean fair) {
        transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
    }
    
    //向SynchronousQueue中添加数据，如果此时线程队列中没有获取数据的线程的话，当前的线程就会挂起等待
    public void put(E e) throws InterruptedException {
        //添加的数据不能是null
        if (e == null) throw new NullPointerException();
        //可以看到添加的方法调用的是transfer方法，如果添加失败会抛出InterruptedException异常
        //后面我们可以在transfer方法的源码中调用put方法添加数据在当前线程被中断时才会返回null
        //这里相当于继续把线程中断的InterruptedException向上抛出
        if (transferer.transfer(e, false, 0) == null) {
            Thread.interrupted();
            throw new InterruptedException();
        }
    }
    
    //不带超时时间的offer方法，如果此时没有线程正在等待获取数据的话transfer就会返回null,也就是添加数据失败
    public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        return transferer.transfer(e, true, 0) != null;
    }
    
    //带超时时间的offer方法，与上面的不同的是这个方法会等待一个超时时间，如果时间过了还没有线程来获取数据就会返回失败
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {
        if (e == null) throw new NullPointerException();
        //添加的数据被其他线程成功获取，返回成功
        if (transferer.transfer(e, true, unit.toNanos(timeout)) != null)
            return true;
        //如果添加数据失败了，有可能是线程被中断了，不是的话直接返回false
        if (!Thread.interrupted())
            return false;
        //是线程被中断的话就向上跑出InterruptedException异常
        throw new InterruptedException();
    }
    
    //take方法用于从队列中取数据，如果此时没有添加数据的线程被挂起，那当前线程就会被挂起等待
    public E take() throws InterruptedException {
        E e = transferer.transfer(null, false, 0);
        //成功获取数据
        if (e != null)
            return e;
        //没有获取到数据，同时又退出挂起状态了，那说明线程被中断了，向上抛出InterruptedException
        Thread.interrupted();
        throw new InterruptedException();
    }
    
    //poll方法同样用于获取数据
    public E poll() {
        return transferer.transfer(null, true, 0);
    }
    
    //带超时时间的poll方法，如果超时时间到了还没有线程插入数据，就会返回失败
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        E e = transferer.transfer(null, true, unit.toNanos(timeout));
        //返回结果有2种情况
        //e != null表示成功取到数据了
        //!Thread.interrupted()表示返回失败了，且是因为超时失败的，此时e是null
        if (e != null || !Thread.interrupted())
            return e;
        //返回失败了，并且是因为当前线程被中断了
        throw new InterruptedException();
    }
    
    //可以看到SynchronousQueue的isEmpty方法一直返回的是true，因为SynchronousQueue没有任何容量
    public boolean isEmpty() {
        return true;
    }

    //同样的size方法也返回0
    public int size() {
        return 0;
    }
    
    <!--下面我们看看TransferQueue的具体实现，TransferQueue中的关键方法就是transfer方法了-->
    
    //先看看TransferQueue的父类Transferer，比较简单，就是提供了一个transfer方法，需要子类具体实现
    abstract static class Transferer<E> {
        abstract E transfer(E e, boolean timed, long nanos);
    }
    
    //TransferQueue
    static final class TransferQueue<E> extends Transferer<E> {
    
        //内部的节点类，用于表示一个请求
        //这里可以看出TransferQueue内部是一个单链表，因此可以保证先进先出
        static final class QNode {
            volatile QNode next;          // next node in queue
            volatile Object item;         // CAS'ed to or from null
            //请求所在的线程
            volatile Thread waiter;       // to control park/unpark
            //用于判断是入队还是出队，true表示的是入队操作，也就是添加数据
            final boolean isData;

            QNode(Object item, boolean isData) {
                this.item = item;
                this.isData = isData;
            }

            //可以看到QNode内部通过volatile关键字以及Unsafe类的CAS方法来实现线程安全
            //compareAndSwapObject方法第一个参数表示需要改变的对象，第二个参数表示偏移量
            //第三个参数表示参数期待的值，第四个参数表示更新后的值
            //下面的方法调用的意思是将当前的QNode对象(this)的next字段赋值为val，当目前的next的值是cmp时就会更新next字段成功
            boolean casNext(QNode cmp, QNode val) {
                return next == cmp &&
                    U.compareAndSwapObject(this, NEXT, cmp, val);
            }

            //方法的原理同上面的类似，这里就是更新item的值了
            boolean casItem(Object cmp, Object val) {
                return item == cmp &&
                    U.compareAndSwapObject(this, ITEM, cmp, val);
            }

            //方法的原理同上面的类似，这里把item赋值为自己，就表示取消当前节点表示的操作了
            void tryCancel(Object cmp) {
                U.compareAndSwapObject(this, ITEM, cmp, this);
            }

            //调用tryCancel方法后item就会是this，就表示当前任务被取消了
            boolean isCancelled() {
                return item == this;
            }

            //表示当前任务已经被返回了
            boolean isOffList() {
                return next == this;
            }

            // Unsafe mechanics
            private static final sun.misc.Unsafe U = sun.misc.Unsafe.getUnsafe();
            private static final long ITEM;
            private static final long NEXT;

            static {
                try {
                    ITEM = U.objectFieldOffset
                        (QNode.class.getDeclaredField("item"));
                    NEXT = U.objectFieldOffset
                        (QNode.class.getDeclaredField("next"));
                } catch (ReflectiveOperationException e) {
                    throw new Error(e);
                }
            }
        }
        
        //首节点
        transient volatile QNode head;
        //尾部节点
        transient volatile QNode tail;
        /**
         * Reference to a cancelled node that might not yet have been
         * unlinked from queue because it was the last inserted node
         * when it was cancelled.
         */
        transient volatile QNode cleanMe;

        //构造函数中会初始化一个出队的节点，并且首尾都指向这个节点
        TransferQueue() {
            QNode h = new QNode(null, false); // initialize to dummy node.
            head = h;
            tail = h;
        }
        
        //transfer方法用于提交数据或者是获取数据
        @SuppressWarnings("unchecked")
        E transfer(E e, boolean timed, long nanos) {
            QNode s = null; // constructed/reused as needed
            //如果e不为null，就说明是添加数据的入队操作
            boolean isData = (e != null);

            for (;;) {
                QNode t = tail;
                QNode h = head;
                if (t == null || h == null)         // saw uninitialized value
                    continue;                       // spin

                //当队列为空的时候或者新加的操作和队尾的操作是同一个操作，可能都是入队操作也可能是出队操作，说明当前没有反向操作的线程空闲
                if (h == t || t.isData == isData) { // empty or same-mode
                    QNode tn = t.next;
                    //这是一个检查，确保t指向队尾
                    if (t != tail)                  // inconsistent read
                        continue;
                    //tn不为null，说明t不是尾部节点，就执行advanceTail操作，将tn作为尾部节点，继续循环
                    if (tn != null) {               // lagging tail
                        advanceTail(t, tn);
                        continue;
                    }
                    //如果timed为true，表示带有超时参数，等待超时期间没有其他相反操作的线程提交就会直接返回null
                    //这里如果nanos初始值就是0，比如不带超时时间的offer和poll方法，当队尾的节点不是相反操作时就会直接返回null
                    if (timed && nanos <= 0L)       // can't wait
                        return null;
                    //如果没有超时时间或者超时时间不为0的话就创建新的节点
                    if (s == null)
                        s = new QNode(e, isData);
                    //使tail的next指向新的节点
                    if (!t.casNext(null, s))        // failed to link in
                        continue;
                    //更新TransferQueue的tail指向新的节点，这样tail节点就始终是尾部节点
                    advanceTail(t, s);              // swing tail and wait
                    //如果当前操作是带超时时间的，则进行超时等待，否则就挂起线程，直到有新的反向操作提交
                    Object x = awaitFulfill(s, e, timed, nanos);
                    //当挂起的线程被中断或是超时时间已经过了，awaitFulfill方法就会返回当前节点，这样就会有x == s为true
                    if (x == s) {                   // wait was cancelled
                        //将队尾节点移出，并重新更新尾部节点，返回null，就是入队或是出队操作失败了
                        clean(t, s);
                        return null;
                    }
                    
                    //如果s还没有被
                    if (!s.isOffList()) {           // not already unlinked
                        advanceHead(t, s);          // unlink if head
                        if (x != null)              // and forget fields
                            s.item = s;
                        s.waiter = null;
                    }
                    return (x != null) ? (E)x : e;

                } 
                //提交操作的时候刚刚好有反向的操作在等待
                else {                            // complementary-mode
                    QNode m = h.next;               // node to fulfill
                    if (t != tail || m == null || h != head)
                        continue;                   // inconsistent read

                    Object x = m.item;
                    //这里先判断m是否是有效的操作
                    if (isData == (x != null) ||    // m already fulfilled
                        x == m ||                   // m cancelled
                        !m.casItem(x, e)) {         // lost CAS
                        advanceHead(h, m);          // dequeue and retry
                        continue;
                    }
                    
                    //更新头部节点
                    advanceHead(h, m);              // successfully fulfilled
                    //唤醒m节点的被挂起的线程
                    LockSupport.unpark(m.waiter);
                    //返回的结果用于给对应的操作，如take、offer等判断是否执行操作成功
                    return (x != null) ? (E)x : e;
                }
            }
        }
        
        <!--下面看看执行挂起线程的方法awaitFulfill-->
        Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
            /* Same idea as TransferStack.awaitFulfill */
            //首先获取超时时间
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            //当前操作所在的线程
            Thread w = Thread.currentThread();
            //线程被挂起或是进入超时等待之前阻止自旋的次数
            int spins = (head.next == s)
                ? (timed ? MAX_TIMED_SPINS : MAX_UNTIMED_SPINS)
                : 0;
            for (;;) {
                //这里首先判断线程是否被中断了，如果被中断了就取消等待，并设置s的item指向s本身作为标记
                if (w.isInterrupted())
                    s.tryCancel(e);
                Object x = s.item;
                //x != e就表示超时时间到了或是线程被中断了，也就是执行了tryCancel方法
                if (x != e)
                    return x;
                //这里先判断超时的时间是否过了
                if (timed) {
                    nanos = deadline - System.nanoTime();
                    if (nanos <= 0L) {
                        s.tryCancel(e);
                        continue;
                    }
                }
                //这里通过多几次循环来避免直接挂起线程
                if (spins > 0)
                    --spins;
                else if (s.waiter == null)
                    s.waiter = w;
                else if (!timed)
                    //park操作会让线程挂起进入等待状态(Waiting)，需要其他线程调用unpark方法唤醒
                    LockSupport.park(this);
                else if (nanos > SPIN_FOR_TIMEOUT_THRESHOLD)
                    //parkNanos操作会让线程挂起进入限期等待(Timed Waiting)，不用其他线程唤醒，时间到了会被系统唤醒
                    LockSupport.parkNanos(this, nanos);
            }
        }
        
    }
}
```
- 通过上述源码我们可以看出SynchronousQueue本身没有容量存储元素，但是它是通过管理提交操作的线程队列来实现阻塞队列的
- SynchronousQueue可以实现控制线程先进先出进行排序，也就是先被挂起的线程先被唤醒，这个内部是通过链表来实现的。SynchronousQueue默认是不保证证唤醒的顺序的
- SynchronousQueue的不带超时时间的offer和poll方法不会挂起线程，而take和put方法可能会挂起线程。
- SynchronousQueue一个典型的应用场景是线程池newCachedThreadPool，从上面的源码可以看出，如果入队操作和出队操作的处理速度相差比较大的话有可能会创建大量线程，有耗尽内存的风险