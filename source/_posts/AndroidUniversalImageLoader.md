---
title: Android-Universal-Image-Loader源码分析
date: 2019-11-26 15:14:00
tags: [Image,ImageLoader]
categories: Android
description: "ImageLoader 是 android 使用中出现比较早（PS：即的刚接触安卓项目的时候就用的是这个图片加载图，算算已经快5年了），使用最多的一个开源图片加载库了。随着glide , fresco 和 picasso等图片加载的库出现，ImageLoader使用变得越来越少。最近在看其他图片加载库的源码，顺便补补之前错过的一些事情。 "
---

# 前言

`ImageLoader` 是 `android` 使用中出现比较早（PS：即的刚接触安卓项目的时候就用的是这个图片加载图，算算已经快5年了），使用最多的一个开源图片加载库了。随着`glide` , `fresco` 和 `picasso`等图片加载的库出现，`ImageLoader`使用变得越来越少。最近在看其他图片加载库的源码，顺便补补之前错过的一些事情。

代码仓库地址：[Android-Universal-Image-Loader](https://github.com/nostra13/Android-Universal-Image-Loader)

# ImageLoader
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191126150914933.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)

这个是 `ImageLoader` 的架构，`ImageLader` 图片加载库的主要组成部分都包括在其中。

下边这幅图对应的是，组成上面架构的每个部分的对应的类实现：

