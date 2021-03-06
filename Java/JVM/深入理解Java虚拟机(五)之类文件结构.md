##### 实现语言无关性的基础是虚拟机和字节码存储格式
#### Class类文件结构
- Class文件是以一组8位字节为基础单位的二进制流，中间没有分隔符。
- Class文件中字节序为Big-Endian，最高位字节在地址最低位、最低位字节在地址最高位
- Class文件中只有2种数据类型：无符号数和表
##### 魔数和Class文件版本
- 每个Class文件的头四个字节称为魔数，用于确定这个文件是否为一个能被虚拟机接受的Class文件。Class文件的魔数值为：0xCAFEBABE
- 紧接着魔数的4个字节存储的是Class文件的版本号，分为次版本号和主版本号。高版本的JDK能向下兼容以前版本的Class文件，但不能运行以后的版本的Class文件
##### 常量池
- 常量池的入口紧接着主次版本号后面，是Class文件结构中与其他项目关联最多的数据类型
- 常量池的容量计数是从1开始的，0用于描述“不引用任何一个常量池项目”
- 常量池中主要存放：字面量和符号引用。
    * 字面量比较接近于Java层面的常量概念，如文本字符串、声明为final的常量值等
    * 符号引用包括：类和接口的全限定名；字段的名称和描述符；方法的名称和描述符
- 常量池中每一项常量都是一个表，JDK1.7中一共有14种常量类型
- Class文件中的方法和字段都需要引用常量池中CONSTANT_Utf8_info的常量，所以Java中方法和字段名的最大长度就是这个常量的最大长度，而CONSTANT_Utf8_info描述长度的length的值是u2类型，也就是2个字节，最大值位65535。所以Java方法和字段名的最大长度不能超过65535
##### 访问标志
- 访问标志用于标示一些类或接口的访问信息，如Class是类还是接口，是否是public，是否定义为abstract等
##### 类索引、父类索引、接口索引
- 这三项用于确定类的继承关系
- 类索引用于确定类的全限定名；父类索引用于确定类的父类的全限定名；接口索引集合用来描述类实现了哪些接口，并按照implements语句的顺序排列
##### 字段表集合
- 用于描述接口或类中声明的变量。包括类级变量(static修饰的)和实例变量，不包括方法内部声明的局部变量
- 字段包括字段修饰符(用标志位描述，和类的访问标志类似)、字段的简单名称、字段和方法的描述符以及属性表集合
    * 简单名称是指没有类型和参数修饰的方法或者字段名称
    * 描述符用于描述字段的数据类型、方法的参数列表和返回值
    * 全限定名就比如类的全限定名：org/soft/clazz/TestClass，仅仅是类全名中的“.”替换成了“／”
