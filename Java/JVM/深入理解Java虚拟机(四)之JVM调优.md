##### VisualVM是JDK自带的运行监视和故障处理程序，可以很直观的查看Java虚拟机的信息和运行情况，这里我们以Android Studio为例，简单看看VisualVM的应用
##### 首先要找到本地JDK的路径
- 在终端输入:java -verbose，得到JDK路径
- 在终端输入：open + JDK路径，打开JDK所在文件夹
- 找到jvisualvm，双击执行即可打开VisualVM
![找到本机JDK路径](https://upload-images.jianshu.io/upload_images/2196721-26e992ac6faa52b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![找到VisualVM命令工具](https://upload-images.jianshu.io/upload_images/2196721-b6f971a0b1b863e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### VisualVM初探
![ VisualVM主界面](https://upload-images.jianshu.io/upload_images/2196721-8efc94ceddc3e888.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 打开Android Studio以后，我们从上图中就能看到左上角Android Studio的进程，双击可以查看详细的信息，如上图
- 在“概述”一栏下面，我们可以从“JVM参数”一项中看到虚拟机的具体设置，比如Java堆的大小，是否开启字节码验证，采用的垃圾收集器等
- 其中的JConsole 和Visual GC是插件，需要另外安装。
- 安装插件：顶部工具-->插件，打开插件面板
![VisualVM安装插件面板](https://upload-images.jianshu.io/upload_images/2196721-e743ec6d087502d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 插件的安装分为在线安装和手动安装，推荐用在线安装。在线安装首先要在“设置”选项卡里面添加新的更新配置，配置好如图的链接，因为默认的链接已经失效了。需要注意的是JDK的版本不一样，链接也不一样。[VisualVM官网插件中心](http://visualvm.github.io/pluginscenters.html)
- 配置好更新链接以后，在“可用插件”选项卡里就能看到有哪些插件可以安装，可以根据需要安装。比如Visual GC，JConsole插件等

##### VisualVM使用
- 安装好Visual GC插件以后，打开Visual GC插件选项卡就可以看到Android Studio的虚拟机运行的详细信息了，包括堆的大小，新生代、老年代的大小，类加载的时间，编译的时间，GC的情况等。
![Visual GC里可以查看很多信息](https://upload-images.jianshu.io/upload_images/2196721-f3261eac371d6fbe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 如上图，“Graphs”中可以看到各个阶段的花费的时间，其中从上到下分别为编译时间、类加载的时间、GC花费的次数和时间。
- 我们可以反复打开Android Studio，查看Android Studio打开过程中的时间花费情况，以便看看有哪些地方可以优化
- 比如，上图中我们看到启动过程中一共发生了19次GC，如果我们想看看GC的具体情况，就需要让虚拟机的打印GC日志。
##### 让Android Studio的虚拟机打印GC日志
- 首先，我们要找到Android Studio中配置虚拟机的文件。这个文件在Android Studio的安装目录里面。“应用程序”-->Android Studio-->右键显示包内容-->contents-->bin
![Android Studio的虚拟机配置文件](https://upload-images.jianshu.io/upload_images/2196721-4aab905eb254b94f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 上图中的studio.vmoptions就是虚拟机的配置文件，打开文件，加入让虚拟机打印GC日志的参数-XX:+PrintGCDetails，同时利用参数-Xloggc:./gclogs设置GC日志的输出位置。
##### 简单分析GC日志
> 0.875: [GC (Allocation Failure) 0.876: [ParNew: 104960K->13056K(118016K), 0.0681057 secs] 104960K->36781K(249088K), 0.0682067 secs] [Times: user=0.12 sys=0.05, real=0.07 secs] 
- ParNew表示的是发生GC的区域是新生代，这个名字和使用的GC收集器有关，因为这里Android Studio的新生代收集器为ParNew收集器，所以是区域名字是ParNew
- [ParNew: 104960K->13056K(118016K), 0.0681057 secs]，104960K表示的是GC前该区域已经使用的容量，13056K为GC后的容量，118016K为总容量，后面的为花费的时间
- 104960K->36781K(249088K)，表示“GC前Java堆已使用的容量->GC后Java堆已使用的容量(Java堆总容量)”

##### 以上是VisualVM的简单使用和GC的分析，实际上VisualVM的功能很强大，可以用于监控分析远程服务器虚拟机的情况，以便调优。
##### 下面附加一些虚拟机的设置参数
- -Xms：堆的最小值，比如-Xms240M,就表示设置堆的最小值为240M
- -Xmx：堆的最大值，比如-Xmx1024M，就表示设置堆的最大值为1G
- -Xmn：堆内新生代的大小，比如-Xmn10M，就表示设置新生代大小为10M
- -Xss：本地方法栈的容量，比如-Xss128K，就表示设置栈容量为128K
- -XX:PermSize，方法区(HotSpot虚拟机中也叫永久代)的大小，比如-XX:PermSize=10M，就表示设置方法区的大小为10M
- -XX:MaxPermSize，方法区的最大值，比如-XX:MaxPermSize=10M，就表示设置方法区最大值为10M(Java 8以后移除了方法区，取而代之的是本地元空间Metaspace，大小由-XX:MetaspaceSize和-XX:MaxMetaspaceSize调节)
- -XX:MaxDirectMemorySize，指定本机直接内存大小，如-XX:MaxdirectMemorySize=10M，就表示设置本机直接内存为10M，不设置的话默认会和Java堆的最大值一样
- -XX:+/-UseTALB  ---是否使用TALB(本地线程分配缓冲)
- -XX:+/-HeapDumpOnOutOfMemoryError ---让虚拟机在出现内存溢出异常时是否Dump出当前的内存堆转储快照
- -Xverify:none ---禁止JDK进行类加载时的字节码验证过程
- XX:+PrintGCTimeStamps ---打印GC停顿时间
- XX:+PrintGCDetails ---打印GC详细信息
- -verbose:gc ---打印GC信息
- -XX:+DisableExplicitGC ---屏蔽掉System.gc()
- -Xnoclassgc ---不对类进行回收，避免反复加载卸载类
- -XX:+/-UseCompressedOops ---是否启用压缩普通对象指针，主要是64位虚拟机的对象指针长度更长，会比32位占用更多内存，启用压缩普通对象指针后可以减少内存占用

##### 垃圾收集器相关
- -XX:+UseConcMarkSweepGC ---使用CMS垃圾收集器
- -XX:+UseParNewGC ---使用ParNew垃圾收集器
- -XX:SurvivorRatio ---新生代Eden区与Survivor区的比例，比如-XX:SurvivorRatio=8，就表示Eden区与一个Survivor区的空间比例是8:1
- -XX:PretenureSizeThreshold ---晋升老年代对象大小，大于设置值的对象直接在老年代中分配，比如-XX:PretenureSizeThreshold=3145728，就表示大于3M的对象都会直接在老年代中分配，其中3145728不能写成3M。这个参数设置只对Serial和ParNew收集器有效
- -XX:+/-HandlePromotionFailure --- 是否允许空间分配担保失败。Minor GC(新生代GC)时会检查老年代最大可用内存是否大于新生代所有对象总空间，如果不满足，就检查HandlePromotionFailure，如果不允许空间分配担保失败，就会触发Full GC(老年代GC)
- -XX:MaxTenuringThreshold ---对象晋升老年代的年龄阈值，比如-XX:MaxTenuringThreshold=15，就是说当对象熬过15次Minor GC(也就是15岁啦)，就将会被晋升到老年代中
- -XX:ParallelGCThreads ---限制垃圾收集的线程数，如XX:ParallelGCThreads=8，表示JVM在进行并行GC的时候，有8个线程用于GC
- -XX:MaxGCPauseMills ---Parallel Scavenge收集器用来控制最大垃圾收集停顿时间，允许大于0的毫秒数
- -XX:GCTimeRatio ---Parallel Scavenge收集器用来控制吞吐量大小，应当是一个大于0小于100的整数
- -XX:+/-UseAdaptiveSizePolicy ---Parallel Scavenge收集器用来开关自适应调节策略
- -XX:CMSInitiatingOccupancyFraction ---CMS收集器用来控制触发垃圾收集的老年代空间使用比例。比如-XX:CMSInitiatingOccupancyFraction=85，就表示当老年代的空间已经使用了85%时就会出发垃圾收集。这个值太低的话会频繁触发收集，太高的话容易导致老年代空间不够用户程序使用产生“Concurrent Mode Failure”失败，导致采用Serial Old收集器而性能下降
- -XX:+/-UseCMSCompactAtFullCollection ---CMS收集器用来开启内存碎片整理的开关。由于CMS收集器采用的是“标记-清除”算法，会产生很多内存碎片，从而需要进行内存整理。默认开启。
- -XX:CMSFullGCsBeforeCompaction ---CMS收集器用于设置执行多少次不压缩的Full GC后，跟着来一次带压缩的(整理内存碎片)。默认值为0，即每次进入Full GC时都进行碎片整理。
