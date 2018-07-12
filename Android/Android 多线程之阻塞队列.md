#### 阻塞队列
> - 阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里取元素的线程。阻塞队列就是生产者存放元素的容器，而消费者也只从容器中取元素
- 阻塞场景
    * 当队列中没有数据的情况下，消费者端的所有线程都会被自动阻塞，直到有数据放入队列
    * 当队列中填满数据的情况下，生产者端的所有线程都会被自动阻塞，知道队列中有空的位置
- Java中提供了7种阻塞队列，分别是ArrayBlockingQueue、LinkedBlockingQueue、PriorityBlockingQueue、DelayQueue、SynchronousQueue、LinkedTransferQueue、LinkedBlockingDeque，它们都实现了BlockingQueue接口
#### BlockingQueue接口
- BlockingQueue接口提供了一些阻塞队列的通用方法，如offer，poll方法等，下面简单介绍几个方法
- offer(E var1)：表示将var1添加到BlockingQueue，如果添加成功返回true,否则返回false。本方法不阻塞当前执行方法的线程
- offer(E var1, long var2, TimeUnit var4)：可以设定等待的时间，如果在指定的时间内还不能往队列里添加，则返回失败
- put(E var1)：将var1添加到BlockingQueue，如果BlockingQueue没有空间了，那调用此方法的线程会被阻塞，直到BlockingQueue里面有空间再继续添加
- poll(long var1, TimeUnit var3)：从BlockingQueue首位取出数据，如果在指定的时间内，队列一旦有数据可取，就立即返回队列中的数据，否则超时返回null
- take()： 取走BlockingQueue里的排在首位的元素，如果BlockingQueue为空，则阻塞进入等待状态，直到BlockingQueue有新的数据加入
- drainTo：一次性从BlockingQueue获取所有可用的对象，还可以指定获取数据的个数。通过该方法可以提升获取数据的效率，无须多次分批加锁或释放锁
#### 阻塞队列的实现原理
##### ArrayBlockingQueue

- ArrayBlockingQueue是一个用数组实现的有界阻塞队列，通过全局独占锁来实现出队和入队操作，同时只能有一个线程进行入队或出队操作
- ArrayBlockingQueue的offer、poll通过简单的加锁进行入队出队操作，并且不会阻塞线程；而put、take则通过重入锁的条件对象实现队列满则等待、队列空则等待，会阻塞当前线程
- ArrayBlockingQueue能通过size方法获取准确的队列元素个数

##### LinkedBlockingQueue

- LinkedBlockingQueue是一个基于链表的队列，并且是一个先进先出的队列。
- LinkedBlockingQueue如果不指定队列的容量，默认容量大小为Integer.MAX_VALUE，有可能造成占用内存过大的情况
- LinkedBlockingQueue内部对入队和出队操作采用了不同的锁，这样入队和出队操作可以并发进行。但同时只能有一个线程可以进行入队或出队操作。
- LinkedBlockingQueue内部采用的是可重入独占的非公平锁，并且通过重入锁的条件变量来进行出队和入队的同步
- LinkedBlockingQueue通过操作原子变量count来获取当前队列的元素个数

##### SynchronousQueue
- SynchronousQueue本身没有容量存储元素，但是它是通过管理提交操作的线程队列来实现阻塞队列的
- SynchronousQueue可以实现控制线程先进先出进行排序，也就是先被挂起的线程先被唤醒，这个内部是通过链表来实现的。SynchronousQueue默认是不保证证唤醒的顺序的
- SynchronousQueue的不带超时时间的offer和poll方法不会挂起线程，而take和put方法可能会挂起线程。
- SynchronousQueue一个典型的应用场景是线程池newCachedThreadPool，如果入队操作和出队操作的处理速度相差比较大的话有可能会创建大量线程，有耗尽内存的风险

##### DelayQueue
- DelayQueue是基于优先级PriorityQueue实现的，而PriorityQueue的默认构造方法设置容量为11，所以DelayQueue是有界的
- DelayQueue中的元素都必须实现Delayed接口的getDelay方法，以便可以定时执行任务
- DelayQueue中的元素不一定会按照添加的顺序，而是根据元素的优先级排序，元素可以通过实现Comparable接口来定制排列的顺序
- DelayQueue的add、put、offer和poll方法不会挂起线程，而take和带有超时时间的poll方法可能会挂起当前线程
- DelayQueue通过全局独占锁来实现同步，这意味着同时只能有一个入队或是出队操作