- 字段表集合不会列出从超类或父接口中继承而来的字段，但可能列出原本Java代码中不存在的字段，比如内部类指向外部类的实例的字段
##### 方法表集合
- 方法表集合和字段表集合类似，包括了方法的访问标志、名称索引、描述符索引和属性表集合
- 方法里的Java代码通过编译器编译成字节码指令后，存放在方法属性表集合中一个名为“Code”的属性里面
- 如果父类的方法在子类中没有被重写(Override)，方法表集合中就不会出现来自父类的方法信息。但有可能会出现由编译器自动添加的方法，如类构造器“<clinit>”方法和实例构造器“<init>”方法
- Java语言中要重载一个方法，除了要有相同的简单名称之外，还要求必须有与原方法不同的特征签名。特征签名就是一个方法中各个参数在常量池中的字段符号引用的集合。由于返回值不包含在特征签名中，所以Java语言里无法仅仅依靠返回值不同来对已有的方法进行重载
##### 属性表集合
- 属性表不要求具有严格的顺序，并且只要不与已有的属性名重复，任何人实现的编译器都可以向属性表中写入自己定义的属性信息，Java虚拟机运行时会忽略不认识的属性
###### Code属性
- Java程序方法体中的代码通过编译器编译成字节码指令后存储在Code属性内
- max_stack代表了操作数栈深度的最大值，虚拟机运行的时候需要根据这个值来分配栈帧中的操作栈深度
- Slot是虚拟机为局部变量分配内存所使用的最小单位，局部变量表中的Slot可以重用
- 虚拟机规范中明确限制了一个方法不允许超过65535条字节码指令
- 通过Javac编译器编译的时候把对this关键字的访问转变为对一个普通方法参数的访问，然后在虚拟机调用实例方法时自动传入此参数。这样就实现了在任何实例方法里面都能通过this关键字访问到此方法所属的对象
- 异常表是Java代码的一部分，编译器使用异常表而不是简单的跳转命令来实现Java异常及finally处理机制
###### Exceptions属性
- Exceptions属性的作用是列举出方法描述时在throws关键字后面列举的异常
###### LineNumberTable属性
- 用于描述Java源码行号和字节码行号(字节码的偏移量)之间的对应关系
- 可以在Javac中分别使用-g:none和-g:lines来取消或要求生成这项信息；如果关闭，当抛出异常时，堆栈中将不会显示出错的行号
###### LocalVariableTable属性
- 描述栈帧中局部变量中的变量与Java源码中定义的变量之间的关系
- 可以在Javac中分别使用-g:none和-g:vars来取消或要求生成这项信息；如果关闭，所有参数的名称都将消失，IDE将会使用诸如arg0、arg1之类的占位符代替原油的参数名
###### ConstantValue属性
- 用于通知虚拟机自动为静态变量赋值，只有类变量(被static关键字修饰的变量)才可以使用这项属性
- 虚拟机对非static类型的变量的赋值是在实例构造器<init>方法中进行；而对于类变量则有2种方式，在类构造器<clinit>方法中或者使用ConstantValue属性
- ConstantValue属性值只能限于基本类型和String
###### Signure属性
- 用于记录泛型签名中包含类型变量或参数化类型的类、接口、初始化方法或成员的泛型签名信息
- Java语言的泛型采用的是擦除法实现的伪泛型，在字节码(code属性)中，泛型信息编译(类型变量、参数化类型)之后都通通被擦除掉
- Java的反射API获取的泛型类型的最终数据源就是来自Signure属性
##### 字节码指令
- Java采用面向操作数栈而不是寄存器的架构
- 字节码指令集由于限制了虚拟机操作码的长度为一个字节，所以指令集的字节码总数不可能超过256条
- 编译器在编译期或运行期将byte和short类型的数据带符号扩展为相应的int类型的数据；将boolean和char类型的数据零扩展为相应的int类型数据
- 虚拟机规范规定在处理整型数据时，只有除法指令和求余指令中当出现除数为零时会导致虚拟机抛出ArithmeticException异常
- Java虚拟机在进行浮点数运算时会采用最接近数舍入模式，把浮点数转换整数时，Java虚拟机使用IEEE 754标准的向零舍入模式；多疑浮点数运算不能用于精确计算
- Java虚拟机处理浮点数运算时，当一个操作产生溢出时，将会使用有符号的无穷大来表示，如果某个操作没有明确的数学定义的话，将会使用NaN表示。所有使用NaN值作为操作数的算法操作，结果都会返回NaN
- 在对long类型的数值进行比较时，虚拟机采用带符号的比较方式，而对浮点数值进行比较采用无信号比较方式
- Java虚拟机直接支持小范围类型向大范围类型的安全转换，比如int到float；而处理窄化类型转化可能会导致结果产生不同的正负号、不同数量级的情况，很可能会导致数值的精度丢失
- Java虚拟机规范中明确规定数值类型的窄化转换指令永远不可能导致虚拟机抛出的运行时异常
- 各种数据类型的比较最后都会转化为int类型的比较操作
- Java虚拟机可以支持方法级的同步和方法内部一段指令序列的同步，2种同步结构都是使用管程(Monitor)来支持。Java虚拟机的指令集中有monitorenter和monitorexit2条指令来支持synchronized关键字的语义