![在这里插入图片描述](https://img-blog.csdnimg.cn/201911261509582.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)

- `ImageLoader` ：为`ImageView` 下载和展示图片的单例；
- `DisplayImageOptions` : 图片展示的配置项（加载中、空url、加载失败默认图等）；
- `ImageLoaderConfiguration` : `ImageLoader` 的配置项；
- `ImageAware` ：表示图像感知视图，该视图提供了图像处理和显示所需的所有属性和行为；
- `ImageLoadingListener` ：监听图片加载进度，开始、失败、成功、取消；
- `ImageLoaderEngine` ：执行图片下载和展现任务；
- `BitmapDisplayer` ：展现 `Bitmap` 在 `ImageView` 上的时候可以修改这个 `Bitmap` 或添加展示的动画效果；
- `BitmapProcessor` ：可以处理原始的`Bitmap` ；
- `MemoryCache` ： `Bitmap` 内存缓存接口；
- `DiskCache` ：磁盘缓存；
- `ImageDecoder` ：根据`ImageDecodingInfo`信息得到图片并根据参数将其转换为 Bitmap。
- `ImageDownloader` ：通过`URI` 获取图片；
- `DisplayBitmapTask` ：展示图片并进行回调；
- `ProcessAndDisplayImageTask` ：处理图片和展现图片的任务，用于加载内存缓存中的图片；
- `LoadAndDisplayImagTask` ：处理加载和显示图像的任务，用于从Internet或文件系统加载图像为 `Bitmap`；

## Config配置

初始化配置参数，参数`configuration`为`ImageLoader`的配置信息，包括图片最大尺寸、任务线程池、磁盘缓存、下载器、解码器等等。

```java
ImageLoaderConfiguration{
    final Resources resources;//上下文环境中的resource
    final int maxImageWidthForMemoryCache;//内存缓存最大宽度
    final int maxImageHeightForMemoryCache;//内存缓存最大高度
    final int maxImageWidthForDiskCache;//磁盘缓存最大宽度
    final int maxImageHeightForDiskCache;//磁盘缓存最大高度
    //在将图像保存到磁盘缓存之前先对其进行调整大小/压缩处理
    final BitmapProcessor processorForDiskCache;
    final Executor taskExecutor;//自定义图片加载和展现的线程池
    final Executor taskExecutorForCachedImages;//自定义展现在磁盘上的图片的线程池
    final boolean customExecutor;//是否自定义下载的线程池
    final boolean customExecutorForCachedImages;//是否自定义缓存图片的线程池
    //默认核心线程数和线程池容量为3
    final int threadPoolSize;
    //默认的线程优先级低两级
    final int threadPriority;
    //LIFO,FIFO;默认为先进先出FIFO
    final QueueProcessingType tasksProcessingType;
    //内存缓存，默认为MemoryClass的八分之一，3.0之后为LargeMemoryClass的八分之一
    //如果开启denyCacheImageMultipleSizesInMemory，那么缓存为FuzzyKeyMemoryCache实例，只判断图片地址不判断大小，如果相同那么刷新缓存
    final MemoryCache memoryCache;
    //LruDiskCache，大小默认存储为Long.MAX_VALUE，默认最大数量为Long.MAX_VALUE;
    final DiskCache diskCache;
    //通过URI从网络或文件系统或应用程序资源中检索图像，默认为HttpURLConnection进行网络下载
    //提供了imageDownloader方法可以自定义，比如使用HttpClient或者OkHttp
    final ImageDownloader downloader;
    //将图像解码为Bitmap，将其缩放到所需大小
    final ImageDecoder decoder;
    //包含图像显示选项(默认图设置以及其他默认选项)
    final DisplayImageOptions defaultDisplayImageOptions;
    //网络禁止下载器，一般不直接应用
    final ImageDownloader networkDeniedDownloader;
    //在慢速网络上处理下载
    final ImageDownloader slowNetworkDownloader;
}
```

还有一个对于某个`ImageView` 进行展示设置的 `DisplayImageOptions` ，配置图片显示的配置项。比如加载前、加载中、加载失败应该显示的占位图片，图片是否需要在磁盘缓存，是否需要在内存缓存等。

## 视图

讲视图主要是想让`ImageView` 与 `ImageLoader` 联系在一起来，`ImageLoader` 通过 `ImageAware` 接口实现图片在视图上的展现。

```java
public interface ImageAware {
	int getWidth();
	int getHeight();
	ViewScaleType getScaleType();
	View getWrappedView();
	boolean isCollected();
	int getId();
	boolean setImageDrawable(Drawable drawable);
	boolean setImageBitmap(Bitmap bitmap);
}
```

- ImageAware->ViewAware
- ImageAware->ViewAware->ImageViewAware
- ImageAware->NonViewAware

其中 `ViewAware` 是抽象类，所以 `ImageAware` 只有 `ImageViewAware` 和 `NonViewAware` 两个实现类。 

`NonViewAware` 提供处理原始图像所需的信息，但不显示图像。当用户只需要加载和解码图像的时候可以使用它。

## 加载回调

主要进行图片加载过程中的事件监听。

```java
public interface ImageLoadingListener {
    //开始加载
    void onLoadingStarted(String imageUri, View view);
    //加载失败
    void onLoadingFailed(String imageUri, View view, FailReason failReason);
    //加载完成
    void onLoadingComplete(String imageUri, View view, Bitmap loadedImage);
    //取消加载
    void onLoadingCancelled(String imageUri, View view);
}
```

## 图片展示

在`ImageAware`中显示`bitmap` 对象的接口。可在实现中对 `bitmap` 做一些额外处理，比如加圆角、动画效果。

默认的`BitmapDisplay` 是 `SimpleBitmapDisplayer` 仅仅实现了加载图片的功能，`ImageLoader` 还提供了`CircleBitmapDisplayer` 、`FadeInBitmapDisplayer` 和 `RoundedBitmapDisplayer` 等其他的实现。

```java
public interface BitmapDisplayer {
	void display(Bitmap bitmap, ImageAware imageAware, LoadedFrom loadedFrom);
}
public final class SimpleBitmapDisplayer implements BitmapDisplayer {
	@Override
	public void display(Bitmap bitmap, ImageAware imageAware, LoadedFrom loadedFrom) {
		imageAware.setImageBitmap(bitmap);
	}
}
```

## 位图处理

图片处理接口。可用于对图片预处理(`Pre-process`)和后处理(`Post-process` )，这两个处理器的配置都是在`DisplayImageOptions` 进行设置。其中预处理是在图片获取完缓存之前处理，后端处理是指在展示前的处理。

```java
public interface BitmapProcessor {
	Bitmap process(Bitmap bitmap);
}
```

## 内存缓存

内存缓存的是`Bitmap` ，默认的缓存容器是`LruMemoryCache` 。内存缓存的`Bitmap` 都是通过数据流解码生成的。

```java
public interface MemoryCache {
	boolean put(String key, Bitmap value);
	Bitmap get(String key);
	Bitmap remove(String key);
	Collection<String> keys();
	void clear();
}
public static MemoryCache createMemoryCache(Context context, int memoryCacheSize) {
    if (memoryCacheSize == 0) {
        ActivityManager am = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
        int memoryClass = am.getMemoryClass();
        if (hasHoneycomb() && isLargeHeap(context)) {
            memoryClass = getLargeMemoryClass(am);
        }
        memoryCacheSize = 1024 * 1024 * memoryClass / 8;
    }
    return new LruMemoryCache(memoryCacheSize);
}
```

`LruMemoryCache` 是区分**size**的，如果`ImageLoaderConfiguration` 设置 `denyCacheImageMultipleSizesInMemory` 那么缓存为 `FuzzyKeyMemoryCache` 实例，只判断图片地址不判断大小，如果相同那么刷新缓存。 `FuzzyKeyMemoryCache`  只是重写了`MemoryCache` 的 `put`  方法。

## 图片解码器

根据`ImageDecodingInfo`信息得到图片并根据参数将其转换为 `Bitmap` 。

```java
public interface ImageDecoder {
	Bitmap decode(ImageDecodingInfo imageDecodingInfo) throws IOException;
}
public static ImageDecoder createImageDecoder(boolean loggingEnabled) {
  return new BaseImageDecoder(loggingEnabled);
}
```

`BaseImageDecoder` 是`ImageLoaderConfiguration`默认的解码器。

```java
public class BaseImageDecoder implements ImageDecoder {
    @Override
	public Bitmap decode(ImageDecodingInfo decodingInfo) throws IOException {
		Bitmap decodedBitmap;
		ImageFileInfo imageInfo;
        //通过ImageDecodingInfo中的信息获取数据流，图片下载器部分会讲怎么获取数据流
		InputStream imageStream = getImageStream(decodingInfo);
		if (imageStream == null) {
			L.e(ERROR_NO_IMAGE_STREAM, decodingInfo.getImageKey());
			return null;
		}
		try {
            //确定图片尺寸和旋转角度，生成图片文件信息
            imageInfo = defineImageSizeAndRotation(imageStream, decodingInfo);
            //数据流的游标重置
            imageStream = resetStream(imageStream, decodingInfo);
            //生成控制Bitmap进行采样的Option
            Options decodingOptions = prepareDecodingOptions(imageInfo.imageSize, decodingInfo);
            //将输入流解码为位图
			decodedBitmap = BitmapFactory.decodeStream(imageStream, null, decodingOptions);
		} finally {
			IoUtils.closeSilently(imageStream);
		}
		if (decodedBitmap == null) {
			L.e(ERROR_CANT_DECODE_IMAGE, decodingInfo.getImageKey());
		} else {
            //对Bitmmap进行缩放和旋转
			decodedBitmap = considerExactScaleAndOrientatiton(decodedBitmap, decodingInfo, imageInfo.exif.rotation,
					imageInfo.exif.flipHorizontal);
		}
		return decodedBitmap;
    }
    /****部分代码省略***/
}
```

在解码的过程中我们之所以要重置游标，是因为我们在读头信息的时候已经读出了部分数据，所以这里要重置游标得到完整的图片数据。

## 磁盘缓存

本地图片缓存，可向本地磁盘缓存保存图片或从本地磁盘读取图片。`LruDiskCache `是`ImageLoaderConfiguration`默认的磁盘缓存容器。这次缓存的图片文件都是通过`InputStream` 保存在磁盘上的，实现是通过调用 `save` 方法。

```java
public interface DiskCache {
	File getDirectory();
	File get(String imageUri);
	boolean save(String imageUri, InputStream imageStream, IoUtils.CopyListener listener) throws IOException;
	boolean save(String imageUri, Bitmap bitmap) throws IOException;
	boolean remove(String imageUri);
	void close();
	void clear();
}
public static DiskCache createDiskCache(Context context, FileNameGenerator diskCacheFileNameGenerator,
        long diskCacheSize, int diskCacheFileCount) {
    File reserveCacheDir = createReserveDiskCacheDir(context);
    if (diskCacheSize > 0 || diskCacheFileCount > 0) {
        ///Android/data/[app_package_name]/cache/uil-images
        File individualCacheDir = StorageUtils.getIndividualCacheDirectory(context);
        try {
            //缓存目录，缓存大小，缓存数量，缓存文件名生成器都不能为空
            return new LruDiskCache(individualCacheDir, reserveCacheDir, diskCacheFileNameGenerator, diskCacheSize,
                    diskCacheFileCount);
        } catch (IOException e) {
            L.e(e);
            // continue and create unlimited cache
        }
    }
    File cacheDir = StorageUtils.getCacheDirectory(context);
    //UnlimitedDiskCache大小没有限制
    return new UnlimitedDiskCache(cacheDir, reserveCacheDir, diskCacheFileNameGenerator);
}
```

## 网络下载

获取`Uri` 对应的 `Stream` , `extra` 为辅助的下载器，可以通过`DisplayImageOptions` 得到`extraForDownloader` 。下载主要有`http` 、`https` 、`file` 、`content` 、`assets` 和 `drawable` 。	

```java
public interface ImageDownloader {
    InputStream getStream(String imageUri, Object extra) throws IOException;
    /** Represents supported schemes(protocols) of URI. Provides convenient methods for work with schemes and URIs. */
    public enum Scheme {
        HTTP("http"), HTTPS("https"), FILE("file"), CONTENT("content"), ASSETS("assets"), DRAWABLE("drawable"), UNKNOWN("");
        private String scheme;
        private String uriPrefix;
        Scheme(String scheme) {
            this.scheme = scheme;
            uriPrefix = scheme + "://";
        }
        /***部分代码省略***/
    }
}
```

`BaseImageDownloader` 为默认的下载器：内部通过下载资源的类型的不同有着不同的实现。

```java
public class BaseImageDownloader implements ImageDownloader {
    @Override
	public InputStream getStream(String imageUri, Object extra) throws IOException {
		switch (Scheme.ofUri(imageUri)) {
			case HTTP:
			case HTTPS:
				return getStreamFromNetwork(imageUri, extra);
			case FILE:
				return getStreamFromFile(imageUri, extra);
			case CONTENT:
				return getStreamFromContent(imageUri, extra);
			case ASSETS:
				return getStreamFromAssets(imageUri, extra);
			case DRAWABLE:
				return getStreamFromDrawable(imageUri, extra);
			case UNKNOWN:
			default:
				return getStreamFromOtherSource(imageUri, extra);
		}
    }
    /***部分代码省略***/
}
```

看一个从网络请求中获取`Stream` 的实现（`ImageLoader`默认的网络下载）：

```java
public class BaseImageDownloader implements ImageDownloader {
    protected InputStream getStreamFromNetwork(String imageUri, Object extra) throws IOException {
        //根据imageUri，创建HttpURLConnection对象
		HttpURLConnection conn = createConnection(imageUri, extra);
        int redirectCount = 0;
        //最多重定向请求5次
		while (conn.getResponseCode() / 100 == 3 && redirectCount < MAX_REDIRECT_COUNT) {
			conn = createConnection(conn.getHeaderField("Location"), extra);
			redirectCount++;
		}
		InputStream imageStream;
		try {
			imageStream = conn.getInputStream();
		} catch (IOException e) {
			// Read all data to allow reuse connection (http://bit.ly/1ad35PY)
			IoUtils.readAndCloseStream(conn.getErrorStream());
			throw e;
        }
        //如果responseCode不是200那么关闭请求抛出IO异常
		if (!shouldBeProcessed(conn)) {
			IoUtils.closeSilently(imageStream);
			throw new IOException("Image request failed with response code " + conn.getResponseCode());
		}
		return new ContentLengthInputStream(new BufferedInputStream(imageStream, BUFFER_SIZE), conn.getContentLength());
	}
}
```

### OkHttp网络下载
只需要在进行`ImageLoader`配置的时候调用`ImageLoaderConfiguration.Builder`的`imageDownloader` 方法进行设置。

```java
public class OkHttpImageDownloader extends BaseImageDownloader {
    private OkHttpClient client;
    public OkHttpImageDownloader(Context context, OkHttpClient client) {
        super(context);
        this.client = client;
    }
    @Override
    protected InputStream getStreamFromNetwork(String imageUri, Object extra) throws IOException {
        Request request = new Request.Builder().url(imageUri).build();
        ResponseBody responseBody = client.newCall(request).execute().body();
        InputStream inputStream = responseBody.byteStream();
        int contentLength = (int) responseBody.contentLength();
        return new ContentLengthInputStream(inputStream, contentLength);
    }
}
```

## ImageLoader

讲完了组成的`ImageLoader` 的一整套图片加载流程的没个部分：网络下载、磁盘缓存、数据解码、内存缓存、位图处理、图片展示和业务回调。下面我们看看`ImageLoader`是怎么将这些部分是怎么串在一起的。

使用双重校验锁（DCL：double-checked locking）实现单例操作 [Java版的7种单例模式](https://dandanlove.blog.csdn.net/article/details/101759634)。

```java
public class ImageLoader {
    private ImageLoaderConfiguration configuration;//图片加载配置信息
    private ImageLoaderEngine engine;//图片加载引擎
    private ImageLoadingListener defaultListener = new SimpleImageLoadingListener();//默认的回调监听
    private volatile static ImageLoader instance;//单例
    /** Returns singleton class instance */
    public static ImageLoader getInstance() {
        if (instance == null) {
            synchronized (ImageLoader.class) {
                if (instance == null) {
                    instance = new ImageLoader();
                }
            }
        }
        return instance;
    }
    //初始化方法
    public synchronized void init(ImageLoaderConfiguration configuration) {
        if (configuration == null) {
            throw new IllegalArgumentException(ERROR_INIT_CONFIG_WITH_NULL);
        }
        if (this.configuration == null) {
            L.d(LOG_INIT_CONFIG);
            engine = new ImageLoaderEngine(configuration);
            this.configuration = configuration;
        } else {
            L.w(WARNING_RE_INIT_CONFIG);
        }
    }
    /***其他代码省略***/
}
```

上面代码是 `ImageLoader` 的构造初始化方法，接下分析它加载图片时候的调用：

```java
public class ImageLoader {
    public void displayImage(String uri, ImageView imageView) {
	    	displayImage(uri, new ImageViewAware(imageView), null, null, null);
  	}
    //最终加载图片的方法
    public void displayImage(String uri, ImageAware imageAware, DisplayImageOptions options,
                             ImageSize targetSize, ImageLoadingListener listener, ImageLoadingProgressListener progressListener) {
        //校验配置是否为空
        checkConfiguration();
        if (imageAware == null) {
            throw new IllegalArgumentException(ERROR_WRONG_ARGUMENTS);
        }
        //添加默认的空回调
        if (listener == null) {
            listener = defaultListener;
        }
        //添加默认的图片展示配置
        if (options == null) {
            options = configuration.defaultDisplayImageOptions;
        }
        //下载地址为空
        if (TextUtils.isEmpty(uri)) {
            engine.cancelDisplayTaskFor(imageAware);//取消对于当前imageAware的展示任务
            listener.onLoadingStarted(uri, imageAware.getWrappedView());//回调展示开始
            //展示配置中有处理为空的url的默认图
            if (options.shouldShowImageForEmptyUri()) {
                //给imageAware设置这个默认图
                imageAware.setImageDrawable(options.getImageForEmptyUri(configuration.resources));
            } else {
                imageAware.setImageDrawable(null);
            }
            listener.onLoadingComplete(uri, imageAware.getWrappedView(), null);//回调展示结束
            return;
        }
        //获取当前需要下载的图片的size
        if (targetSize == null) {
            targetSize = ImageSizeUtils.defineTargetSizeForView(imageAware, configuration.getMaxImageSize());
        }
        //获取内存缓存的key(url_width_height)
        String memoryCacheKey = MemoryCacheUtils.generateKey(uri, targetSize);
        //添加到执行引擎cacheKeysForImageAwares的容器中
        engine.prepareDisplayTaskFor(imageAware, memoryCacheKey);
        listener.onLoadingStarted(uri, imageAware.getWrappedView());//回调展示开始
        //从内存中获取缓存的memoryCacheKey对应的bitmap
        Bitmap bmp = configuration.memoryCache.get(memoryCacheKey);
        //bitmap不为空，而且没有被回收
        if (bmp != null && !bmp.isRecycled()) {
            L.d(LOG_LOAD_IMAGE_FROM_MEMORY_CACHE, memoryCacheKey);
            //如果需要展示加载的进度，默认是不设置BitmapProcessor处理器的
            if (options.shouldPostProcess()) {
                //构造图片加载信息
                ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,
                        options, listener, progressListener, engine.getLockForUri(uri));
                //构造处理展示图片的任务
                ProcessAndDisplayImageTask displayTask = new ProcessAndDisplayImageTask(engine, bmp, imageLoadingInfo,
                        defineHandler(options));
                //如果需要同步加载
                if (options.isSyncLoading()) {
                    displayTask.run();//直接进行展现任务
                } else {
                    engine.submit(displayTask);//提交任务到加载引擎中
                }
            } else {
                //从内存中加载bitmap设置给imageAware，
                options.getDisplayer().display(bmp, imageAware, LoadedFrom.MEMORY_CACHE);
                //回调加载完成
                listener.onLoadingComplete(uri, imageAware.getWrappedView(), bmp);
            }
        } else {
            //如果需要展示加载的进度
            if (options.shouldShowImageOnLoading()) {
                //展示默认的加载中的图片
                imageAware.setImageDrawable(options.getImageOnLoading(configuration.resources));
            } else if (options.isResetViewBeforeLoading()) {
                imageAware.setImageDrawable(null);
            }
            //构造图片加载信息
            ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,
                    options, listener, progressListener, engine.getLockForUri(uri));
            //构造加载展示图片的任务
            LoadAndDisplayImageTask displayTask = new LoadAndDisplayImageTask(engine, imageLoadingInfo,
                    defineHandler(options));
            //如果需要同步加载
            if (options.isSyncLoading()) {
                displayTask.run();//直接进行展现任务
            } else {
                engine.submit(displayTask);//提交任务到加载引擎中
            }
        }
    }
    /***其他代码省略***/
}
```

`Imageloader`图片加载流程叙述：

>1. 校验配置；
>2. 赋值默认值（回调监听、图片展现配置）；
>3. 判断下载地址为空；
>3.1. 取消当前imageAware的图片展示任务；
>3.2. 如果图片展示配置有url为空的默认处理图那么加载默认图；
>4. 获取当前需要加载图的size；
>5. 获取缓存的key
>5.1. 根据key从内存缓存中获取bitmap，且bitmap有效；
>5.1.1. 如果需要展现加载进度，那么构造处理图片展示任务（ProcessAndDisplayImageTask）并执行（如果展现需要同步那么直接展示，否则任务提交到线程池）；
>5.1.2. 否则直接加载bitmap给当前的imageAware；
>5.2. 如果需要展现加载进度，那么获取图片展示配置中的加载状态资源进行展示，准备下一步加载真实图片资源；
>5.2.1. 构造加载展示图片任务（LoadAndDisplayImageTask）并执行（如果展现需要同步那么直接展示，否则任务提交到线程池）；



## 图片加载引擎

虽然叫做图片加载引起，但其实它仅仅只是一个任务分发处理器，负责分发`LoadAndDisplayImageTask`和`ProcessAndDisplayImageTask`给具体的线程池去执行，以及任务的暂停等操作。

```java
class ImageLoaderEngine {
    final ImageLoaderConfiguration configuration;//图片加载配置信息
    private Executor taskExecutor;//configuration.taskExecutor
    private Executor taskExecutorForCachedImages;//configuration.taskExecutorForCachedImages
    private Executor taskDistributor;//分配任务的线程池为newCachedThreadPool
    //imageview的hashcode和下载的key(url_width_height)
    private final Map<Integer, String> cacheKeysForImageAwares = Collections.synchronizedMap(new HashMap<Integer, String>());
    private final Map<String, ReentrantLock> uriLocks = new WeakHashMap<String, ReentrantLock>();
    private final AtomicBoolean paused = new AtomicBoolean(false);
    private final AtomicBoolean networkDenied = new AtomicBoolean(false);
    private final AtomicBoolean slowNetwork = new AtomicBoolean(false); 
    private final Object pauseLock = new Object();
    ImageLoaderEngine(ImageLoaderConfiguration configuration) {
        this.configuration = configuration;
        taskExecutor = configuration.taskExecutor;
        taskExecutorForCachedImages = configuration.taskExecutorForCachedImages;
        taskDistributor = DefaultConfigurationFactory.createTaskDistributor();
    }
    /***其他代码省略***/
}
```

任务提交处理，主要做了不同类型的任务分发给对应的任务执行的线程池：

```java
class ImageLoaderEngine {
    /** Submits task to execution pool */
    //执行从磁盘获取和网络上加载图片的任务
    void submit(final LoadAndDisplayImageTask task) {
        taskDistributor.execute(new Runnable() {
            @Override
            public void run() {
                File image = configuration.diskCache.get(task.getLoadingUri());
                //是否已经缓存在磁盘上
                boolean isImageCachedOnDisk = image != null && image.exists();
                initExecutorsIfNeed();
                if (isImageCachedOnDisk) {
                    taskExecutorForCachedImages.execute(task);
                } else {
                    taskExecutor.execute(task);
                }
            }
        });
    }
    /** Submits task to execution pool */
    //支持从缓存中加载图片的任务
    void submit(ProcessAndDisplayImageTask task) {
        initExecutorsIfNeed();
        taskExecutorForCachedImages.execute(task);
    }
    //任务线程池是否关闭，关闭则重新创建
    private void initExecutorsIfNeed() {
        if (!configuration.customExecutor && ((ExecutorService) taskExecutor).isShutdown()) {
            taskExecutor = createTaskExecutor();
        }
        if (!configuration.customExecutorForCachedImages && ((ExecutorService) taskExecutorForCachedImages)
                .isShutdown()) {
            taskExecutorForCachedImages = createTaskExecutor();
        }
    }
    //创建任务线程池
    private Executor createTaskExecutor() {
        return DefaultConfigurationFactory
                .createExecutor(configuration.threadPoolSize, configuration.threadPriority,
                configuration.tasksProcessingType);
    }
    /***其他代码省略***/
}
```

从`ImageLoader` 的`displayImage` 方法实现和 `ImageLoaderEngine` 的任务分发可以看出来，`ImageLoader` 主要有两种类型的任务 `ProcessAndDisplayImageTask` 和 `LoadAndDisplayImageTask` 。

## 处理和展示图片任务

```java
final class ProcessAndDisplayImageTask implements Runnable {
    /***部分代码省略***/
    @Override
    public void run() {
        L.d(LOG_POSTPROCESS_IMAGE, imageLoadingInfo.memoryCacheKey);
        //获取图片展现配置中的图片处理器
        BitmapProcessor processor = imageLoadingInfo.options.getPostProcessor();
        //获取处理过后的Biamtp
        Bitmap processedBitmap = processor.process(bitmap); 
        DisplayBitmapTask displayBitmapTask = new DisplayBitmapTask(processedBitmap, imageLoadingInfo, engine,
                LoadedFrom.MEMORY_CACHE);
        //如果isSyncLoading那么调用displayBitmapTask的run方法，否则如果handler不为空切换到主线程执行displayBitmapTask.run
        LoadAndDisplayImageTask.runTask(displayBitmapTask, imageLoadingInfo.options.isSyncLoading(), handler, engine);
    }
}
```

## 加载和展示图片任务

先看`LoadAndDisplayImageTask.runTask` 方法：

```java
static void runTask(Runnable r, boolean sync, Handler handler, ImageLoaderEngine engine) {
    if (sync) {//如果需要同步那么在当前线程执行
        r.run();
    } else if (handler == null) {//handler为空切换线程到taskDistributor线程池中执行
        engine.fireCallback(r);
    } else {
        handler.post(r);//切换到handler主线程执行
    }
}
```

### run（内存加载）

```java
final class LoadAndDisplayImageTask implements Runnable, IoUtils.CopyListener {
    /***部分代码省略***/
    @Override
    public void run() {
        //如果ImageLoader暂停执行任务（ImageLoader.pause方法被调用），那么当前线程进入等待被唤醒（ImageLoader.resume方法被调用）；
        //否则校验当前任务是否有效（校验目标ImageAware是否已经被回收，或者ImageAware需要加载的uri已经不是当前的uri）
        if (waitIfPaused()) return;
        //是否需要延迟加载（图片展示配置中如果delayBeforeLoading时间大于0）
        ////否则校验当前任务是否有效（校验目标ImageAware是否已经被回收，或者ImageAware需要加载的uri已经不是当前的uri）
        if (delayIfNeed()) return;
        //获取当前图片加载任务的锁
        ReentrantLock loadFromUriLock = imageLoadingInfo.loadFromUriLock;
        L.d(LOG_START_DISPLAY_IMAGE_TASK, memoryCacheKey);
        if (loadFromUriLock.isLocked()) {
            L.d(LOG_WAITING_FOR_IMAGE_LOADED, memoryCacheKey);
        }
        loadFromUriLock.lock();
        Bitmap bmp;
        try {
            //校验目标ImageAware是否已经被回收，或者ImageAware需要加载的uri已经不是当前的uri
            checkTaskNotActual();
            //先从内存缓存中获取对应的Bitmap
            bmp = configuration.memoryCache.get(memoryCacheKey);
            //如果bitmap被回收或者为空
            if (bmp == null || bmp.isRecycled()) {
                //尝试加载Bitmap（磁盘、资源、网络等）
                bmp = tryLoadBitmap();
                //加载失败直接返回
                if (bmp == null) return; // listener callback already was fired
                //校验目标ImageAware是否已经被回收，或者ImageAware需要加载的uri已经不是当前的uri
                checkTaskNotActual();
                //检验是否当前线程被打断
                checkTaskInterrupted();
                //根据图片展示配置项是否要进行保存前处理
                if (options.shouldPreProcess()) {
                    L.d(LOG_PREPROCESS_IMAGE, memoryCacheKey);
                    bmp = options.getPreProcessor().process(bmp);
                    if (bmp == null) {
                        L.e(ERROR_PRE_PROCESSOR_NULL, memoryCacheKey);
                    }
                }
                //是否需要对这个Bitmap进行内存缓存
                if (bmp != null && options.isCacheInMemory()) {
                    L.d(LOG_CACHE_IMAGE_IN_MEMORY, memoryCacheKey);
                    configuration.memoryCache.put(memoryCacheKey, bmp);
                }
            } else {
                loadedFrom = LoadedFrom.MEMORY_CACHE;
                L.d(LOG_GET_IMAGE_FROM_MEMORY_CACHE_AFTER_WAITING, memoryCacheKey);
            }
            //根据图片展示配置项是否要进行展示前处理
            if (bmp != null && options.shouldPostProcess()) {
                L.d(LOG_POSTPROCESS_IMAGE, memoryCacheKey);
                bmp = options.getPostProcessor().process(bmp);
                if (bmp == null) {
                    L.e(ERROR_POST_PROCESSOR_NULL, memoryCacheKey);
                }
            }
            //校验目标ImageAware是否已经被回收，或者ImageAware需要加载的uri已经不是当前的uri
            checkTaskNotActual();
            //检验是否当前线程被打断
            checkTaskInterrupted();
        } catch (TaskCancelledException e) {
            //进行失败处理
            fireCancelEvent();
            return;
        } finally {
            //释放锁
            loadFromUriLock.unlock();
        }
        //执行展示图片任务此处和ProcessAndDisplayImageTask任务后的展示逻辑相同
        DisplayBitmapTask displayBitmapTask = new DisplayBitmapTask(bmp, imageLoadingInfo, engine, loadedFrom);
        runTask(displayBitmapTask, syncLoading, handler, engine);
    }
    /***部分代码省略***/
}
```

**任务是否有效**：校验目标`ImageAware`是否已经被回收，或者`ImageAware`需要加载的`uri`已经不是当前的`uri`(被取消或者被代替)。

>1. 校验`ImageLoader`是否暂停执行任务和当前的任务是否有效；
>2. 是否需要进行延迟加载，延迟加载后校验当前是否任务有效；
>3. 获取当前图片加载任务的锁进行上锁；
>4. 校验当前是否任务有效后开始进行`Bitmap`获取；
>4.1 先从内存缓存中获取对应的`Bitmap`；
>4.2 获取`Bitmap` 为空获取已经被回收那么尝试加载`Bitmap`;
>4.2.1 `Bitmap`加载失败直接返回；
>4.2.2 校验当前是否任务有效；
>4.2.3 检验是否当前线程被打断；
>4.2.4 根据图片展示配置项是否要进行保存前处理；
>4.2.5 是否需要对这个`Bitmap`进行内存缓存;
>4.3 根据图片展示配置项是否要进行展示前处理
>4.4 校验当前是否任务有效；
>4.5 检验是否当前线程被打断；
>5. 释放锁;
>6. 执行展示图片任务;
### 加载图片

```java
final class LoadAndDisplayImageTask implements Runnable, IoUtils.CopyListener {
    /***部分代码省略***/
    private Bitmap tryLoadBitmap() throws TaskCancelledException {
        Bitmap bitmap = null;
        try {
            File imageFile = configuration.diskCache.get(uri);
            //从磁盘获取存储的图片
            if (imageFile != null && imageFile.exists() && imageFile.length() > 0) {
                L.d(LOG_LOAD_IMAGE_FROM_DISK_CACHE, memoryCacheKey);
                loadedFrom = LoadedFrom.DISC_CACHE;
                //校验目标ImageAware是否已经被回收，或者ImageAware需要加载的uri已经不是当前的uri
                checkTaskNotActual();
                bitmap = decodeImage(Scheme.FILE.wrap(imageFile.getAbsolutePath()));//进行图片解码
            }
            //bitmap为空，或者长宽小于0重新进行数据获取
            if (bitmap == null || bitmap.getWidth() <= 0 || bitmap.getHeight() <= 0) {
                L.d(LOG_LOAD_IMAGE_FROM_NETWORK, memoryCacheKey);
                loadedFrom = LoadedFrom.NETWORK;
                String imageUriForDecoding = uri;
                //是否需要缓存在磁盘上，如果需要进行磁盘缓存
                if (options.isCacheOnDisk() && tryCacheImageOnDisk()) {
                    imageFile = configuration.diskCache.get(uri);
                    if (imageFile != null) {
                        imageUriForDecoding = Scheme.FILE.wrap(imageFile.getAbsolutePath());
                    }
                }
                //校验目标ImageAware是否已经被回收，或者ImageAware需要加载的uri已经不是当前的uri
                checkTaskNotActual();
                bitmap = decodeImage(imageUriForDecoding);//进行图片解码
                //bitmap为空，或者长宽小于0进行异常处理
                if (bitmap == null || bitmap.getWidth() <= 0 || bitmap.getHeight() <= 0) {
                    fireFailEvent(FailType.DECODING_ERROR, null);
                }
            }
        } catch (）{
            /***异常处理省略***/
        }
        return bitmap;
    }
    /***部分代码省略***/
}
```

### 缓存图片到磁盘

```java
final class LoadAndDisplayImageTask implements Runnable, IoUtils.CopyListener {
    /***部分代码省略***/
    private boolean tryCacheImageOnDisk() throws TaskCancelledException {
        L.d(LOG_CACHE_IMAGE_ON_DISK, memoryCacheKey);
        boolean loaded;
        try {
            loaded = downloadImage();//下载图片，缓存到磁盘
            if (loaded) {
                int width = configuration.maxImageWidthForDiskCache;
                int height = configuration.maxImageHeightForDiskCache;
                if (width > 0 || height > 0) {
                    L.d(LOG_RESIZE_CACHED_IMAGE_FILE, memoryCacheKey);
                    //设置图片的大小，重新保存到磁盘
                    resizeAndSaveImage(width, height); // TODO : process boolean result
                }
            }
        } catch (IOException e) {
            L.e(e);
            loaded = false;
        }
        return loaded;
    }
    private boolean downloadImage() throws IOException {
        //通过download获取数据流
        InputStream is = getDownloader().getStream(uri, options.getExtraForDownloader());
        if (is == null) {
            L.e(ERROR_NO_IMAGE_STREAM, memoryCacheKey);
            return false;
        } else {
            try {//保存到磁盘
                return configuration.diskCache.save(uri, is, this);
            } finally {
                IoUtils.closeSilently(is);
            }
        }
    }
    /***部分代码省略***/
    //解码图像文件，压缩并重新保存（会覆盖之前的文件）
    private boolean resizeAndSaveImage(int maxWidth, int maxHeight) throws IOException {
        boolean saved = false;
        //获取磁盘缓存的文件
        File targetFile = configuration.diskCache.get(uri);
        if (targetFile != null && targetFile.exists()) {
            ImageSize targetImageSize = new ImageSize(maxWidth, maxHeight);
            //生成新的配置
            DisplayImageOptions specialOptions = new DisplayImageOptions.Builder().cloneFrom(options)
                    .imageScaleType(ImageScaleType.IN_SAMPLE_INT).build();
            ImageDecodingInfo decodingInfo = new ImageDecodingInfo(memoryCacheKey,
                    Scheme.FILE.wrap(targetFile.getAbsolutePath()), uri, targetImageSize, ViewScaleType.FIT_INSIDE,
                    getDownloader(), specialOptions);
            //对图像文件做解码
            Bitmap bmp = decoder.decode(decodingInfo);
            //压缩文件
            if (bmp != null && configuration.processorForDiskCache != null) {
                L.d(LOG_PROCESS_IMAGE_BEFORE_CACHE_ON_DISK, memoryCacheKey);
                bmp = configuration.processorForDiskCache.process(bmp);
                if (bmp == null) {
                    L.e(ERROR_PROCESSOR_FOR_DISK_CACHE_NULL, memoryCacheKey);
                }
            }
            //重新保存，覆盖之前的uri对应的缓存文件
            if (bmp != null) {
                saved = configuration.diskCache.save(uri, bmp);
                bmp.recycle();
            }
        }
        return saved;
    }
}
```

## 其他

- 取消当前`imageview`对应的任务

```java
public void cancelDisplayTask(ImageView imageView) {
    engine.cancelDisplayTaskFor(new ImageViewAware(imageView));
}
```

- 拒绝或允许`ImageLoader`从网络下载图像

```java
public void denyNetworkDownloads(boolean denyNetworkDownloads) {
    engine.denyNetworkDownloads(denyNetworkDownloads);
}
```

- 设置`ImageLoader`是否使用`FlushedInputStream`进行网络下载的选项

```java
public void handleSlowNetwork(boolean handleSlowNetwork) {
    engine.handleSlowNetwork(handleSlowNetwork);
}
```

- 暂停ImageLoader。在ImageLoader#resume恢复之前，不会执行所有新的“加载和显示”任务。
- 已经运行的任务不会暂停。

```java
public void pause() {
    engine.pause();
}
```

- 恢复等待的“加载和显示”任务

```java
public void resume() {
    engine.resume();
}
```

- 取消所有正在运行和计划的显示图像任务
- 还可以继续使用`ImageLoader`

```java
public void stop() {
    engine.stop();
}
```

- 取消所有正在运行和计划的显示图像任务
- 销毁所有配置，重新使用ImageLoader需要进行初始化

```java
public void destroy() {
    if (configuration != null) L.d(LOG_DESTROY);
    stop();
    configuration.diskCache.close();
    engine = null;
    configuration = null;
}
```

- 为了更友好的用户体验，在列表滑动过程中可以暂停加载（调用`pause`和`resume`）；
- RGB_565代替ARGB_8888，减少占用内存；
- 使用`memoryCache(new WeakMemoryCache())` 将内存中的`Bitmap` 变为软引用；


文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！