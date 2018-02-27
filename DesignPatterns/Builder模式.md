### 在日常开发过程中时常需要用到设计模式，但是设计模式有23种，如何将这些设计模式了然于胸并且能在实际开发过程中应用得得心应手呢？和我一起跟着《Android源码设计模式解析与实战》一书边学边应用吧！
#### 今天我们要讲的是Builder模式（建造者模式）
---
### 定义
> 将一个复杂对象的构建和它的表示分离，使得同样的构建过程可以创建不同的表示
### 使用场景
- 当初始化一个对象特别复杂时，如参数多，且很多参数都具有默认值时
- 相同的方法，不同的执行顺序，产生不同的事件结果时
- 多个部件或零件，都可以装配到一个对象中，但是产生的运行效果又不相同时
- 产品类非常复杂，或者产品类中的调用顺序不同产生了不同的作用，这个时候使用建造者模式非常合适
### 使用例子
- AlertDialog
- universal-image-loader
---
### 实现
#### 实现的要点
- 简言之，就是把需要通过set方法来设置的多个属性封装在一个配置类里面
- 每个属性都应该有默认值
- 具体的set方法放在配置类的内部类Builder类中，并且每个set方法都返回自身，以便进行链式调用
### 实现方式
#### 下面以我们的图片加载框架ImageLoader为例来看看Builder模式的好处

##### 未采用Builder模式的ImageLoader

```
public class ImageLoader {
    //图片加载配置
    private int loadingImageId;
    private int loadingFailImageId;

    // 图片缓存，依赖接口
    ImageCache mImageCache = new MemoryCache();

    // 线程池，线程数量为CPU的数量
    ExecutorService mExecutorService = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());

    //省略单例模式实现
    
    /**
     * 设置图片缓存
     * @param cache
     */
    public void setImageCache(ImageCache cache) {
        mImageCache = cache;
    }

    /**
     * 设置图片加载中显示的图片
     * @param resId
     */
    public Builder setLoadingPlaceholder(int resId) {
        loadingImageId = resId;
    }

    /**
     * 设置加载失败显示的图片
     * @param resId
     */
    public Builder setLoadingFailPlaceholder(int resId) {
        loadingFailImageId = resId;
    }
    
    /**
     * 显示图片
     * @param imageUrl
     * @param imageView
     */
    public void displayImage(String imageUrl, ImageView imageView) {
        Bitmap bitmap = mImageCache.get(imageUrl);
        if (bitmap != null) {
            imageView.setImageBitmap(bitmap);
            return;
        }
        // 图片没有缓存，提交到线程池下载
        submitLoadRequest(imageUrl, imageView);
    }

    /**
     * 下载图片
     * @param imageUrl
     * @param imageView
     */
    private void submitLoadRequest(final String imageUrl, final ImageView imageView) {
        imageView.setImageResource(loadingImageId);
        imageView.setTag(imageUrl);
        mExecutorService.submit(new Runnable() {
            @Override
            public void run() {
                Bitmap bitmap = downloadImage(imageUrl);
                if (bitmap == null) {
                    imageView.setImageResource(loadingFailImageId);
                    return;
                }
                if (imageUrl.equals(imageView.getTag())) {
                    imageView.setImageBitmap(bitmap);
                }
                mImageCache.put(imageUrl, bitmap);
            }
        });
    }

    /**
     * 下载图片
     * @param imageUrl
     * @return
     */
    private Bitmap downloadImage(String imageUrl) {
        Bitmap bitmap = null;
        //省略下载部分代码
        return bitmap;
    }
}
```
> 从上面的代码中我们可以看出，每当需要增加一个设置选项的时候，就需要修改ImageLoader的代码，违背了开闭原则，而且ImageLoader中的代码会越来越多，不利于维护
##### 下面我们来看看如何用Builder模式来改造ImageLoader
- 首先是把ImageLoader的设置都放在单独的配置类里，每个set方法都返回this，从而达到链式调用的目的

