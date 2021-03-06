#### Java虚拟机管理的内存包括以下几个运行时数据区域：程序计数器、虚拟机栈和本地方法栈为线程私有；Java堆和方法区为线程共享
##### 线程私有内存
> 程序计数器：是当前线程所执行的字节码的行号指示器
- 所占内存空间较小
- 每个线程有一个独立的程序计数器
- 如果线程正在执行的是一个Java方法，则计数器记录的是正在执行的虚拟机字节码指令的地址
- 如果线程正在执行的是Native方法，则计数器值为空（Undefined)
- 此内存区域是唯一一个在Java虚拟机规范中没有规定任何OutOfMemoryError情况的区域
> Java虚拟机栈：描述Java方法执行的内存模型，每个Java方法在执行时都会创建一个栈帧，Java方法的执行过程就是栈帧在虚拟机栈中入栈到出栈的过程
- 生命周期与线程相同
- 如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常
- 如果虚拟机栈可以动态扩展，如果扩展时无法申请到足够的内存，就会抛出OutOfMemoryError异常
> 局部变量表: 位于虚拟机栈中，存放了编译期可知的各种基本数据类型（如int、long等）、对象引用（refrence类型，可能是指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄）和returnAddress类型（指向了一条字节码指令的地址）
- 局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小

> 本地方法栈：为虚拟机使用到的Native方法服务，会抛出StackOverflowError和OutOfMemoryError异常
##### 线程共享内存
> Java堆：在虚拟机启动时创建，几乎所有的对象实例都在堆上分配内存
- 是垃圾收集器管理的主要区域
- 从内存回收角度看，还可以细分为新生代和老年代；从内存分配的角度看，堆中可能划分出多个线程私有的分配缓冲区
- 可以处于物理上不连续的内存空间，只要逻辑连续即可
- 可以实现成固定大小，也可以是可扩展的
- 如果堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出OutOfMemoryError异常

> 方法区：用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等
- 不需要连续的内存和可以选择固定大小或者可扩展，也可以不实现垃圾收集
- 这一区域的内存回收目标主要是针对常量池的回收和对类型的卸载
- 当方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常

> 运行时常量池：是方法区的一部分，用于类加载后存放编译期生成的各种字面量和符号引用
- 保存Class文件中描述的符号引用和翻译出来的直接引用
- 运行时常量池具备动态性，运行期间也能将新的常量放入常量池
- 当常量池无法再申请到内存时会抛出OutOfMemoryError异常
---
#### 普通Java对象的创建
- 首先检查常量池是否能定位到一个类的符号引用，并检查这个类是否已经被加载、解析和初始化，如果没有就要先进行类的加载过程
- 对象所需内存的大小在类加载完成后便可以完全确定
- 对象内存在Java堆中分配,分配方式分为"指针碰撞"和"空闲列表"，分配方式由Java堆是否规整决定，而Java堆是否规整又由所采用的垃圾收集器是否带有压缩整理功能决定
    * "指针碰撞":假设Java堆中的内存是绝对规整的，所有用过的内存都放在一边，空闲的内存放在另一边，中间放着一个指针作为分界点的指示器，分配内存就是把这个指针向空闲的内存那边挪动一段与对象大小相等的距离
    * "空闲列表"：假设Java堆中的内存是不规整的，虚拟机就必须维护一个表，用来记录哪些内存块是可用的，在分配的时候从列表中找到一块足够大的空间分对象，并更新表上的记录
- 为对象划分空间的时候还需要注意线程安全的问题，解决方案有2个，一个是同步处理；一个是本地线程分配缓冲
    * 同步处理：对分配内存空间的动作进行同步处理，虚拟机采用的是CAS（比较并替换）配上失败重试的方式保证更新操作的原子性
    * 本地线程分配缓冲（TLAB）：把内存分配的动作按照线程划分在不同的空间之中进行，只有TLAB用完并分配新的TLAB时，才需要同步锁定
- 内存分配完成以后，接着虚拟机会将分配的内存空间的初始化为零值（不包括对象头），如果使用TLAB，这一过程也可以提前至TLAB分配时进行。这一步可以保证在Java代码中可以不赋初值就可以直接使用
- 再接着，虚拟机需要对对象进行必要的设置，例如这个对象是哪个类的实例、对象的哈希码、对象的GC分代年龄等，这些信息存放在对象的对象头中
- 这样对象就创建完毕了
#### 对象的内存布局
- 包括三部分：对象头、实例数据、对齐填充
##### 对象头
- 对象头包括2部分内容
- 一部分存储对象本身的运行时数据，如哈希码、GC分代年龄、锁状态标志、线程持有的锁等，这部分在32位和64位的虚拟机中长度分别为32位和64位
- 另一部分是类型指针，指向类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。并不是所有的虚拟机实现都必须在对象数据上保留类型指针。
#### 实例数据
- 实例数据部分是对象真正存储的有效信息，也是在程序代码中所定义的各种类型的字段内容
- 这部分的存储顺序会收到虚拟机分配策略参数和字段在Java源码中定义的顺序的影响
- 一般相同宽度的字段总是被分配到一起，在父类中定义的变量会出现在子类之前；如果CompactFields参数值为true（默认为true），那么子类中较窄的变量也可能插入到父类变量的空隙中
##### 对齐填充
- 对齐填充并不一定是必然存在的，也没有特别的含义，仅仅起到占位符的作用
---
#### 对象的访问定位
- Java程序需要通过虚拟机栈上的reference数据来操作堆上的具体对象，reference数据存储在虚拟机栈的局部变量表中
- 主流的访问方式有使用句柄和直接指针2种
##### 使用句柄访问
- Java堆中会划分出一块内存作为句柄池，reference中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自的具体的地址信息
##### 使用直接指针访问
- Java堆对象的布局中就必须考虑R如何放置访问类型数据的相关信息，而reference中存储的直接就是对象地址
##### 2种访问方式比较
- 使用句柄访问的最大好处就是reference中存储的时稳定的句柄地址，在对象被移动时只会改变句柄中的实例数据的指针，而reference本身不需要修改
- 使用直接指针访问的最大好处是速度更快，节省了一次指针定位的时间开销
