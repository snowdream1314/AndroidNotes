#### 分析系统生成的Binder类

```
package com.xxq2dream.aidl;

public interface IBookManager extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.xxq2dream.aidl.IBookManager {
        private static final java.lang.String DESCRIPTOR = "com.xxq2dream.aidl.IBookManager";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * 用于将服务端的Binder对象转换成客户端所需要的AIDL接口类型的对象
         */
        public static com.xxq2dream.aidl.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            //这里通过instanceof方法判断是否是跨进程，如果客户端和服务端位于同一进程，则返回的是服务端的Stub对象本身，不会调用跨进程的transact方法
            if (((iin != null) && (iin instanceof com.xxq2dream.aidl.IBookManager))) {
                return ((com.xxq2dream.aidl.IBookManager) iin);
            }
            //如果客户端和服务端不在同一个进程中，那么返回的是系统封装好的Stub.Proxy代理对象，客户端调用会走transact方法发起跨进程请求
            return new com.xxq2dream.aidl.IBookManager.Stub.Proxy(obj);
        }

        //返回当前Binder对象
        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        /**
         * 运行在服务端的Binder线程池中
         *
         * @param code 要运行的方法标识
         * @param data 客户端传递过来的参数
         * @param reply 回传给客户端的数据
         * @param flags
         * @return true 表示执行成功；false，执行失败，客户端的请求失败
         * @throws RemoteException
         */
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_getBookList: {
                    data.enforceInterface(DESCRIPTOR);
                    java.util.List<com.xxq2dream.aidl.Book> _result = this.getBookList();
                    reply.writeNoException();
                    //将结果写入reply返回
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(DESCRIPTOR);
                    com.xxq2dream.aidl.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.xxq2dream.aidl.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addBook(_arg0);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.xxq2dream.aidl.IBookManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            /**
             * 运行在客户端的Binder线程池中
             * 当客户端和服务端不在同一个进程中时，通过上面的asInterface方法返回的就是这个Proxy对象
             * 调用服务端的方法就是调用这里对应的方法，最终走的是transact方法发起远程过程调用
             *
             * @return
             * @throws RemoteException
             */
            @Override
            public java.util.List<com.xxq2dream.aidl.Book> getBookList() throws android.os.RemoteException {
                //用于向服务端传递参数
                android.os.Parcel _data = android.os.Parcel.obtain();
                //用于接收服务端传回来的结果
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.xxq2dream.aidl.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    //发起远程过程调用请求，同时当前线程挂起，服务端的onTransact方法会被调用
                    mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                    //上述调用过程返回后，当前线程继续执行，从reply中取出结果返回
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.xxq2dream.aidl.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            @Override
            public void addBook(com.xxq2dream.aidl.Book book) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        //接口的标识
        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }
    
    //提供的接口
    public java.util.List<com.xxq2dream.aidl.Book> getBookList() throws android.os.RemoteException;

    public void addBook(com.xxq2dream.aidl.Book book) throws android.os.RemoteException;

}

```
#### 动手编写Binder类
- 从上面的分析可以看出Binder类主要可以分为2个部分，一个是Stub类，也就是真正的Binder类；另一个就是AIDL接口
##### 抽取出AIDL接口

```
public interface IBookManager extends IInterface {

    static final String DESCRIPTOR = "com.xxq2dream.manualbinder.IBookManager";

    static final int TRANSACTION_getBookList = IBinder.FIRST_CALL_TRANSACTION + 0;
    static final int TRANSACTION_addBook = IBinder.FIRST_CALL_TRANSACTION + 1;
    <!--注释1-->
    <!--static final int TRANSACTION_registerListener= IBinder.FIRST_CALL_TRANSACTION + 2;-->
    <!--static final int TRANSACTION_unRegisterListener= IBinder.FIRST_CALL_TRANSACTION + 3;-->

    public List<Book> getBookList() throws RemoteException;
    public void addBook(Book book) throws RemoteException;
    <!--注释2-->
    <!--void registerListener(IOnNewBookArrivedListener listener) throws RemoteException;-->
    <!--void unRegisterListener(IOnNewBookArrivedListener listener) throws RemoteException;-->
}
```
##### 实现Stub类和Stub类的代理类
- 这个和上面自动生成的代码大部分都是一样的

