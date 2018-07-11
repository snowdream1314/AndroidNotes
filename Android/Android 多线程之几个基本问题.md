#### Android中的进程和线程
> - Android中的一个应用程序一般就对应着一个进程，多进程的情况可以参考[Android 多进程通信之几个基本问题](https://www.jianshu.com/p/3c974976abdc)
> - Android中更常见的是多线程的情况，一个应用程序中一般都有包括UI线程等多个线程。Android中规定网络访问必须在子线程中进行，而操作更新UI则只能在UI线程。
> - 常见的网络请求库，如OkHttp、Volly等都为我们封装好了线程池，所以我们在进行网络请求时一般不是很能直观地感受到创建线程以及切换线程的过程。
> - 线程是一种很宝贵的资源，要避免频繁创建销毁线程，一般推荐用线程池来管理线程。

#### 线程的状态
> 线程可能存在6种不同的状态：新创建(New)、可运行(Runnable)、阻塞状态(Blocked)、等待状态(Waiting)、限期等待(Timed Waiting)、终止状态(Terminated)
- 新创建(New)：创建后但还未启动的线程(还没有调用start方法)处于这种状态
- 可运行(Runnable)：一旦调用了start方法，线程就处于这种状态。需要注意的是此时线程可能正在执行，也可能在等待CPU分配执行的时间
- 阻塞状态(Blocked)：表示线程被锁阻塞，等待获取到一个排他锁。在程序等待进入同步区域时，线程将进入这种状态
- 等待状态(Waiting)：处于这种状态的线程不会被分配CPU执行时间，它们要等待被其他线程显示地唤醒。调用以下方法会让线程进入这种状态：
    * 没有设置Timeout参数的Object.wait()方法
    * 没有设置Timeout参数的Thread.join()方法
- 限期等待(Timed Waiting)：与等待状态(Waiting)不同的是，处于这种状态的线程不需要等待其它线程唤醒，在一定时间之后会由系统唤醒。调用以下方法会让线程进入这种状态：
    * Thread.sleep()方法
    * 设置了Timeout参数的Object.wait()方法
    * 设置了Timeout参数的Thread.join()方法 
- 终止状态(Terminated)：表示线程已经执行完毕。导致线程终止有2种情况：
    * 线程的run方法执行完毕，正常退出
    * 因为一个没有捕获的异常而终止了run方法

#### 创建线程
> 创建线程一般有如下几种方式：继承Thread类；实现Runnable接口；实现Callable接口
- 继承Thread类，重写run方法

```
public class TestThread extends Thread {
    @Override
    public void run() {
        System.out.println("Hello World");
    }

    public static void main(String[] args) {
        Thread mThread = new TestThread();
        mThread.start();
    }
}
```

- 实现Runnable接口，并实现run方法

```
public class TestRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Hello World");
    }

    public static void main(String[] args) {
        TestRunnable mTestRunnable = new TestRunnable();
        Thread mThread = new Thread(mTestRunnable);
        mThread.start();
    }
}
```

- 实现Callable接口，重写call方法
    * Callable可以在任务接受后提供一个返回值而Runnable不行
    * Callable的call方法可以抛出异常，Runnable的run方法不行
    * 运行Callable可以拿到一个Future对象，表示计算的结果，通过Future的get方法可以拿到异步计算的结果，不过当前线程会阻塞。

```
public class TestCallable {

    public static class MyTestCallable implements Callable<String> {

        @Override
        public String call() throws Exception {
            //call方法可以提供返回值，而Runnable不行
            return "Hello World";
        }
    }

    public static void main(String[] args) {
        MyTestCallable myTestCallable = new MyTestCallable();
        //手动创建线程池
        ExecutorService executorService = new ThreadPoolExecutor(1,1,0L, TimeUnit.SECONDS, new LinkedBlockingQueue<Runnable>(10));
        //运行callable可以拿到一个Future对象
        Future future = executorService.submit(myTestCallable);
        try {
            //等待线程结束，future.get()方法会使当前线程阻塞
            System.out.println(future.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

    }
}
```

- 以上三种方式就是常见的创建线程的方式。推荐使用实现Runnable接口的方法。

#### 线程中断
- 当一个线程调用interrupt方法时，线程的中断标识为将被设置成true
- 通过Thread.currentThread().isInterrupted()方法可以判断线程是否应该被中断
- 可以通过调用Thread.interrupted()对中断标志位进行复位(设置为false)
- 如果一个线程处于阻塞状态，线程在检查中断标志位时如果发现中断标志位为true，则会在阻塞方法处抛出InterruptedException异常，并且在抛出异常前会将中断标志位复位，即重新设置为false
- 不要在代码底层捕获InterruptedException异常后不做处理
#### 同步的几种方法
> 同步的方式一般有如下3种:volatile关键字、synchronized关键字、重入锁ReentrantLock

##### volatile关键字
- volatile关键字实现多线程安全关键在于它的可见性特性，但它需要满足一些条件才能保证线程安全，具体可以查看文章[深入理解Java虚拟机(八)之Java内存模型](https://www.jianshu.com/p/bf4523b8a1d0)
- 在用volatile关键来实现多线程安全时需要注意volatile不保证原子性，也就是不能用于一些自增、自减等操作，也不能用于一些不变式中，自增、自减比较好理解，下面看看不变式的情况

```
public class VolatileTest {
    private volatile int lower,upper;

    public int getLower() {
        return lower;
    }

    public void setLower(int value) {
        if (value > upper) {
            throw new IllegalArgumentException();
        }
        this.lower = value;
    }

    public int getUpper() {
        return upper;
    }

    public void setUpper(int value) {
        if (value < lower) {
            throw new IllegalArgumentException();
        }
        this.upper = value;
    }
}
```
- 上面的例子中，如果初始值是(0,5)，线程A调用setLower(4)，线程B调用setUpper(3)，显然最后结果就会变成(4,3)了
- volatile使用的场景常见的有作为状态标志以及DCL单例模式
##### synchronized关键字和重入锁ReentrantLock
- synchronized关键字比较常见，可以用于同步方法也可以用于同步代码块，一般推荐用同步方法，同步代码块的安全性不高。
- 重入锁ReentrantLock相比synchronized提供了一些独有的特性：可以绑定多个解锁的条件Condition、可以实现公平锁、可以设置放弃等待获取锁的时间。

```
public class ReentrantLockTest {
    private Lock mLock = new ReentrantLock();
    //true，表示实现公平锁
    <!--private Lock mLock = new ReentrantLock(true);-->
    private Condition condition;

    private void thread1() throws InterruptedException{
        mLock.lock();
        try {
            condition = mLock.newCondition();
            condition.await();
            System.out.println("thread1：Hello World");
        }finally {
            mLock.unlock();
        }
    }

    private void thread2() throws InterruptedException{
        mLock.lock();
        try {
            System.out.println("thread2：Hello World");
            condition.signalAll();
        }finally {
            mLock.unlock();
        }
    }
}
```
- 一个ReentrantLock有多个相关的Condition,调用Condition的await方法会让当前线程进入该条件的等待集并阻塞，直到另一个线程调用了同一个条件的signalAll方法激活因为这个条件而进入阻塞的所有线程

- 一般线程同步用得比较多的还是synchronized同步方法和一些java.util.concurrent包提供的一些类