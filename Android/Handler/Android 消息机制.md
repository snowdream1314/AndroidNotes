#### Android中的消息机制主要是指Handler的运行机制，而Handler的运行又离不开Looper和MessageQueue。
#### MessageQueue
- MessageQueue用来接收Hnadler发送过来的消息Message，其内部是用单链表的数据结构来存储消息列表的。

```
//MessageQueue.class
Message next() {
    ...
    for (;;) {
        ...
        synchronized (this) {
            ...
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }
            ...
        }
        ...
    }
}
```

- MessageQueue在调用Looper的prepare方法时创建。

```
//Looper.class
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    //将Looper存到当前线程的ThreadLocal中
    sThreadLocal.set(new Looper(quitAllowed));
}

private Looper(boolean quitAllowed) {
    //创建MessageQueue
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

- MessageQueue的next方法是一个无限循环，除非调用quit方法才会停止

```
Message next() {
    ...
    for (;;) {
        ...
        synchronized (this) {
            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                //返回null，退出循环
                return null;
            }
        }
        ...
    }
    ...
}

void quit(boolean safe) {
    ...
    synchronized (this) {
        ...
        //这样MessageQueue就退出循环啦
        mQuitting = true;
        ...
    }
}
```

#### Looper

- Looper用于从MessageQueue中取出待处理的消息，会调用MessageQueue的next方法来取消息。如果MessageQueue中没有消息，Looper就会一直阻塞等待，除非MessageQueue返回null才退出消息的循环监听。
- 调用Looper的prepare方法后会创建MessageQueue和Looper。但需要调用Looper的loop方法才会开启消息循环，Looper才会开始循环监听MessageQueue中的消息
- Looper取到消息后，会调用消息Message所对应的Handler的dispatchMessage方法来处理消息。这样Handler的dispatchMessage方法就会在Looper所在的线程中执行了。

```
public static void loop() {
    final Looper me = myLooper();
    //调用loop方法前，必须先通过prepare方法创建looper和消息队列MessageQueue
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    for (;;) {
        //调用MessageQueue的next方法取消息，因为next是个无限循环，所以loop方法也会阻塞
        Message msg = queue.next(); // might block
        //只有在调用MessageQueue的quit方法后MessageQueue才会返回null，才会退出循环监听
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        ...
        
        try {
            //msg.target就是Handler，这里就是调用Handler的dispatchMessage方法来处理消息。
            msg.target.dispatchMessage(msg);
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }

        ...

    }
}
```

#### Handler
- Looper获取到消息后，最后会交给Handler的dispatchMessage方法处理

```
//Handler.class
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```
- 首先看看Message的callbac是否为null，不为null就通过handleCallback方法来处理消息。

```
private static void handleCallback(Message message) {
    message.callback.run();
}
```
- Message的callback是个Runnable对象，是Handler的post方法传递的Runnable参数。

```
public final boolean post(Runnable r)
{
   return  sendMessageDelayed(getPostMessage(r), 0);
}

private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

- 如果callback参数为null，就检查mCallback参数是否为null。不为null就调用mCallback的handleMessage方法处理消息。mCallback是一个Callback接口

```
/**
 * Callback interface you can use when instantiating a Handler to avoid
 * having to implement your own subclass of Handler.
 *
 * @param msg A {@link android.os.Message Message} object
 * @return True if no further handling is desired
 */
public interface Callback {
    public boolean handleMessage(Message msg);
}
```
- 如果我们在使用Handler时不想派生Handler的子类，就可以用Callback的方式

```
private Handler callbackHandler = new Handler(new Handler.Callback() {
    @Override
    public boolean handleMessage(Message msg) {

        return false;
    }
});
```

- 如果mCallback参数也为null，那就调用handleMessage方法处理消息，也就是我们用派生Handler子类的方式创建Handler时重写的handleMessage方法

```
//Handler.class
/**
 * Subclasses must implement this to receive messages.
 */
public void handleMessage(Message msg) {
}
```
- 用派生Handler子类的方式创建Handler
```
private static class MyHandler extends Handler {
    @Override
    public void handleMessage(Message message) {
        switch (message.what) {
            case MESSAGE_TAG:
                Log.d(Thread.currentThread().getName(),"receive message");
                break;
            default:

                break;
        }
    }
}
```

#### Handler为什么能切换线程？

> 一句话总结就是：Handler是线程间共享的一个对象，而Looper则是每个线程私有的。

##### 怎么理解上面的说法呢？

- Handler是共享的意思是，在需要切换的2个线程中都能访问我们创建的Handler对象，比如能调用Handler的sendMessage方法。
- 比如下面的代码中myHandler变量是在HandlerActivity的onCreate方法中创建的，也就是在UI线程中。myHandler即可以在UI线程中调用，也可以在子线程中调用。

