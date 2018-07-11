##### 接上一篇文章[《Android多进程之手动编写Binder类》]()中向服务端注册监听事件的问题，在扩展了Binder类后，我们还需要改造对应的服务端和客户端
#### 客户端和服务端的改造
##### 服务端改造
- 增加注册监听接口的功能

```
private CopyOnWriteArrayList<IOnNewBookArrivedListener> mListenerList = new CopyOnWriteArrayList<IOnNewBookArrivedListener>();

private Binder mBinder = new BookManagerImpl(){
    @Override
    public List<Book> getBookList() throws RemoteException {
        Log.e(TAG, "getBookList-->"+ System.currentTimeMillis());
        return mBookList;
    }

    @Override
    public void addBook(Book book) throws RemoteException {
        Log.e(TAG, "addBook-->");
        mBookList.add(book);
    }

    @Override
    public void registerListener(IOnNewBookArrivedListener listener) {
        if (!mListenerList.contains(listener)) {
            mListenerList.add(listener);
        }else {
            Log.e(TAG, "already exists");
        }
        Log.e(TAG, "registerListener, size:"+mListenerList.size());
    }

    @Override
    public void unRegisterListener(IOnNewBookArrivedListener listener) {
        if (mListenerList.contains(listener)) {
            mListenerList.remove(listener);
            Log.e(TAG, "unRegisterListener listener succeed");
        }else {
            Log.e(TAG, "not found, can not unregister");
        }
        Log.e(TAG, "unRegisterListener, current size:"+mListenerList.size());

    }
};
```
- 添加一个任务，定时像书籍列表中添加一本书，并触发通知客户端的操作

```
@Override
public void onCreate() {
    super.onCreate();
    Log.e(TAG, "onCreate-->"+ System.currentTimeMillis());
    mBookList.add(new Book(1, "Android"));
    mBookList.add(new Book(2, "IOS"));
    new Thread(new ServiceWorker()).start();
}

private void onNewBookArrived(Book book) throws RemoteException{
    mBookList.add(book);
    Log.e(TAG, "new book arrived, notify listeners:" + mListenerList.size());
    for (int i=0; i<mListenerList.size(); i++) {
        IOnNewBookArrivedListener listener = mListenerList.get(i);
        Log.e(TAG, "new book arrived, notify listener:" + listener);
        listener.onNewBookArrived(book);
    }
}

private class ServiceWorker implements Runnable {

    @Override
    public void run() {
        while (!mIsServiceDestoryed.get()) {
            try {
                Thread.sleep(5000);
            }catch (InterruptedException e) {
                e.printStackTrace();
            }

            int bookId = mBookList.size() + 1;
            Book newBook = new Book(bookId, "new book#" + bookId);
            try {
                onNewBookArrived(newBook);
            }catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }
}
```
- 通知客户端的过程就是调用每个注册的监听接口的onNewBookArrived方法，也就是一个服务端调用客户端的过程，分别是调用onNewBookArrived-->onTransact-->Proxy的onNewBookArrived-->客户端
##### 客户端改造
- 首先要声明一个IOnNewBookArrivedListener对象，并注册到服务端中

```
private IBookManager mRemoteBookManager;
private ServiceConnection mConnection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
        Log.e(TAG, "ServiceConnection-->"+ System.currentTimeMillis());
        IBookManager bookManager = BookManagerImpl.asInterface(iBinder);
        mRemoteBookManager = bookManager;
        try {
            List<Book> list = bookManager.getBookList();
            Log.e(TAG, "query book list, list type:" + list.getClass().getCanonicalName());
            Log.e(TAG, "query book list:" + list.toString());
            Book newBook = new Book(3, "Android 进阶");
            bookManager.addBook(newBook);
            Log.e(TAG, "add book:" + newBook);
            List<Book> newList = bookManager.getBookList();
            Log.e(TAG, "query book list:" + newList.toString());
            bookManager.registerListener(mOnNewBookArrivedListener);

        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onServiceDisconnected(ComponentName componentName) {
        mRemoteBookManager = null;
        Log.e(TAG, "binder died");
    }
};

/**
 * 这个方法运行在客户端的binder线程池中，不能直接进行UI操作
 */
private IOnNewBookArrivedListener mOnNewBookArrivedListener = new OnNewBookArrivedListenerImpl(){
    @Override
    public void onNewBookArrived(Book book) {
        mHandler.obtainMessage(MESSAGE_NEW_BOOK_ARRIVED, book).sendToTarget();
    }
};
```

- 然后由于客户端的IOnNewBookArrivedListener回调方法onNewBookArrived运行在Binder线程池中，所以需要一个Handler来切换到UI线程