```
public class ImageLoaderConfig {
    // 图片缓存，依赖接口
    public ImageCache mImageCache = new MemoryCache();

    //加载图片时的loading和加载失败的图片配置对象
    public DisplayConfig displayConfig = new DisplayConfig();

    //线程数量，默认为CPU数量+1；
    public int threadCount = Runtime.getRuntime().availableProcessors() + 1;

    private ImageLoaderConfig() {
    }


    /**
     * 配置类的Builder
     */
    public static class Builder {
        // 图片缓存，依赖接口
        ImageCache mImageCache = new MemoryCache();

        //加载图片时的loading和加载失败的图片配置对象
        DisplayConfig displayConfig = new DisplayConfig();

        //线程数量，默认为CPU数量+1；
        int threadCount = Runtime.getRuntime().availableProcessors() + 1;

        /**
         * 设置线程数量
         * @param count
         * @return
         */
        public Builder setThreadCount(int count) {
            threadCount = Math.max(1, count);
            return this;
        }

        /**
         * 设置图片缓存
         * @param cache
         * @return
         */
        public Builder setImageCache(ImageCache cache) {
            mImageCache = cache;
            return this;
        }

        /**
         * 设置图片加载中显示的图片
         * @param resId
         * @return
         */
        public Builder setLoadingPlaceholder(int resId) {
            displayConfig.loadingImageId = resId;
            return this;
        }

        /**
         * 设置加载失败显示的图片
         * @param resId
         * @return
         */
        public Builder setLoadingFailPlaceholder(int resId) {
            displayConfig.loadingFailImageId = resId;
            return this;
        }

        void applyConfig(ImageLoaderConfig config) {
            config.displayConfig = this.displayConfig;
            config.mImageCache = this.mImageCache;
            config.threadCount = this.threadCount;
        }

        /**
         * 根据已经设置好的属性创建配置对象
         * @return
         */
        public ImageLoaderConfig create() {
            ImageLoaderConfig config = new ImageLoaderConfig();
            applyConfig(config);
            return config;
        }
    }
}
```
- ImageLoader的修改

```
public class ImageLoader {
    //图片加载配置
    ImageLoaderConfig mConfig;

    // 图片缓存，依赖接口
    ImageCache mImageCache = new MemoryCache();

    // 线程池，线程数量为CPU的数量
    ExecutorService mExecutorService = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());

    //省略单例模式实现
    
    //初始化ImageLoader
    public void init(ImageLoaderConfig config) {
        mConfig = config;
        mImageCache = mConfig.mImageCache;
    }
    
    /**
     * 显示图片
     * @param imageUrl
     * @param imageView
     */
    public void displayImage(String imageUrl, ImageView imageView) {
        Bitmap bitmap = mImageCache.get(imageUrl);
        if (bitmap != null) {
            imageView.setImageBitmap(bitmap);
            return;
        }
        // 图片没有缓存，提交到线程池下载
        submitLoadRequest(imageUrl, imageView);
    }

    /**
     * 下载图片
     * @param imageUrl
     * @param imageView
     */
    private void submitLoadRequest(final String imageUrl, final ImageView imageView) {
        imageView.setImageResource(mConfig.displayConfig.loadingImageId);
        imageView.setTag(imageUrl);
        mExecutorService.submit(new Runnable() {
            @Override
            public void run() {
                Bitmap bitmap = downloadImage(imageUrl);
                if (bitmap == null) {
                    imageView.setImageResource(mConfig.displayConfig.loadingFailImageId);
                    return;
                }
                if (imageUrl.equals(imageView.getTag())) {
                    imageView.setImageBitmap(bitmap);
                }
                mImageCache.put(imageUrl, bitmap);
            }
        });
    }

    /**
     * 下载图片
     * @param imageUrl
     * @return
     */
    private Bitmap downloadImage(String imageUrl) {
        Bitmap bitmap = null;
        //省略下载部分代码
        return bitmap;
    }
}
```
- 调用形式，是不是很熟悉？

```
ImageLoaderConfig config = new ImageLoaderConfig.Builder()
        .setImageCache(new MemoryCache())
        .setThreadCount(2)
        .setLoadingFailPlaceholder(R.drawable.loading_fail)
        .setLoadingPlaceholder(R.drawable.loading)
        .create();
ImageLoader.getInstance().init(config);
```
### 总结
- 在构建的对象需要很多配置的时候可以考虑Builder模式，可以避免过多的set方法，同时把配置过程从目标类里面隔离出来，代码结构更加清晰
- Builder模式比较常用的实现形式是通过链式调用实现，这样更简洁直观
---
源码地址：https://github.com/snowdream1314/DesignPatternsExamples