```
public class HandlerActivity extends Activity {

    public static final int MESSAGE_TAG = 0x01;
    private MyHandler myHandler;

    //通过派生Handler子类的方式创建我们的Handler
    private static class MyHandler extends Handler {
        @Override
        public void handleMessage(Message message) {
            switch (message.what) {
                case MESSAGE_TAG:
                    Log.d(Thread.currentThread().getName(),"receive message");
                    break;
                default:

                    break;
            }
        }
    }
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //创建Handler
        myHandler = new MyHandler();

        new Thread(new Runnable() {
            @Override
            public void run() {
                Message message = new Message();
                message.what = MESSAGE_TAG;
                //在子线程中调用UI线程创建的myHandler，发送消息
                myHandler.sendMessage(message);
            }
        }).start();
    }
}
```
- 调用Handler发送消息以后怎么就把线程从子线程切换到UI线程了呢？这是因为每一个Handler在创建的时候都需要当前线程有Looper，而每个线程都会有自己的Looper

```
//Handler.class
public Handler() {
    this(null, false);
}

public Handler(Callback callback, boolean async) {
    ...
    //获取Looper，如果当前线程没有Looper就会抛异常
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}

//Looper.class
//每个线程都有一个自己的ThreadLocal，用于存储当前线程中的数据
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
private static void prepare(boolean quitAllowed) {
    //每个线程只能有一个Looper
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    //第一次调用Looper的prepare方法创建Looper时就会将Looper存入当前线程的ThreadLocal中
    sThreadLocal.set(new Looper(quitAllowed));
}
//从当前线程中的ThreadLocal中获取Looper
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

- 从上面的源码中我们可以看到，每个Handler都对应一个Looper，而每个线程都有自己的Looper，而且每个线程只能有一个Looper。虽然每个线程我们可以创建多个Handler，但是实际上对应的都是一个Looper。

```
//Handler.class
public final boolean sendMessage(Message msg)
{
    return sendMessageDelayed(msg, 0);
}

public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    //往MessageQueue中插入消息
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
- 从上面的源码中我们可以看出Handler的sendMessage方法所做的只是将Message插入到对应的MessageQueue当中。这样Looper调用MessageQueue的next方法就会取到添加的信息，Looper取到信息以后就会调用Handler的handleMessage方法处理信息。
- 而因为上面我们分析过，Looper是每个线程独有的。那这个Looper所在的线程是哪个，上面调用Handler的handleMessage方法的逻辑就是在哪个线程中执行的。
- 所以Handler的handleMessage方法是运行在创建Handler的线程中的。

#### 总结
##### Handler机制
我们用上面派生Handler的子类的例子来总结下Handler机制
- 首先我们在需要执行Handler的handleMessage方法的线程中创建Handler，比如在UI线程中，这样以来后面我们在子线程中执行完耗时的任务后就可以给Handler发送一个消息，最后在Handler的handleMessage方法中更新UI
- 在子线程中执行完耗时任务，比如网络请求后，将结果封装在Message中通过Handler的sendMessage方法把消息插入到Handler的消息队列
- Handler的Looper取到MessageQueue中的消息后交由Handler调用handleMessage方法执行。从而就把逻辑从子线程切换到了UI线程，并带回了子线程的执行结果

##### 使用Handler的3种方式

- 使用Handler有3种方式，分别是：派生Handler子类、Callback接口、post一个Runnable
- 派生Handler子类、Callback接口上面的分析中都举过例子，post一个Runnable的方式并不常用，这里也贴个简单的例子

```
public class HandlerActivity extends Activity {
    private Handler myHandler;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        myHandler = new Handler();

        new Thread(new Runnable() {
            @Override
            public void run() {
                myHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        Log.d(Thread.currentThread().getName(),"receive message");
                    }
                });
            }
        }).start();
    }
}
```
- 需要注意理解的是上面的run方法执行的线程也是在创建Handler的线程，在这个例子中就是UI线程。其实就是相当于把handleMessage方法中的代码直接放在run方法中了而已。

##### 注意点

- 主线程，也就是UI线程中的Looper和消息队列，在我们启动应用的时候系统已经为我们创建好了，所以我们可以在Activity中直接就创建Handler而不会报错
- 消息队列MessageQueue底层是单链表结构，插入删除有优势。通过Handler发送的多条消息会按发送的顺序排队执行。
- 由于Looper的loop方法是个无限循环，所以子线程中创建的Handler在明确不需要再使用时需要调用Looper的quit方法来退出循环。