```
private static final int MESSAGE_NEW_BOOK_ARRIVED = 1;
private Handler mHandler = new Handler() {
    @Override
    public void handleMessage(Message message) {
        switch (message.what) {
            case MESSAGE_NEW_BOOK_ARRIVED:
                Log.e(TAG, "receive new book:" + message.obj);
                break;
            default:
                super.handleMessage(message);
        }
    }

};
```

- 最后需要在客户端退出时解除服务端的注册监听

```
@Override
protected void onDestroy() {
    if (mRemoteBookManager != null && mRemoteBookManager.asBinder().isBinderAlive()) {
        try {
            Log.e(TAG, "unRegister listener:" + mOnNewBookArrivedListener);
            mRemoteBookManager.unRegisterListener(mOnNewBookArrivedListener);
        }catch (RemoteException e) {
            e.printStackTrace();
        }
    }
    unbindService(mConnection);
    super.onDestroy();
}
```
##### 改造结果

![每隔5秒添加一本书](https://upload-images.jianshu.io/upload_images/2196721-aaa8ed68e755715b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/720)

- 通过上图我们可以看到，我们已经成功实现了预期的功能，并且服务端通知客户端的调用过程也如我们上面所说的那样

- 接下去我们退出应用，这样可以测试解绑监听的功能
- 
![解除绑定失败](https://upload-images.jianshu.io/upload_images/2196721-e068e746b4778030.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 从上图我们可以看到，服务端调用解绑失败了，提示找不到接口，这是咋回事呢？

##### 利用Binder进行进程间通信，Binder会把客户端传递的参数AIDL接口和Parcelable对象，重新转化并生成一个新的对象。因为对象是不能跨进程传输的，对象的跨进程传输本质上就是序列化和反序列化的过程。所以上述情况服务端根本就没有客户端的那个对象，那肯定找不到会解绑失败。那咋办呢？
#### RemoteCallBackList
##### RemoteCallBackList是啥

```
public class RemoteCallbackList<E extends IInterface> {
    /*package*/ ArrayMap<IBinder, Callback> mCallbacks
            = new ArrayMap<IBinder, Callback>();
    ...
}
```
- RemoteCallBackList的内部有一个Map结构用来保存所有的AIDL回调，这个Map的key是IBinder类型，value是CallBack类型

```
public boolean register(E callback, Object cookie) {
    synchronized (mCallbacks) {
        if (mKilled) {
            return false;
        }
        IBinder binder = callback.asBinder();
        try {
            Callback cb = new Callback(callback, cookie);
            binder.linkToDeath(cb, 0);
            mCallbacks.put(binder, cb);
            return true;
        } catch (RemoteException e) {
            return false;
        }
    }
}
```
- 每次有新的接口来，就调用register方法注册，添加到mCallbacks中
- 通过上面的源码我们可以看到，由于客户端跨进程传输的对象的底层的Binder对象都是同一个，所以我们可以通过这一点，当需要解除注册监听时，就可以通过遍历RemoteCallBackList找到与解绑客户端Binder对象相同的listener并删除即可
- 而且我们可以看到RemoteCallBackList中实现了线程同步，我们在利用它进行注册和解注册时不需要处理同步问题。
- 特别的是，客户端进程终止后，RemoteCallBackList能够自动解除客户端所注册的listener
##### 用RemoteCallBackList实现实现解绑注册

```
private RemoteCallbackList<IOnNewBookArrivedListener> mListenerList = new RemoteCallbackList<IOnNewBookArrivedListener>();
private Binder mBinder = new BookManagerImpl(){
    @Override
    public List<Book> getBookList() throws RemoteException {
        Log.e(TAG, "getBookList-->"+ System.currentTimeMillis());
        return mBookList;
    }

    @Override
    public void addBook(Book book) throws RemoteException {
        Log.e(TAG, "addBook-->");
        mBookList.add(book);
    }

    @Override
    public void registerListener(IOnNewBookArrivedListener listener) {
        //注册接口
        mListenerList.register(listener);
    }

    @Override
    public void unRegisterListener(IOnNewBookArrivedListener listener) {
        //解注册接口
        mListenerList.unregister(listener);
    }
};

//通知客户端
private void onNewBookArrived(Book book) throws RemoteException{
        mBookList.add(book);
        final int N = mListenerList.beginBroadcast();
        for (int i=0; i<N; i++) {
            IOnNewBookArrivedListener listener = mListenerList.getBroadcastItem(i);
            if (listener != null) {
                try {
                    listener.onNewBookArrived(book);
                }catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        }
        mListenerList.finishBroadcast();
    }
```
- 通过RemoteCallBackList的改造，我们就可以成功解注册客户端的listener了

##### 注意点
- RemoteCallBackList并不是一个List，我们无法像操作list一样操作它，比如调用size方法
- 遍历RemoteCallBackList必须按照上面代码中的方式进行，beginBroadcast方法和finishBroadcast方法必须配对使用