##### Messenger也可以作为Android多进程的一种通信方式，通过构建Message来在客户端和服务端之间传递数据
#### 简单使用Messenger
##### 客户端通过Messenger向服务端进程发送消息
- 构建一个运行在独立进程中的服务端Service：

```
public class MessengerService extends Service {
    private static final String TAG = "MessagerService";

    /**
     * 处理来自客户端的消息，并用于构建Messenger
     */
    private static class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message message) {
            switch (message.what) {
                case MESSAGE_FROM_CLIENT:
                    Log.e(TAG, "receive message from client:" + message.getData().getString("msg"));
                    break;
                default:
                    super.handleMessage(message);
                    break;
            }
        }
    }

    /**
     * 构建Messenger对象
     */
    private final Messenger mMessenger = new Messenger(new MessengerHandler());

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        //将Messenger对象的Binder返回给客户端
        return mMessenger.getBinder();
    }
}
```
- 注册service，当然要设置在不同的进程

```
<service
    android:name="com.xxq2dream.service.MessengerService"
    android:process=":remote" />
```
- 然后客户端是通过绑定服务端返回的binder来创建Messenger对象，并通过这个Messenger对象来向服务端发送消息

```
public class MessengerActivity extends AppCompatActivity {
    private static final String TAG = "MessengerActivity";

    private Messenger mService;

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            Log.e(TAG, "ServiceConnection-->" + System.currentTimeMillis());
            //通过服务端返回的Binder创建Messenger
            mService = new Messenger(iBinder);
            //创建消息，通过Bundle传递数据
            Message message = Message.obtain(null, MESSAGE_FROM_CLIENT);
            Bundle bundle = new Bundle();
            bundle.putString("msg", "hello service,this is client");
            message.setData(bundle);
            try {
                //向服务端发送消息
                mService.send(message);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {
            Log.e(TAG, "onServiceDisconnected-->binder died");
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_messenger);
        //绑定服务
        Intent intent = new Intent(this, MessengerService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        //解绑服务
        unbindService(mConnection);
        super.onDestroy();
    }
}
```
![服务端接收到客户端的消息](https://upload-images.jianshu.io/upload_images/2196721-b57c58504b617007.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 通过上面的额实践，我们可以看出利用Messenger进行跨进程通信，需要通过Message来传递消息，而Message可以通过setData方法利用Bundle来传递复杂的数据。
##### 服务端如果要回复消息给客户端，那就要用到Message的replyTo参数了
- 服务端改造

```
private static class MessengerHandler extends Handler {
    @Override
    public void handleMessage(Message message) {
        switch (message.what) {
            case Constant.MESSAGE_FROM_CLIENT:
                Log.e(TAG, "receive message from client:" + message.getData().getString("msg"));
                //获取客户端传递过来的Messenger，通过这个Messenger回传消息给客户端
                Messenger client = message.replyTo;
                //当然，回传消息还是要通过message
                Message msg = Message.obtain(null, Constant.MESSAGE_FROM_SERVICE);
                Bundle bundle = new Bundle();
                bundle.putString("msg", "hello client, I have received your message!");
                msg.setData(bundle);
                try {
                    client.send(msg);
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
                break;
            default:
                super.handleMessage(message);
                break;
        }
    }
}
```
- 客户端改造，主要是通过Handle构建一个Messenger对象，并在向服务端发送消息的时候，通过Message的replyTo参数将Messenger对象传递给服务端

```
/**
 * 用于构建客户端的Messenger对象，并处理服务端的消息
 */
private static class MessengerHandler extends Handler {
    @Override
    public void handleMessage(Message message) {
        switch (message.what) {
            case Constant.MESSAGE_FROM_SERVICE:
                Log.e(TAG, "receive message from service:" + message.getData().getString("msg"));
                break;
            default:
                super.handleMessage(message);
                break;
        }
    }
}

/**
 * 客户端Messenger对象
 */
private Messenger mClientMessenger = new Messenger(new MessengerHandler());

private ServiceConnection mConnection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
        Log.e(TAG, "ServiceConnection-->" + System.currentTimeMillis());
        mService = new Messenger(iBinder);
        Message message = Message.obtain(null, MESSAGE_FROM_CLIENT);
        Bundle bundle = new Bundle();
        bundle.putString("msg", "hello service,this is client");
        message.setData(bundle);
        //将客户端的Messenger对象传递给服务端
        message.replyTo = mClientMessenger;
        try {
            mService.send(message);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onServiceDisconnected(ComponentName componentName) {
        Log.e(TAG, "onServiceDisconnected-->binder died");
    }
};
```
![客户端收到服务端的消息回复](https://upload-images.jianshu.io/upload_images/2196721-ff8e7b2df868696d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 总结
- 使用Messager来传递Message，Message中能使用的字段只有what、arg1、arg2、Bundle和replyTo,自定义的Parcelable对象无法通过object字段来传输
- Message中的Bundle支持多种数据类型，replyTo字段用于传输Messager对象，以便进程间相互通信
- Messager以串行的方式处理客户端发来的消息，不适合有大量并发的请求
- Messager方法只能传递消息，不能跨进程调用方法