```
public class BookManagerImpl extends Binder implements IBookManager {
    private static final String TAG = "BookManagerImpl";

    public BookManagerImpl() {
        Log.e(TAG, "attachInterface-->"+ System.currentTimeMillis());
        this.attachInterface(this,DESCRIPTOR);
    }

    /**
     * Cast an IBinder object into an com.xxq2dream.IBookManager interface,
     * generating a proxy if needed.
     */
    public static IBookManager asInterface(IBinder obj) {
        if ((obj == null)) {
            return null;
        }
        android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
        if (((iin != null) && (iin instanceof IBookManager))) {
            Log.e(TAG, "asInterface");
            return ((IBookManager) iin);
        }
        Log.e(TAG, "asInterface Proxy-->"+ System.currentTimeMillis());
        return new BookManagerImpl.Proxy(obj);
    }

    /**
     * 运行在服务端的Binder线程池中
     *
     * @param code 要运行的方法标识
     * @param data 客户端传递过来的参数
     * @param reply 回传给客户端的数据
     * @param flags
     * @return true 表示执行成功；false，执行失败，客户端的请求失败
     * @throws RemoteException
     */
    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        Log.e(TAG, "onTransact-->"+System.currentTimeMillis());

        switch (code) {
            case INTERFACE_TRANSACTION: {
                Log.e(TAG, "INTERFACE_TRANSACTION-->"+ System.currentTimeMillis());
                reply.writeString(DESCRIPTOR);
                return true;
            }
            case TRANSACTION_getBookList: {
                Log.e(TAG, "TRANSACTION_getBookList-->"+ System.currentTimeMillis());

                data.enforceInterface(DESCRIPTOR);
                List<Book> result = this.getBookList();
                reply.writeNoException();
                reply.writeTypedList(result);
                return true;
            }
            case TRANSACTION_addBook: {
                Log.e(TAG, "TRANSACTION_addBook-->"+ System.currentTimeMillis());

                data.enforceInterface(DESCRIPTOR);
                Book arg0;
                if (0 != data.readInt()) {
                    arg0 = Book.CREATOR.createFromParcel(data);
                }else {
                    arg0 = null;
                }
                this.addBook(arg0);
                reply.writeNoException();
                return true;
            }
            <!--注释6-->
            <!--case TRANSACTION_registerListener: {-->
            <!--    Log.e(TAG, "TRANSACTION_registerListener-->" +System.currentTimeMillis());-->
            <!--    data.enforceInterface(DESCRIPTOR);-->
            <!--    IOnNewBookArrivedListener arg0;-->
            <!--    arg0 = OnNewBookArrivedListenerImpl.asInterface(data.readStrongBinder());-->
            <!--    this.registerListener(arg0);-->
            <!--    reply.writeNoException();-->
            <!--    return true;-->
            <!--}-->
            <!--case TRANSACTION_unRegisterListener: {-->
            <!--    Log.e(TAG, "TRANSACTION_unRegisterListener-->" +System.currentTimeMillis());-->
            <!--    data.enforceInterface(DESCRIPTOR);-->
            <!--    IOnNewBookArrivedListener arg0;-->
            <!--    arg0 = OnNewBookArrivedListenerImpl.asInterface(data.readStrongBinder());-->
            <!--    this.unRegisterListener(arg0);-->
            <!--    reply.writeNoException();-->
            <!--    return true;-->
            <!--}-->
        }
        return super.onTransact(code, data, reply, flags);
    }

    @Override
    public List<Book> getBookList() throws RemoteException {
        return null;
    }

    @Override
    public void addBook(Book book) throws RemoteException {

    }

    <!--注释5-->
    <!--@Override-->
    <!--public void registerListener(IOnNewBookArrivedListener listener) {-->

    <!--}-->

    <!--@Override-->
    <!--public void unRegisterListener(IOnNewBookArrivedListener listener) {-->

    <!--}-->

    /**
     * Retrieve the Binder object associated with this interface.
     * You must use this instead of a plain cast, so that proxy objects
     * can return the correct result.
     */
    @Override
    public IBinder asBinder() {
        Log.e(TAG, "asBinder-->"+ System.currentTimeMillis());
        return this;
    }

    private static class Proxy implements IBookManager {

        private IBinder mRemote;

        Proxy(IBinder remote) {
            mRemote = remote;
        }

        public String getInterfaceDescriptor() {
            return DESCRIPTOR;
        }

        /**
         * 运行在客户端的Binder线程池中
         *
         * @return
         * @throws RemoteException
         */
        @Override
        public List<Book> getBookList() throws RemoteException {
            Log.e(TAG, "Proxy-->getBookList-->"+ System.currentTimeMillis());

            //用于向服务端传递参数
            Parcel data = Parcel.obtain();
            //用于接收服务端传回来的结果
            Parcel reply = Parcel.obtain();
            List<Book> result;
            try {
                data.writeInterfaceToken(DESCRIPTOR);
                //发起远程过程调用请求，同时当前线程挂起，服务端的onTransact方法会被调用
                mRemote.transact(TRANSACTION_getBookList, data, reply, 0);
                //上述调用过程返回后，当前线程继续执行，从reply中取出结果返回
                reply.readException();
                result = reply.createTypedArrayList(Book.CREATOR);
            } finally {
                reply.recycle();
                data.recycle();
            }
            return result;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            Log.e(TAG, "Proxy-->addBook-->"+ System.currentTimeMillis());

            //用于向服务端传递参数
            Parcel data = Parcel.obtain();
            //用于接收服务端传回来的结果
            Parcel reply = Parcel.obtain();
            try {
                data.writeInterfaceToken(DESCRIPTOR);
                if (book != null) {
                    data.writeInt(1);
                    book.writeToParcel(data,0);
                }else {
                    data.writeInt(0);
                }
                mRemote.transact(TRANSACTION_addBook, data, reply, 0);
                reply.readException();
            }finally {
                reply.recycle();
                data.recycle();
            }
        }
        
        <!--注释3-->
        <!--@Override-->
        <!--public void registerListener(IOnNewBookArrivedListener listener) throws RemoteException{-->
        <!--    Log.e(TAG, "Proxy-->registerListener-->"+ System.currentTimeMillis());-->

        <!--    //用于向服务端传递参数-->
        <!--    Parcel data = Parcel.obtain();-->
        <!--    //用于接收服务端传回来的结果-->
        <!--    Parcel reply = Parcel.obtain();-->
        <!--    try {-->
        <!--        data.writeInterfaceToken(DESCRIPTOR);-->
        <!--        data.writeStrongBinder((listener != null)?(listener.asBinder()) : (null));-->
        <!--        mRemote.transact(TRANSACTION_registerListener, data, reply, 0);-->
        <!--        reply.readException();-->
        <!--    }finally {-->
        <!--        reply.recycle();-->
        <!--        data.recycle();-->
        <!--    }-->

        <!--}-->
        
        <!--注释4-->
        <!--@Override-->
        <!--public void unRegisterListener(IOnNewBookArrivedListener listener) throws RemoteException{-->
        <!--    Log.e(TAG, "Proxy-->unRegisterListener-->"+ System.currentTimeMillis());-->

        <!--    //用于向服务端传递参数-->
        <!--    Parcel data = Parcel.obtain();-->
        <!--    //用于接收服务端传回来的结果-->
        <!--    Parcel reply = Parcel.obtain();-->
        <!--    try {-->
        <!--        data.writeInterfaceToken(DESCRIPTOR);-->
        <!--        data.writeStrongBinder((listener != null)?(listener.asBinder()) : (null));-->
        <!--        mRemote.transact(TRANSACTION_unRegisterListener, data, reply, 0);-->
        <!--        reply.readException();-->
        <!--    }finally {-->
        <!--        reply.recycle();-->
        <!--        data.recycle();-->
        <!--    }-->
        <!--}-->

        /**
         * Retrieve the Binder object associated with this interface.
         * You must use this instead of a plain cast, so that proxy objects
         * can return the correct result.
         */
        @Override
        public IBinder asBinder() {
            Log.e(TAG, "Proxy-->asBinder-->"+ System.currentTimeMillis());
            return mRemote;
        }
    }
}
```
- 以上就是我们自己动手写的Binder类，没有使用系统的方法，也不用编写AIDL文件

