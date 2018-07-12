#### Andorid中的线程除了传统的Thread外，主要还有AsyncTask、HandlerThread、IntentService。
#### AsyncTask
> AsyncTask是一种轻量的异步任务类，不仅可以在后台执行任务，还能把执行的进度和最终的结果传递给UI线程以便更新UI。AsyncTask底层是封装了Thread和Handler。AsyncTask不适合执行特别耗时的任务，这种情况下还是推荐用线程池来处理
- 四个核心方法
    * onPreExecute：在主线程中执行，在异步任务执行之前执行，可以做一些准备工作
    * doInBackground：在线程池中执行，执行具体的异步任务。在这个方法中可以调用publishProgress方法来更新任务的进度
    * onProgressUpdate：在主线程中执行，调用publishProgress方法后会被调用
    * onPostExecute：在主线程中执行，异步任务执行完毕后会被调用，返回执行结果
    * onCancelled：在主线程中执行，异步任务被取消时会被调用，这样onPostExecute方法将不会被调用
    
- AsyncTask必须在主线程中加载，这意味着第一次访问AsyncTask必须发生在主线程。Android4.1及以上的版本中已经在ActivityThread的main方法中自动完成了
- AsyncTask对象必须在主线程中创建
- AsyncTask的execute方法必须在UI线程调用
- 不要在程序中直接调用onPreExecute、doInBackground、onProgressUpdate、onPostExecute方法
- 一个AsyncTask对象只能执行一次，只能调用一次execute方法,否则会报IllegalStateException异常
- 默认情况下AsyncTask是串行执行的

#### HandlerThread

- HandlerThread继承自Thread
```
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;

    //构造方法需要传入线程的名字
    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
    
    //用来重写，在开启消息循环之前被调用
    protected void onLooperPrepared() {
    }

    //主要的地方在这个run方法
    @Override
    public void run() {
        mTid = Process.myTid();
        //创建消息队列
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        //开启消息循环
        Looper.loop();
        mTid = -1;
    }
    
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

    //用于停止消息循环
    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }
    //用于停止消息循环
    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }

    public int getThreadId() {
        return mTid;
    }
```
- HandlerThread与普通的Thread不同的地方在于它在run方法中开启了消息循环，外界需要通过Handler的消息的方式来通知HandlerThread执行一个具体的任务
- 由于HandlerThread的run方法是一个无限循环，因此当不需要再使用HandlerThread时，可以通过quit和quitSafely方法来终止线程的执行

#### IntentService
- IntentService 是一个抽象类，所以必须创建它的子类才能使用IntentService
- IntentService继承自Service，因此是一种特殊的服务，优先级比普通的后台线程要高，不容易被回收，适合执行一些高优先级的后台任务
- IntentService封装了HandlerThread和Handler

```
public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper;
    private volatile ServiceHandler mServiceHandler;
    private String mName;
    private boolean mRedelivery;

    //自定义的Handler，这样可以把收到的任务消息发送给HandlerThread处理
    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            //onHandleIntent处理具体的任务，运行在HandlerThread中
            onHandleIntent((Intent)msg.obj);
            //任务执行完毕会自动结束
            stopSelf(msg.arg1);
        }
    }

    public IntentService(String name) {
        super();
        mName = name;
    }

    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }

    //第一次启动时会调用onCreate
    @Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.

        super.onCreate();
        //创建HandlerThread
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        //调用HandlerThread的run方法，开启HandlerThread的消息循环，以便接收消息处理具体的任务
        thread.start();
        
        //获取HandlerThread的looper，创建ServiceHandler，这样通过ServiceHandler发送的消息最后都会在HandlerThread中处理
        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    /**
     * You should not override this method for your IntentService. Instead,
     * override {@link #onHandleIntent}, which the system calls when the IntentService
     * receives a start request.
     * @see android.app.Service#onStartCommand
     */
    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        //每次调用IntentService都会执行onStartCommand方法，发送一个任务消息给HandlerThread处理，任务的信息通过Intent传递
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }

    /**
     * Unless you provide binding for your service, you don't need to implement this
     * method, because the default implementation returns null.
     * @see android.app.Service#onBind
     */
    @Override
    @Nullable
    public IBinder onBind(Intent intent) {
        return null;
    }

    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);
}
```
- IntentService通过Intent来传递任务参数，当任务执行完毕时会调用stopSelf(int startId)自己停止服务
- IntentService的onHandleIntent是一个抽象方法，需要我们自己实现，用于处理具体的任务。
- 每执行一个后台任务就需要启动一次IntentService，IntentService内部通过消息的方式向HanderThread请求执行任务，Handler的Looper是顺序执行任务的，所以IntentService也是顺序执行后台任务的。

