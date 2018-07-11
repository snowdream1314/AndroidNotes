#### 开启多进程的方法
> Android 中使用多进程只有一种方法，那就是给四大组件(Activity、Service、Receiver、ContentProvider)在AndroidMenifest中指定android:process属性

```
<service
    android:name="com.xxq2dream.service.BookManagerService"
    android:process=":remote" />
```
- 通过如上的设置，BookManagerService就运行在独立的进程下，进程名为：包名:remote，比如：com.xxq2dream.android_ipc:remote
- 上面的设置方法表示remote进程是当前应用的私有进程，其他应用的组件不可以和它跑在同一个进程中
- android:process属性的设置还有另一种方式com.xxq2dream.android_ipc.remote，这种方式的进程是全局进程，其他应用通过ShareUID方式可以和它跑在同一个进程中。
- 一般我们在应用中用的比较多的是第一种，也即是应用的私有进程

#### 开启多进程模式后的问题
> Android为每个进程都分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，这就导致了不同的虚拟机中访问同一个类的对象会产生多个副本。一个应用间的多进程可以理解为就相当于2个不同的应用采用了SharedUID的模式。
- 所有运行在不同的进程中的四大组件，只要它们之间需要通过内存来共享数据，都会共享失败
- 静态成员和单例模式完全失效（不同进程的内存区域都不一样了）
- 线程同步机制完全失效（不同的进程锁的都不是同一个对象）
- Application会多次创建
- SharedPreferce的可靠性下降（SharedPreferce底层是读写xml文件实现的，系统对它的读写有一定的缓存策略，在内存中会有一份SharedPreferce文件的缓存，所以多个进程并发写操作可能导致数据丢失）

#### 多进程通信的实现方式
##### 使用Bundle
- 四大组件中的三大组件（Activity、Service、Receiver）都支持在Intent中传递Bundle数据
- Bundle中的数据除了基本类型，其他的都需要可序列化
- 适用于从一个进程启动另一个进程，比如在一个进程中启动另一个进程中的Activity、Service、Receiver，与此同时传递数据的情况
##### 使用文件共享
- 2个进程通过读写同一个文件来交换数据
- 当然要把数据写入文件，必然要求数据可以序列化和反序列化
- 文件共享对文件格式没有要求，只要读写双方约定好数据格式
- 文件共享的方式存在并发读写的问题，适合对数据同步要求不高的进程间通信，并且要妥善处理并发读写的问题

##### 使用Messager
- 使用Messager来传递Message，Message中能使用的字段只有what、arg1、arg2、Bundle和replyTo,自定义的Parcelable对象无法通过object字段来传输
- Message中的Bundle支持多种数据类型，replyTo字段用于传输Messager对象，以便进程间相互通信
- Messager以串行的方式处理客户端发来的消息，不适合有大量并发的请求
- Messager方法只能传递消息，不能跨进程调用方法

##### 使用AIDL
- AIDL接口可以通过编写AIDL文件然后由系统生成对应的Binder类，通过Binder我们就可以进行跨进程通信了
- AIDL文件中只支持如下集中类型：
    * 基本类型，如int、long等
    * String和CharSequence
    * List：只支持ArrayList，里面的每个元素都必须被AIDL支持
    * Map：只支持HashMap，里面的key、value都必须被AIDL支持
    * AIDL：所有的AIDL接口本身也可以在AIDL文件中使用
- AIDL文件中用到的自定义Parcelable对象和AIDL对象必须要显示的import进来
- 出了基本数据类型，其他类型的参数必须标明是入参还是出参，in表示输入型参数，out表示输出型参数，inout表示输入输出型参数

#### 序列化之Serializable接口和Parcelable接口
- Serializable接口和Parcelable接口都能实现序列化，但是有区别
##### Serializable接口
- Serializable接口实现序列化比较简单，只需要在需要实现序列化的类实现Serializable接口就行
- serialVersionUID可以指定也可以不指定，不指定的话系统会默认给我们生成
- serialVersionUID的作用是用于标识当前类的版本，便于在反序列化过程中判断类是否有更改，如果serialVersionUID不一致，反序列化就会失败
- 人为指定serialVersionUID后，如果不是破坏性的改动了类，比如版本升级后增加了一个字段，那反序列化以后仍然能成功；而如果没有指定，反序列化就会失败
- serialVersionUID是否指定需要根据具体的需要来确定
- 静态成员变量和transient关键字标记的变量不参与序列化过程
##### Parcelable接口
- Parcelable接口实现序列化稍微复杂，需要实现writeToParcel方法、describeContents方法以及生成器Creator
- 内容描述功能的describeContents方法一般都返回0，当当前对象中存在文件描述符时才返回1
- Parcelable接口中的Parcel内部包装了可序列化的数据，可以在Binder中自由传输
##### Parcelable接口和Serializable接口
- Serializable接口是Java中的序列化接口，使用简单但是开销大，序列化和反序列化过程需要大量的I/O操作，一般用于将对象序列化到存储设备中或是将对象序列化后通过网络进行传输
- Parcelable接口是Android中的序列化方式，使用稍微麻烦，但是效率很高，是Android中推荐的序列化方式，主要用在内存序列化上
