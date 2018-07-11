#### Binder是什么
- Binder是Android的一个类，实现了IBinder接口
- 从IPC角度来说，Binder是Android中的一种跨进程通信方式
- Binder还可以理解为一种虚拟的物理设备，设备驱动是/dev/binder，Linux中没有
- 从Android Framework角度来说，Binder是ServiceManger连接各种Manger(ActivityManager、WindowManager等)和相应的ManagerService的桥梁
- 从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当bindService的时候会返回服务端的Binder对象，通过这个Binder对象可以调用服务端的服务
#### 使用Binder
##### 生成Binder类
- 可以通过2中方式生成Binder：通过AIDL文件让系统自动生成；我们自己手动编写Binder
##### 通过AIDL文件生成Binder
- 首先需要准备一个Parcelable对象

```
public class Book implements Parcelable {

    public int bookId;
    public String bookName;

    public Book(int bookId, String bookName) {
        this.bookId = bookId;
        this.bookName = bookName;
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel in) {
            return new Book(in);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };

    protected Book(Parcel in) {
        bookId = in.readInt();
        bookName = in.readString();
    }

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(bookId);
        dest.writeString(bookName);
    }
}
```

- 然后编写AIDL文件

![新建AIDL文件，AS中会自动创建aidl的目录](https://upload-images.jianshu.io/upload_images/2196721-08386756d27627e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/720)

```
// Book.aidl
package com.xxq2dream.aidl;

parcelable Book;

```

```
// IBookManager.aidl
package com.xxq2dream.aidl;

// Declare any non-default types here with import statements
//必须显示导入需要的Parcelable对象
import com.xxq2dream.aidl.Book;

//除了基本类型，其他参数都要标上方向，in表示输入型参数，out表示输出型参数，inout表示输入输出参数
interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
}
```
![最好把AIDL相关的文件都放在一个目录下](https://upload-images.jianshu.io/upload_images/2196721-036b6d9405e9e247.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/720)

- 文件编写完成以后通过Make Project命令就可以生成对应的Binder类

![通过make让系统自动生成Binder类](https://upload-images.jianshu.io/upload_images/2196721-bdf79a20090a8295.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/720)

![生成的Binder类](https://upload-images.jianshu.io/upload_images/2196721-49c95c0b7d9e86d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 模拟客户端和服务端的进程间通信
- 首先编写一个Service类，并设置android:process属性，开启在不同的进程

```
public class BookManagerService extends Service {
    private static final String TAG = "BookManagerService";

    private CopyOnWriteArrayList<Book> mBookList = new CopyOnWriteArrayList<Book>();

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
    };

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        Log.e(TAG, "onBind-->"+ System.currentTimeMillis());
        return mBinder;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        Log.e(TAG, "onCreate-->"+ System.currentTimeMillis());
        mBookList.add(new Book(1, "Android"));
        mBookList.add(new Book(2, "IOS"));
    }
}
```
- 注册Service

```
//AndroidManifest.xml
<application
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:roundIcon="@mipmap/ic_launcher_round"
    android:supportsRtl="true"
    android:theme="@style/AppTheme">
    <activity android:name=".MainActivity">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />

            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>

    <service
        android:name="com.xxq2dream.service.BookManagerService"
        android:process=":remote" />
</application>
```
![可以看到2个进程](https://upload-images.jianshu.io/upload_images/2196721-699c71979d0be7d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 客户端的话就简单在activity中通过bindService方法绑定服务端，然后通过返回的Binder调用服务端的方法

```
public class MainActivity extends AppCompatActivity {
    private static final String TAG = "MainActivity";

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            Log.e(TAG, "ServiceConnection-->"+ System.currentTimeMillis());
            //通过服务端回传的binderd得到客户端所需要的AIDL接口类型的对象，即我们上面的IBookManager
            IBookManager bookManager = BookManagerImpl.asInterface(iBinder);
            try {
                // 通过AIDL接口类型的对象bookManager调用服务端方法
                List<Book> list = bookManager.getBookList();
                Log.e(TAG, "query book list, list type:" + list.getClass().getCanonicalName());
                Log.e(TAG, "query book list:" + list.toString());
                Book newBook = new Book(3, "Android 进阶");
                bookManager.addBook(newBook);
                Log.e(TAG, "add book:" + newBook);
                List<Book> newList = bookManager.getBookList();
                Log.e(TAG, "query book list:" + newList.toString());

            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {
            Log.e(TAG, "binder died");
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Intent intent = new Intent(this, BookManagerService.class);
        //绑定服务，后面会回调onServiceConnected方法
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
- 通过以上的几个步骤我们就实现了一个简单的进程间通信的例子

![客户端进程日志](https://upload-images.jianshu.io/upload_images/2196721-a8a1d55775b84731.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![服务端进程日志](https://upload-images.jianshu.io/upload_images/2196721-b380cd0f80d3629e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 调用过程
- 通过打印的日志我们可以大概分析上面的例子中各个方法的调用过程，这里我们分析下调用服务端获取书本列表的过程
- 客户端调用bindService方法后，BookManagerService创建，调用Binder类的构造方法创建Binder
- 然后BookManagerService的onBind方法将创建的Binder返回给客户端
- 客户端的onServiceConnected方法被调用，然后调用Binder的asInterface方法得到AIDL接口类型的对象bookManager
- 调用bookManager的getBookList方法实际上调用的是Binder类中的Proxy类对应的getBookList方法
- 在Proxy类对应的getBookList方法中调用Binder的transact方法发起远程过程调用请求，同时当前线程挂起，服务端的onTransact方法会被调用
- 服务端的onTransact方法被调用，通过code找到具体要调用的方法，这里是TRANSACTION_getBookList
- 最后会调用BookManagerService中的mBinder对象对应的getBookList方法，将书籍列表mBookList返回，返回的结果在Parcel变量reply中
- Proxy类中的getBookList方法通过result = reply.createTypedArrayList(Book.CREATOR);获取到服务端返回的数据
- 客户端onServiceConnected方法中接收到数据，调用过程结束
##### 需要注意的地方
- 客户端调用远程请求时客户端当前的线程会被挂起，直到服务端进程返回数据，所以不能在UI线程中发起远程请求
- 客户端的onServiceConnected和onServiceDisconnected方法都运行在UI线程中，不可以在里面直接调用服务端的耗时方法
- 服务端的Binder方法运行在Binder线程池中，所以Binder方法不管是否耗时都应该采用同步的方式去实现
- 同上面一点，服务端的Binder方法需要处理线程同步的问题，上面的例子中CopyOnWriteArrayList支持并发读写，自动处理了线程同步
- AIDL中能够使用的List只有ArrayList，但AIDL支持的是抽象的List。因此虽然服务端返回的是CopyOnWriteArrayList，但是在Binder中会按照List的规范去访问数据并最终形成一个新的ArrayList传递给客户端

#### 结语
- 以上只是Binder的简单应用，Binder的使用过程中还是有很多问题需要注意的
- 比如Binder意外死亡以后怎么办？
- 如何注册监听回调，当服务端有新消息后马上通知注册回调的客户端？如何解除注册？
- 如何进行权限验证等