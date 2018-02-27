### 在日常开发过程中时常需要用到设计模式，但是设计模式有23种，如何将这些设计模式了然于胸并且能在实际开发过程中应用得得心应手呢？和我一起跟着《Android源码设计模式解析与实战》一书边学边应用吧！
#### 今天我们要讲的是单例模式
---
### 定义
> 确保某一个类只有一个实例，而且自行实例化并向整个系统提供这个实例
### 使用场景
- 确保某个类有且只有一个对象的场景，避免产生多个对象消耗过多的资源
- 某个类型的对象只应该有一个
### 使用例子
- 应用的Application
- 图片加载框架对象，比如我们的ImageLoader，常用的图片加载框架Glide，universal-image-loader等
- 数据请求管理类，比如可以用一个类来统一所有的数据请求处理，访问数据库，网络请求等，这样的类肯定只需要一个实例
---
### 实现
#### 实现的要点
- 构造函数不对外开放，必须为Private（就是不能用New的形式生成对象）
- 通过一个静态方法或者枚举返回单例对象
- 确保单例类的对象有且只有一个，尤其是在多线程环境下
- 确保单例类对象在反序列化时不会重新创建对象
### 常见的实现方式
#### 饿汉单例模式
```
public class Singleton {
    private static final Singleton singleton = new Singleton();
    //构造函数私有化
    private Singleton() {
    }
    //公有的静态函数，对外暴露获取单例对象的接口
    public static Singleton getInstance() {
        return singleton;
    }
}
```
- 饿汉单例模式采用的是静态变量 + fianl关键字的方式来确保单例模式，应用启动的时候就生成单例对象，效率不高
#### 懒汉模式

```
public class Singleton {
    private static Singleton singleton;
    //构造函数私有化
    private Singleton() {
    }
    //公有的静态函数，对外暴露获取单例对象的接口
    public static synchronized Singleton getInstance() {
        if (singleton == null) {
            singleton = new Singleton();
        }
        return singleton;
    }
}
```
- 懒汉模式的主要问题在于由于加了synchronized关键字，每调用一次getInstance方法，都会进行同步，造成了不必要的开销
> 以上的2种模式用的都不多，了解一下就好，下面介绍平时用得比较多的单例模式
#### Double Check Lock(DCL)模式（双重检查锁定模式）

```
public class Singleton {
    private static Singleton singleton = null;
    //构造函数私有化
    private Singleton() {
    }
    //公有的静态函数，对外暴露获取单例对象的接口
    public static Singleton getInstance() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```
- DCL模式是使用最多的单例模式，它不仅能保证线程安全，资源利用率高，第一次执行getInstance时单例对象才会实例化；同时，后续调用getInstance方法时又不会有懒汉模式的重复同步的问题，效率更高；在绝大多数情况下都能保证单例对象的唯一性
- DCL模式的缺点是第一次加载时由于需要同步反应会稍慢；在低于JDK1.5的版本里由于Java内存模型的原因有可能会失效
#### 静态内部类单例模式

```
public class Singleton {
    private Singleton() {
    }
    
    public static Singleton getInstance() {
        return SingletonHolder.sInstance;
    }
    
    //静态内部类
    private static class SingletonHolder {
        private static final Singleton sInstance = new Singleton();
    }
}
```
- 第一次加载Singleton类时不会初始化sInstance，只有在第一次调用getInstance方法时才会初始化sInstance，延迟了单例对象的实例化
- 静态内部类单例模式不仅能保证线程安全也能保证单例对象的唯一性
> 静态内部类单例模式和DCL模式是推荐的单例实现模式
#### 枚举单例

```
public enum Singleton {
    INSTANCE;
}
```
- 默认枚举实例的创建是线程安全的，并且在任何情况下它都是一个单例
- 其他的单例模式，在一种情况下会出现失效的情况——反序列化，但是枚举即使在反序列化情况下也不会失效
### 总结
- 单例模式是运用频率很高的模式，由于在客户端一般没有高并发的情况，现在的JDK版本也已经到了9了，一般推荐用DCL模式和静态内部类2种实现。
- 单例对象的生命周期很长，如果持有Context，很容易引发内存泄漏，所以传递给单例对象的Context最好是Application Context
### 最后加点福利
- 单例模式的代码格式都是固定的，每次都要那么写有点麻烦，咱们可以用添加模板的方法来偷懒，详情见图。

![添加单例模式模板](http://img.blog.csdn.net/20171009112412962?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbXl0aDEzMTQxMzE0/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

- 添加了模板后，在需要实现单例模式的类里面直接输入你的模板名字，如图中的sin, Android Studio就会出现提示，回车搞定！赶紧试试吧！

---
源码地址：https://github.com/snowdream1314/DesignPatternsExamples