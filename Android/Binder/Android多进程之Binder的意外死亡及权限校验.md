##### 通过前几篇文章，我们对Binder的使用和工作流程有了一定的了解，但是还有几个问题休要我们去解决。一个是如果服务端进程意外退出，Binder死亡，那客户端就会请求失败；还有一个就是权限校验问题，就是服务端需要校验一下客户端的身份权限，不能谁都能请求服务端的服务
#### Binder意外死亡的处理
##### 给Binder设置DeathRecipient监听

- 在绑定Service服务后的onServiceConnected回调中给Binder注册死亡回调DeathRecipient
```
private ServiceConnection mConnection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
        Log.e(TAG, "ServiceConnection-->"+ System.currentTimeMillis());
        IBookManager bookManager = BookManagerImpl.asInterface(iBinder);
        mRemoteBookManager = bookManager;
        try {
            //注册死亡回调
            iBinder.linkToDeath(mDeathRecipient,0);
            ...

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
- 在DeathRecipient中相应的处理，比如重新连接服务端

```
private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient() {
    @Override
    public void binderDied() {
        Log.e(TAG, "mDeathRecipient-->binderDied-->");
        if (mRemoteBookManager == null) {
            return;
        }
        mRemoteBookManager.asBinder().unlinkToDeath(mDeathRecipient, 0);
        mRemoteBookManager = null;
        //Binder死亡，重新绑定服务
        Log.e(TAG, "mDeathRecipient-->bindService");
        Intent intent = new Intent(MainActivity.this, BookManagerService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }
};
```

- 为了测试，我们在服务端添加结束进程的代码

```
@Override
public void onCreate() {
    super.onCreate();
    Log.e(TAG, "onCreate-->"+ System.currentTimeMillis());
    new Thread(new ServiceWorker()).start();
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
            if (bookId == 8) {
                //结束当前进程，测试Binder死亡回调
                android.os.Process.killProcess(android.os.Process.myPid());
                return;
            }
            ...
        }
    }
}
```
- 测试效果

![服务端进程意外退出，客户端收到回调成功启动服务端](https://upload-images.jianshu.io/upload_images/2196721-f80947d38050400f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 从上面的测试我们可以看到客户端在服务端进程意外退出后，通过重新绑定服务又把服务端进程启动了。此外，我们还可以在ServiceConnection的onServiceDisconnected方法中处理服务端进程意外退出的情况，方法是一样的，就不测试了。2种方法的区别就在于onServiceDisconnected方法在客户端的UI线程中被回调，而binderDied方法在客户端的Binder线程池中被回调
#### 权限验证
##### 在onBind中通过自定义权限来验证
- 首先要自定义一个权限

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.xxq2dream.android_ipc">
    
    <permission
        android:name="com.xxq2dream.permission.ACCESS_BOOK_SERVICE"
        android:protectionLevel="normal"/>
    
</manifest>
```
- 然后在Service中的onBind方法中校验

```
@Nullable
@Override
public IBinder onBind(Intent intent) {
    Log.e(TAG, "onBind-->"+ System.currentTimeMillis());
    int check = checkCallingOrSelfPermission("com.xxq2dream.permission.ACCESS_BOOK_SERVICE");
    if (check == PackageManager.PERMISSION_DENIED) {
        Log.e(TAG, "PERMISSION_DENIED");
        return null;
    }
    Log.e(TAG, "PERMISSION_GRANTED");
    return mBinder;
}
```
![权限校验被拒绝](https://upload-images.jianshu.io/upload_images/2196721-bc5fa0edb2565dde.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 从上图中我们可以看到，由于我们的应用客户端没有声明服务端校验的权限，所以服务端校验不通过，我们只需要在我们的客户端添加相应的权限声明即可

```
<uses-permission android:name="com.xxq2dream.permission.ACCESS_BOOK_SERVICE"/>

```
![权限校验通过](https://upload-images.jianshu.io/upload_images/2196721-2d6c77f421bcfc98.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 在onTransact方法中进行权限校验

```
private Binder mBinder = new BookManagerImpl(){
    @Override
    public List<Book> getBookList() throws RemoteException {
        Log.e(TAG, "getBookList-->"+ System.currentTimeMillis());
        return mBookList;
    }

    ...
    
    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        int check = checkCallingOrSelfPermission("com.xxq2dream.permission.ACCESS_BOOK_SERVICE");
        if (check == PackageManager.PERMISSION_DENIED) {
            Log.e(TAG, "PERMISSION_DENIED");
            return false;
        }
        
        String packageName = null;
        //获取客户端包名 
        String[] packages = getPackageManager().getPackagesForUid(getCallingUid());
        if (packages != null && packages.length > 0) {
            packageName = packages[0];
        }
        //校验包名
        if (!packageName.startsWith("com.xxq2dream")) {
            return false;
        }
        Log.e(TAG, "PERMISSION_GRANTED");
        return super.onTransact(code, data, reply,flags);
    }
};
```

##### 结语
- Binder的一个很好的应用就是推送消息和保活。比如我们可以创建一个Service运行在一个独立的进程中，然后和我们的应用进程中的一个Service绑定。独立进程的Service每隔一定的时间向我们的服务端请求查看是否有新的消息，有的话就拉取新的消息，然后通知给应用进程的Service，执行弹出通知之类的操作。应用程序退出后，我们的Service进程还是存活的，这样就可以一直接收消息。当然，如果用户一键清理或是直接结束应用的话，我们的Service进程仍然会被干掉。