##### 扩展我们的Binder类
- 比如我们要实现当服务端有新书时就通知客户端，也就是观察者模式。我们需要在客户端绑定服务端后绑定监听的接口，还要提供解绑的接口，上面的代码中标记的注释1-6就是扩展Binder类的步骤
- 当然，由于AIDL中不支持普通的接口，只支持AIDL接口，我们要增加服务端的AIDL接口服务，像上面的代码里那样传递的接口肯定就是AIDL实现的接口，也就是
- 所以我们还得增加AIDL接口，同样的我们可以通过编写AIDL文件然后由系统自动生成，也可以手动编写。这里我们还是以手动的方式编写
- AIDL接口IOnNewBookArrivedListener

```
public interface IOnNewBookArrivedListener extends IInterface {
    static final String DESCRIPTOR = "com.xxq2dream.aidl.IOnNewBookArrivedListener";

    static final int TRANSACTION_onNewBookArrived = IBinder.FIRST_CALL_TRANSACTION + 0;

    void onNewBookArrived(Book book) throws RemoteException;
}
```
- 对应的Binder类

```
public class OnNewBookArrivedListenerImpl extends Binder implements IOnNewBookArrivedListener {
    private static final String TAG = "NewBookArrivedListener";

    public OnNewBookArrivedListenerImpl() {
        Log.e(TAG, "attachInterface-->"+ System.currentTimeMillis());
        this.attachInterface(this,DESCRIPTOR);
    }

    public static IOnNewBookArrivedListener asInterface (IBinder obj) {
        if ((obj == null)) {
            return null;
        }
        android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
        if (((iin != null) && (iin instanceof IOnNewBookArrivedListener))) {
            Log.e(TAG, "asInterface");
            return ((IOnNewBookArrivedListener) iin);
        }
        Log.e(TAG, "asInterface Proxy-->"+ System.currentTimeMillis());
        return new OnNewBookArrivedListenerImpl.Proxy(obj);
    }

    /**
     * 运行在服务端的Binder线程池中
     *
     * @param code 要运行的方法标识
     * @param data 客户端传递过来的参数
     * @param reply 回传给客户端的数据
     * @param flags
     * @return true 表示执行成功；false，执行失败，客户端的请求失败
     * @throws RemoteException
     */
    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        Log.e(TAG, "onTransact-->"+System.currentTimeMillis());

        switch (code) {
            case INTERFACE_TRANSACTION: {
                Log.e(TAG, "INTERFACE_TRANSACTION-->"+ System.currentTimeMillis());
                reply.writeString(DESCRIPTOR);
                return true;
            }
            case TRANSACTION_onNewBookArrived: {
                Log.e(TAG, "TRANSACTION_onNewBookArrived-->"+ System.currentTimeMillis());

                data.enforceInterface(DESCRIPTOR);
                Book _arg0;
                if ((0 != data.readInt())) {
                    _arg0 = Book.CREATOR.createFromParcel(data);
                } else {
                    _arg0 = null;
                }
                this.onNewBookArrived(_arg0);
                reply.writeNoException();
                return true;
            }
        }
        return super.onTransact(code, data, reply, flags);
    }

    @Override
    public void onNewBookArrived(Book book) {

    }

    /**
     * Retrieve the Binder object associated with this interface.
     * You must use this instead of a plain cast, so that proxy objects
     * can return the correct result.
     */
    @Override
    public IBinder asBinder() {
        return this;
    }

    private static class Proxy implements IOnNewBookArrivedListener {

        private IBinder mRemote;

        Proxy(IBinder remote) {
            mRemote = remote;
        }

        public String getInterfaceDescriptor() {
            return DESCRIPTOR;
        }

        /**
         * Retrieve the Binder object associated with this interface.
         * You must use this instead of a plain cast, so that proxy objects
         * can return the correct result.
         */
        @Override
        public IBinder asBinder() {
            Log.e(TAG, "Proxy-->asBinder-->"+ System.currentTimeMillis());
            return mRemote;
        }

        @Override
        public void onNewBookArrived(Book newBook) throws RemoteException{
            Log.e(TAG, "onNewBookArrived-->" + System.currentTimeMillis());
            android.os.Parcel _data = android.os.Parcel.obtain();
            android.os.Parcel _reply = android.os.Parcel.obtain();
            try {
                _data.writeInterfaceToken(DESCRIPTOR);
                if ((newBook != null)) {
                    _data.writeInt(1);
                    newBook.writeToParcel(_data, 0);
                } else {
                    _data.writeInt(0);
                }
                mRemote.transact(TRANSACTION_onNewBookArrived, _data, _reply, 0);
                _reply.readException();
            } finally {
                _reply.recycle();
                _data.recycle();
            }
        }
    }
}
```
##### Binder传递AIDL接口
- 通过上面的代码我们可以看到，和传递普通的Parcelable序列化对象不同的是，在Proxy类中要writeStrongBinder方法将AIDL接口加入到参数中去
- 在服务端的onTransact方法中要通过对应的Binder类的asInterface方法获得传递过来的接口参数

```
IOnNewBookArrivedListener arg0;
arg0 = OnNewBookArrivedListenerImpl.asInterface(data.readStrongBinder());
```

##### 结语
- 通过手动编写Binder类，可以让我们更好地理解Binder的工作机制
- 扩展了Binder类以后，下一步我们还需要相应的改造客户端和服务端，以便增加注册监听的功能