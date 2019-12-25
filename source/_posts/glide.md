---
title: Glide源码阅读理解一小时
date: 2019-12-20 19:35:00
tags: [Glide]
categories: Android
description: "这篇图、文、表、代码一起组成的Glide源码分析。这篇Glide的代码分析量可以说至少是ImageLoader的3倍多，本来想对Glide代码进行拆分，细化每个部分进行讲解这个每个部分讲的更加清楚一些。但最终还是打算整体一篇文章讲完，因为我觉得整体性的学习能更深的的了解到Glide的框架的设计之美。阅读本文需要大量的时间，最好选择对应的目录进行逐步阅读。 "
---

# 前言

这篇**图、文、表、代码**一起组成的 `Glide` 源码分析的文章是在上一篇文章 [Android-Universal-Image-Loader源码分析](https://dandanlove.blog.csdn.net/article/details/103256724) 中之后的又一篇图片加载框架源码解析，它也具备了 `ImageLoader` 中讲述了`Android`一个图片加载库所需要的一些基础必备的：`MemoryCahce`、`DiskCahce` `Decoder` `DownLoader` 和`Executor` 等部分。这篇 `Glide`  的代码分析量可以说至少是 `ImageLoader` 的3倍多，本来想对 `Glide` 代码进行拆分，细化每个部分进行讲解这个每个部分讲的更加清楚一些。但最终还是打算整体一篇文章讲完，因为我觉得整体性的学习能更深的的了解到 `Glide` 的框架的设计之美。

> 本篇文章讲述的`Glide` 相关知识比较多，阅读完需要大量的时间。所以我们按需分配，根据目录寻找自己需要的知识点进行查看。


# Glide介绍

[Glide的Git地址：https://github.com/bumptech/glide](https://github.com/bumptech/glide)

[简体中文文档：https://muyangmin.github.io/glide-docs-cn/](https://muyangmin.github.io/glide-docs-cn/)

[Glide's documentation：https://bumptech.github.io/glide/](https://bumptech.github.io/glide/)

## 关于Glide

`Glide`是一个快速高效的`Android`图片加载库，注重于平滑的滚动。`Glide`提供了易用的`API`，高性能、可扩展的图片解码管道（`decode pipeline`），以及自动的资源池技术。

`Glide` 支持拉取，解码和展示视频快照，图片，和GIF动画。`Glide`的Api是如此的灵活，开发者甚至可以插入和替换成自己喜爱的任何网络栈。默认情况下，`Glide`使用的是一个定制化的基于`HttpUrlConnection`的栈，但同时也提供了与`Google Volley`和`Square OkHttp`快速集成的工具库。

虽然`Glide` 的主要目标是让任何形式的图片列表的滚动尽可能地变得更快、更平滑，但实际上，`Glide`几乎能满足你对远程图片的拉取/缩放/显示的一切需求。

## Glide性能

`Glide` 充分考虑了`Android`图片加载性能的两个关键方面：

- 图片解码速度
- 解码图片带来的资源压力

为了让用户拥有良好的App使用体验，图片不仅要快速加载，而且还不能因为过多的主线程I/O或频繁的垃圾回收导致页面的闪烁和抖动现象。

`Glide`使用了多个步骤来确保在`Android`上加载图片尽可能的快速和平滑：

- 自动、智能地下采样(`downsampling`)和缓存(`caching`)，以最小化存储开销和解码次数；
- 积极的资源重用，例如字节数组和`Bitmap`，以最小化昂贵的垃圾回收和堆碎片影响；
- 深度的生命周期集成，以确保仅优先处理活跃的`Fragment`和`Activity`的请求，并有利于应用在必要时释放资源以避免在后台时被杀掉。

## GlideAPI

Glide 使用简明的流式语法API，这是一个非常棒的设计，因为它允许你在大部分情况下一行代码搞定需求：

```java
Glide.with(fragment)
    .load(url)
    .into(imageView);
```

上述是`Fragmeng`中`Glide`将一张网络图片显示到`ImageView`的代码，下面源码分析的时候我们也会用这段代码进行分析，看看这么简单的`API`到底是怎么实现的。

# Glide源码分析

我们学习和了解一些框架主要不是看它某个功能的具体实现，主要是学习框架结构搭建和框架中模块的设计与实现。当然每个人的对每个框架的理解都各不相同，不过没关系我们可以多学习多总结，慢慢培养我们自己的框架结构意识。这个在我们平时开发过程中对我们帮助非常大。

<center>![在这里插入图片描述](https://img-blog.csdnimg.cn/20191220185346375.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)</center>

上图是对我`Glide`的一个总结。

## Glide接入

`Glide`的用法网上有很多文章讲述的都非常好，这里不再进行讲述。这块主要想通过`Glide`的配置来分析`Glide`的运行机制。

我们在使用`Glide`的时候都会使用注解`@GlideModule` 实现`AppGlideModule` 或者 `GeneratedAppGlideModule` ，生成一个类名为`GeneratedAppGlideModuleImpl` 它是`Glide` 模块的代理。 在`GeneratedAppGlideModuleImpl` 会包含我们自定义的`GlideModel`。

下面为我们接入项目的`Glide`配置：

> 实现Glide对缓存的配置

```java
@GlideModule
public final class GlideModuleConfig extends AppGlideModule {
    @Override
    public void applyOptions(@NonNull Context context, @NonNull GlideBuilder builder) {
        super.applyOptions(context, builder);
        long memoryCacheSizeBytes = 1024 * 1024 * 20;
        builder.setMemoryCache(new LruResourceCache(memoryCacheSizeBytes));
        long diskCacheSizeBytes = 1024 * 1024 * 100;
        builder.setDiskCache(new InternalCacheDiskCacheFactory(context, diskCacheSizeBytes));
    }
}
```

> 实现Okhttp的接入

```java
@GlideModule
public final class OkHttpLibraryGlideModule extends LibraryGlideModule {
    @Override
    public void registerComponents(
            @NonNull Context context, @NonNull Glide glide, @NonNull Registry registry) {
        //将GlideUrl数据转为InputStream的ModelLoader替换为Okhttp
        registry.replace(GlideUrl.class, InputStream.class, new OkHttpUrlLoader.Factory());
    }
}
```

我们编译之后生成的`GeneratedAppGlideModuleImpl`：

```java
final class GeneratedAppGlideModuleImpl extends GeneratedAppGlideModule {
    private final GlideModuleConfig appGlideModule;
    public GeneratedAppGlideModuleImpl(Context context) {
      appGlideModule = new GlideModuleConfig();
    }
    //GlideBuilder的配置项进行应用
    @Override
    public void applyOptions(@NonNull Context context, @NonNull GlideBuilder builder) {
      appGlideModule.applyOptions(context, builder);
    }
    //注册自定义GlideModule
    @Override
    public void registerComponents(@NonNull Context context, @NonNull Glide glide,
        @NonNull Registry registry) {
      new OkHttpLibraryGlideModule().registerComponents(context, glide, registry);
      appGlideModule.registerComponents(context, glide, registry);
    }
    /***部分代码省略***/
}
```

下面将`GeneratedAppGlideModuleImpl`的使用。

## Glide初始化

```java
public class Glide implements ComponentCallbacks2 {
    /**
    * Get the singleton.
    * @return the singleton
    */
    @NonNull
    public static Glide get(@NonNull Context context) {
        if (glide == null) {
            //
            GeneratedAppGlideModule annotationGeneratedModule = 
                getAnnotationGeneratedGlideModules(context.getApplicationContext());
            synchronized (Glide.class) {
                if (glide == null) {
                    checkAndInitializeGlide(context, annotationGeneratedModule);
                }
            }
        }
        return glide;
    }
    //通过Java反射机制获取通过注解生成的GeneratedAppGlideModuleImpl
    private static GeneratedAppGlideModule getAnnotationGeneratedGlideModules(Context context) {
        GeneratedAppGlideModule result = null;
        try {
        Class<GeneratedAppGlideModule> clazz =
            (Class<GeneratedAppGlideModule>)
                Class.forName("com.bumptech.glide.GeneratedAppGlideModuleImpl");
        result =
            clazz.getDeclaredConstructor(Context.class).newInstance(context.getApplicationContext());
        } catch (ClassNotFoundException e) {
        /***部分代码省略**/
        }
        return result;
    }
}
```

`Glide`是个单例，初始化的时候需要注解生成的`GeneratedAppGlideModuleImpl` 。

```java
public class Glide implements ComponentCallbacks2 {
    @GuardedBy("Glide.class")
    private static void initializeGlide(
        @NonNull Context context, @Nullable GeneratedAppGlideModule generatedAppGlideModule) {
        initializeGlide(context, new GlideBuilder(), generatedAppGlideModule);
    }
    @GuardedBy("Glide.class")
    @SuppressWarnings("deprecation")
    private static void initializeGlide(
        @NonNull Context context,
        @NonNull GlideBuilder builder,
        @Nullable GeneratedAppGlideModule annotationGeneratedModule) {
        Context applicationContext = context.getApplicationContext();
        List<com.bumptech.glide.module.GlideModule> manifestModules = Collections.emptyList();
        //如果Manifest文件中配置了GlideModule，并且annotationGeneratedModule允许Manifest文件中配置生效，那么需要解析出来
        if (annotationGeneratedModule == null || annotationGeneratedModule.isManifestParsingEnabled()) {
            manifestModules = new ManifestParser(applicationContext).parse();
        }
        //如果annotationGeneratedModule配置了扩展模块需要解析出来，并且不能与果Manifest中的重复
        if (annotationGeneratedModule != null
            && !annotationGeneratedModule.getExcludedModuleClasses().isEmpty()) {
            Set<Class<?>> excludedModuleClasses = annotationGeneratedModule.getExcludedModuleClasses();
            Iterator<com.bumptech.glide.module.GlideModule> iterator = manifestModules.iterator();
            while (iterator.hasNext()) {
                com.bumptech.glide.module.GlideModule current = iterator.next();
                if (!excludedModuleClasses.contains(current.getClass())) {
                    continue;
                }
                /***省略部分日志***/
                iterator.remove();
            }
        }
        /***省略部分日志***/
        //获取Request的构造管理工厂类
        RequestManagerRetriever.RequestManagerFactory factory =
            annotationGeneratedModule != null
                ? annotationGeneratedModule.getRequestManagerFactory()
                : null;
        builder.setRequestManagerFactory(factory);
        //Manifest中的配置项进行应用
        for (com.bumptech.glide.module.GlideModule module : manifestModules) {
            module.applyOptions(applicationContext, builder);
        }
        //annotationGeneratedModule进行配置项的应用
        if (annotationGeneratedModule != null) {
            annotationGeneratedModule.applyOptions(applicationContext, builder);
        }
        //构造Glide
        Glide glide = builder.build(applicationContext);
        //Manifest中的组件进行注册
        for (com.bumptech.glide.module.GlideModule module : manifestModules) {
            try {
                module.registerComponents(applicationContext, glide, glide.registry);
            } catch (AbstractMethodError e) {
                /***省略部分日志***/
            }
        }
        //annotationGeneratedModule中的组件进行注册
        if (annotationGeneratedModule != null) {
            annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
        }
        //application注册Glide组件
        applicationContext.registerComponentCallbacks(glide);
        Glide.glide = glide;
    }
}
```

1. 获取`Manifest`的`GlideModule`，反射构造；
2. 获取`GeneratedAppGlideModule`中扩展模块，如果包含`Manifest`的扩展那么进行删除；
3. 获取`RequestManagerFactroy`;
4. `Manifest.applyoptions`;
5. `GeneratedAppGlideModule.applytions`;
6. 构建`Glide`;
7. `Manifest.registerComponents`;
8. `GeneratedAppGlideModule.registerComponents`;

## Glide和Engine构造

```java
public class Glide implements ComponentCallbacks2 {
    private final Engine engine;//负责启动负载以及管理活动和缓存的资源。
    private final BitmapPool bitmapPool;//Bitmap的缓存池（LruBitmapPool）
    private final MemoryCache memoryCache;//Resource缓存池（LruResourceCache）
    private final GlideContext glideContext;
    private final Registry registry;
    private final ArrayPool arrayPool;//存储大小可变的数组的缓存池(LruArrayPool)
    //一组静态方法用来创建一个新的RequestManager或者从已经存在的activity和fragment中获取
    private final RequestManagerRetriever requestManagerRetriever;
    private final ConnectivityMonitorFactory connectivityMonitorFactory;
    new Glide(
        @NonNull context,//上下文环境
        @NonNull engine,//任务执行引擎
        @NonNull memoryCache,//内存资源缓存
        @NonNull bitmapPool,//内存Bitmap缓存
        @NonNull arrayPool,//数据缓存池
        @NonNull requestManagerRetriever,//RequestManager管理集合
        @NonNull connectivityMonitorFactory,//网络监听器的生产工厂
        int logLevel,//log日志等级，默认为Log.info=4
        @NonNull defaultRequestOptionsFactory,//默认的RequestOptions生产工厂
        @NonNull defaultTransitionOptions,//默认的资源展现过渡配置容器，，默认map大小为0
        @NonNull defaultRequestListeners,//在图像加载时的监听器数组，默认数组大小为0
        boolean isLoggingRequestOriginsEnabled,//是否需要请求日志
        boolean isImageDecoderEnabledForBitmaps,//在安卓P或更高版本进行解码bitmap
        int hardwareBitmapFdLimit,//700,
        int minHardwareDimension){//128,
        /***部分代码省略***/
    }
}
```

`Glide`构造的时候需要构造`Engine`:

```java
public class Engine
    implements EngineJobListener,
        MemoryCache.ResourceRemovedListener,
        EngineResource.ResourceListener {
    /***部分代码省略***/
    public Engine(
        MemoryCache memoryCache,//内存缓存
        DiskCache.Factory diskCacheFactory,//磁盘缓存
        //磁盘缓存执行器，该线程池的核心线程数和最大线程数为1
        GlideExecutor diskCacheExecutor,
        //检索资源尚未在缓存中，该线程池的核心线程数和最大线程数为cpu内核数量，最大为4
        GlideExecutor sourceExecutor,
        ////newScheduledThreadPool，核心线程数为0，用来执行网络操作
        GlideExecutor sourceUnlimitedExecutor,
        //加载动画线程池，加载动画图像的帧时使用，尤其是GitDrawable，该线程池的核心线程数和最大线程数为1或2（cpu内核数量>=4）
        GlideExecutor animationExecutor,
        //活动资源是否允许被保留，默认为false
        boolean isActiveResourceRetentionAllowed) {
            /***部分代码省略***/
    }
    /***部分代码省略***/
}
```

在阅读源码的时候`Glide.java`的构造方法，除过基础的赋值操作之前就剩下大量的注册。注册的所有组件都由`Registry`进行管理。

## Register

> `Register` ：管理组件注册以扩展或替换`Glide`的默认加载，解码和编码逻辑。

```java
public class Registry {
    public static final String BUCKET_GIF = "Gif";
    public static final String BUCKET_BITMAP = "Bitmap";
    public static final String BUCKET_BITMAP_DRAWABLE = "BitmapDrawable";
    private static final String BUCKET_PREPEND_ALL = "legacy_prepend_all";
    private static final String BUCKET_APPEND_ALL = "legacy_append";
    //维护{@link ModelLoader}的有序放置以及它们处理的模型和数据类型,从最高优先级到最低优先级的顺序。
    private final ModelLoaderRegistry modelLoaderRegistry;
    //编码数据容器，有序列表
    private final EncoderRegistry encoderRegistry;
    //包含能够解码任意数据类型的{@link ResourceDecoder}的有序列表
    //分为从最高优先级解码器到最低优先级解码器的任意资源类型。
    private final ResourceDecoderRegistry decoderRegistry;
    //包含能够编码任意资源*类型的{@link ResourceEncoder}的有序列表。
    private final ResourceEncoderRegistry resourceEncoderRegistry;
    //DataRewinder的生产工厂
    private final DataRewinderRegistry dataRewinderRegistry;
    //注册和转化ResourceTranscoder
    private final TranscoderRegistry transcoderRegistry;
    //包含能够解析图像标题的{@link ImageHeaderParser}的无序列表。
    private final ImageHeaderParserRegistry imageHeaderParserRegistry;
    //维护“模型+资源”类的高速缓存到一组已注册资源类的集合，这些资源类是可以从模型类中解码的资源类的子类。
    private final ModelToResourceClassCache modelToResourceClassCache = new ModelToResourceClassCache();
    //维护数据，资源和转码类的缓存
    private final LoadPathCache loadPathCache = new LoadPathCache();
    //异常列表维护容器
    private final Pool<List<Throwable>> throwableListPool = FactoryPools.threadSafeList();
    /***部分代码省略***/
}
```

下面我们看看`Glide`默认注册的各种组件。

###  Encoder

**Encoder**：用于将数据编码成文件。

```java
//用于将数据写入某些持久性数据存储的接口，例如文件
public interface Encoder<T> {
    //将给定数据写入给定输出流，如果写入完成，则返回True
    boolean encode(@NonNull T data, @NonNull File file, @NonNull Options options);
}
```


> `EncoderRegistry` ：包含能够编码任意数据类型的`Encoder`的有序列表。

| DataClass  | Encoder           |
| ---------- | ----------------- |
| ByteBuffer | ByteBufferEncoder |
| InputStream | StreamEncoder|
| Bitmap | BitmapEncoder |
| BitmapDrawable | BitmapDrawableEncoder |
| GifDrawable | GifDrawableEncoder |

```java
//用于将数据从资源写入某些持久性数据存储的接口，例如文件
public interface ResourceEncoder<T> extends Encoder<Resource<T>> {
    //获取对应的策略模式
    @NonNull
    EncodeStrategy getEncodeStrategy(@NonNull Options options);
}
//ResourceEncoder进行资源编码缓存枚举
public enum EncodeStrategy {
    //将资源的原始未修改数据写入磁盘,不包括采样和转化
    SOURCE,
    //将资源的解码，下采样和转换后的数据写入磁盘。
    TRANSFORMED,
    /** Will write no data. */
    NONE,
}
```

> `ResourceEncoderRegistry` ：包含能够编码任意资源类型的`ResourceEncoder`的有序列表。

| ResourceClass  | ResourceEncoder       |
| -------------- | --------------------- |
| BitmapDrawable | BitmapDrawableEncoder |
| GifDrawable    | GifDrawableEncoder    |
| Bitmap         | BitmapEncoder         |

### Decoder

**Decoder**：解码给定的资源类型文件。

```java
/**
 *用于解码资源的接口。
 *@param <T>将从中解码资源的类型（文件，InputStream等）。
 *@param <Z>解码资源的类型（位图，可绘制等）。
 */
public interface ResourceDecoder<T, Z> {
    /**
     *如果此解码器能够使用给定的源解码给定的源，则返回true选项，否则为false。
     *解码器应尽最大努力快速确定是否可能够解码数据，但不应尝试完全读取给定的数据。
     *典型的实现将检查文件头，以确保它们与解码器期望的内容匹配句柄（即GIF解码器应验证图像是否包含GIF标头块)。
     */
    boolean handles(@NonNull T source, @NonNull Options options) throws IOException;
    //从给定的数据返回已解码的资源；如果无法解码任何资源，则返回null。
    //注意width和height参数仅是提示，没有要求解码后的资源与给定尺寸完全匹配。
    //一个典型的用例将使用目标尺寸来确定对位图进行降采样的量，以避免分配过多。
    @Nullable
    Resource<Z> decode(@NonNull T source, int width, int height, @NonNull Options options)throws IOException;
}
```

> `ResourceDecoderRegistry` ：包含`ResourceDecoder`的有序列表，这些列表可以将任意数据类型解码为从最高优先级解码器到最低优先级解码器的任意资源类型。

| Bucket | DataClass   | ResourceClass | ResourceDecoder       |
| ------ | ----------- | ------------- | --------------- |
| Bitmap|ByteBuffer|Bitmap| ByteBufferBitmapImageDecoderResourceDecoder|
|Bitmap|InputStream|Bitmap| InputStreamBitmapImageDecoderResourceDecoder|
| Bitmap | ParcelFileDescriptor | Bitmap| VideoDecoder |
| Bitmap | AssetFileDescriptor  | Bitmap  | VideoDecoder |
| Bitmap | Bitmap   | Bitmap  | UnitBitmapDecoder  |
| Bitmap | GifDecoder  | Bitmap   | GifFrameResourceDecoder |


| Bucket | DataClass| ResourceClass   | ResourceDecoder   |
| --------------- | ---------- | -------- | --------------- |
| BitmapDrawables |ByteBuffer| BitemapDrawable | BitmapDrawableDecoder |
| BitmapDrawables| InputStream| BitemapDrawable | BitmapDrawableDecoder |
|BitmapDrawables|ParcelFileDescriptor|BitemapDrawable|BitmapDrawableDecoder|

| Bucket | DataClass  | ResourceClass | ResourceDecoder      |
| ------ | ---------- | ------------- | -------------------- |
| Gif    | ByteBuffer | GifDrawable   | ByteBufferGifDecoder |


| Bucket | DataClass | ResourceClass | ResourceDecoder         |
| ------ | --------- | ------------- | ----------------------- |
| All    | Uri       | Drawable      | ResourceDrawableDecoder |
| All    | Uri       | Bitmap        | ResourceBitmapDecoder   |
| All    | File      | File          | FileDecoder             |
| All    | Drawable  | Drawable      | UnitDrawableDecoder     |

### Transcoder

**Transcoder**：将一种类型的资源转码为另一种类型的资源。

```java
/**
 * 将一种类型的资源转码为另一种类型的资源。
 * @param <Z> 要被转码的资源类型
 * @param <R> 需要转成的资源类型
 */
public interface ResourceTranscoder<Z, R> {
    //将给定资源转码为新资源类型并返回新资源。
    @Nullable
    Resource<R> transcode(@NonNull Resource<Z> toTranscode, @NonNull Options options);
}
```

> `TranscoderRegistry` ：该类允许`ResourceTranscoder`在它们之间进行转换的类中进行注册和检索。


| ResourceClass | TranscoeClass  | ResourceTranscoder         |
| ------------- | -------------- | -------------------------- |
| Bitmap        | BitmapDrawable | BitmapDrawableTranscoder   |
| Bitmap        | byte[]         | BitmapBytesTranscoder      |
| Drawable      | byte[]         | DrawableBytesTranscoder    |
| GifDrawable   | byte[]         | GifDrawableBytesTranscoder |

### ModelLoader

`ModelLoader` 是`Glide` 比较核心的类，主要是用来加载数据源`Model` 中的数据。

一般加载资源类型有`Bitmap` 、`String(网络图片、本地图片、资源图片)` 、`Uri(网络图片、本地图片、资源图片)` 、`URL(网络图片)` 、`Integer(资源图片)` 和`File(本地文件)`等。

`Glide` 为每中资源类型设计了对应的`ModelLoaderFactory` ，每种`ModelLoaderFactory`对应一种`ModleLoader`。

资源类型可以相互转化，比如`String`转`URL`。所以`ModelLoader` 内部也是可以相互进行代理。

<center>![在这里插入图片描述](https://img-blog.csdnimg.cn/20191220185627504.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)</center>

```java
//com.bumptech.glide.load.model.MultiModelLoaderFactory
//通过仅在以下位置创建加载器来避免堆栈溢出递归地创建模型加载器
//递归请求（如果尚未在链中的早期创建）。例如：Uri加载程序可以转换为另一个模型，而后者又可以转换回Uri。
//尽管原始Uri加载程序不会提供给中间模型加载程序，其他的Uri装载程序也会。
synchronized <Model> List<ModelLoader<Model, ?>> build(@NonNull Class<Model> modelClass) {
    try {
        List<ModelLoader<Model, ?>> loaders = new ArrayList<>();
        for (Entry<?, ?> entry : entries) {
            if (alreadyUsedEntries.contains(entry)) {//去重
                continue;
            }
            if (entry.handles(modelClass)) {
                alreadyUsedEntries.add(entry);
                //递归调用MultiModelLoaderFactory.build
                loaders.add(this.<Model, Object>build(entry));
                alreadyUsedEntries.remove(entry);
            }
        }
        return loaders;
    } catch (Throwable t) {
        alreadyUsedEntries.clear();
        throw t;
    }
}
//调用ModelLoaderFactory.build
private <Model, Data> ModelLoader<Model, Data> build(@NonNull Entry<?, ?> entry) {
    return (ModelLoader<Model, Data>) Preconditions.checkNotNull(entry.factory.build(this));
}
```

比如`ModelClass` 是 `String` 类型，我们遍历找到 `StreamFactory` ：

```java
public static class StreamFactory implements ModelLoaderFactory<String, InputStream> {
    @NonNull
    @Override
    public ModelLoader<String, InputStream> build(@NonNull MultiModelLoaderFactory multiFactory) {
        return new StringLoader<>(multiFactory.build(Uri.class, InputStream.class));
    }
    @Override
    public void teardown() {
      // Do nothing.
    }
}
```

这个时候有会进行一次递归：

```java
//com.bumptech.glide.load.model.MultiModelLoaderFactory
public synchronized <Model, Data> ModelLoader<Model, Data> build(
      @NonNull Class<Model> modelClass, @NonNull Class<Data> dataClass) {
    try {
        List<ModelLoader<Model, Data>> loaders = new ArrayList<>();
        boolean ignoredAnyEntries = false;
        for (Entry<?, ?> entry : entries) {
            if (alreadyUsedEntries.contains(entry)) {//去重
                ignoredAnyEntries = true;
                continue;
            }
            if (entry.handles(modelClass, dataClass)) {
                alreadyUsedEntries.add(entry);
                //递归调用MultiModelLoaderFactory.build
                loaders.add(this.<Model, Data>build(entry));
                alreadyUsedEntries.remove(entry);
            }
        }
        if (loaders.size() > 1) {//数量大于1的时候构建MultiModelLoader存储loaders
            return factory.build(loaders, throwableListPool);
        } else if (loaders.size() == 1) {
            return loaders.get(0);
        } else {
            //如果递归导致没有可用的加载程序，请避免崩溃。该断言应该捕获完全未处理的类型，递归可能意味着未在某处处理子类型进入堆栈，这通常是可以的。 
            if (ignoredAnyEntries) {
                return emptyModelLoader();
            } else {
                throw new NoModelLoaderAvailableException(modelClass, dataClass);
            }
        }
    } catch (Throwable t) {
        alreadyUsedEntries.clear();
        throw t;
    }
}
```

所以我们根据`Model`的类型从注册到`Glide`的`ModelLoaderFactory`中寻找匹配项，这个查找是按照添加的顺序进行遍历。找到对应的`ModelLoaderFactory` 在生成 `ModelLoader`的时候可能会继续寻找它的代理的`ModelLoader`，直到不需要代理为止，我们会对这个结果**去重**然后如果结果数量大于**1**那么会生成`MultiModelLoader` 存储这一组 `ModelLoader` 。如果这组中的 `ModelLoader` 中还包括 `MultiModelLoader` 那么这个 `MultiModelLoader` 内还会有一组 `ModelLoader` 。

下面为**ModelLoader**涉及到的类:

> `DataSource` ：表示某些检索到的数据的来源。

```java
public enum DataSource {
    //表示数据可能是从设备本地检索的，尽管可能已经是通过可能已从远程源获取数据的内容提供者获得的。
    LOCAL,
    //表示从设备以外的远程源检索到数据。
    REMOTE,
    //表示从设备缓存中检索的数据未经修改。
    DATA_DISK_CACHE,
    //表示数据是从设备上缓存中的已修改内容中检索到的。
    RESOURCE_DISK_CACHE,
    //表示已从内存缓存中检索数据。
    MEMORY_CACHE,
}
```

> `DataFetcher` ：加载资源的数据。

```java
/**
 * 懒惰地检索可用于加载资源的数据。
 * @param <T> 要加载的数据类型  (InputStream, byte[], File etc).
 */
public interface DataFetcher<T> {
    //获取可以从中解码资源的数据。
    //没有数据的时候调用，异步线程执行，io不会阻塞
    void loadData(@NonNull Priority priority, @NonNull DataCallback<? super T> callback);
    //清理或回收此数据获取器使用的任何资源。
    void cleanup();
    //取消任务
    void cancel();
    //返回此访存器将尝试获取的数据的类。
    @NonNull
    Class<T> getDataClass();
    //返回要处理的资源数据类型
    @NonNull
    DataSource getDataSource();
}
```

> `ModelLoaderFactory` ：用于为给定模型类型创建`ModelLoader`的接口。使用设计模式之工厂模式，让每一个`ModelLoader` 对应一个工厂。

```java
public interface ModelLoaderFactory<T, Y> {
    //为此模型类型构建一个具体的ModelLoader。
    @NonNull
    ModelLoader<T, Y> build(@NonNull MultiModelLoaderFactory multiFactory);
    //一种生命周期方法，该工厂即将被替换时将被调用。
    void teardown();
}
```

> `ModelLoader` ：用于将任意复杂的数据模型转换为具体的数据类型。

```java
/**
 *工厂接口，用于将任意复杂的数据模型转换为具体的数据类型,DataFetcher可以使用来获取由模型。
 *此接口有两个目标：
 *1.将特定模型转换为可以被解码为资源。
 *2.允许将模型与视图的尺寸组合以获取模型的资源具体尺寸。
 *这不仅避免了必须在xml和代码中重复尺寸，以便确定具有不同密度的设备上视图的大小，
 *但也允许您使用布局权重或通过编程方式放置视图的尺寸而不会强迫您获取通用资源大小。
 *您获取的资源越小，使用的带宽和电池寿命越少，并且越低每个资源的内存占用量。
 *
 *@param <Model> 模型的类型。
 *@param <Data> 可以使用的数据类型ResourceDecoder来解码资源。
 */
public interface ModelLoader<Model, Data> {
    //返回可以解码model的LoadData来进行资源解码
    //注意-如果无法返回有效的数据提取程序（例如，如果模型的URL为空），然后可以从此方法返回空数据获取程序。
    @Nullable
    LoadData<Data> buildLoadData(@NonNull Model model, int width, int height, @NonNull Options options);
    //当前的数据模型是否能被处理
    boolean handles(@NonNull Model model);
}
```

> `LoadData` ：加载的资源的`Key`和对应加载该资源的`DataFetcher` 的包装对象。

```java
//包含一组标识负载源的keys，指向等效数据的备用缓存键以及DataFetcher，可用于获取在缓存中找不到的数据。
class LoadData<Data> {
  public final Key sourceKey;//数据源标识
  public final List<Key> alternateKeys;//备用的标识
  public final DataFetcher<Data> fetcher;//可用于加载资源的数据
  public LoadData(@NonNull Key sourceKey, @NonNull DataFetcher<Data> fetcher) {
    this(sourceKey, Collections.<Key>emptyList(), fetcher);
  }
  public LoadData(
    @NonNull Key sourceKey,
    @NonNull List<Key> alternateKeys,
    @NonNull DataFetcher<Data> fetcher) {
    this.sourceKey = Preconditions.checkNotNull(sourceKey);
    this.alternateKeys = Preconditions.checkNotNull(alternateKeys);
    this.fetcher = Preconditions.checkNotNull(fetcher);
  }
}
```

> `ModelLoaderRegistry` ：维护`ModelLoader`的有序放置以及它们处理的模型和数据类型,从最高优先级到最低优先级的顺序。

| ModelClass | DataClass     | ModelLoaderFactory      |
| ---------- | -------------------- | ------------------- |
| Bitmap     | Bitmap               | UnitModelLoader.Factory           |
| GifDecoder | GifDecoder           | UnitModelLoader.Factory           |
| File       | ByteBuffer           | ByteBufferFileLoader.Factory      |
| File       | InputStream          | FileLoader.StreamFactory          |
| File       | ParcelFileDescriptor | FileLoader.FileDescriptorFactory  |
| File       | File                 | UnitModelLoader.Factory           |
| int        | InputStream          | ResourceLoader.StreamFactory      |
| int   | ParcelFileDescriptor | ResourceLoader.FileDescriptorFactory |
|int|Uri|ResourceLoader.UriFactory|
|int|AssetFileDescriptor|ResourceLoader.AssetFileDescriptorFactory|
| Integer  | InputStream | ResourceLoader.StreamFactory      |
|Integer|ParcelFileDescriptor|ResourceLoader.FileDescriptorFactory|
|Integer|Uri|ResourceLoader.UriFactory|
|Integer|AssetFileDescriptor|ResourceLoader.AssetFileDescriptorFactory|
|String|InputStream|DataUrlLoader.StreamFactory|
|String|InputStream|StringLoader.StreamFactory|
|String|ParcelFileDescriptor|StringLoader.FileDescriptorFactory|
|String|AssetFileDescriptor|StringLoader.AssetFileDescriptorFactory|
|Uri|InputStream|DataUrlLoader.StreamFactory|
|Uri|InputStream|HttpUriLoader.Factory()|
|Uri|InputStream|AssetUriLoader.StreamFactory|
|Uri|InputStream|MediaStoreImageThumbLoader.Factory|
|Uri|InputStream|MediaStoreVideoThumbLoader.Factory|
|Uri|InputStream|UriLoader.StreamFactory|
|Uri|InputStream|UrlUriLoader.StreamFactory|
|Uri|ParcelFileDescriptor|AssetUriLoader.FileDescriptorFactory|
|Uri|ParcelFileDescriptor|UriLoader.FileDescriptorFactory|
|Uri|AssetFileDescriptor|UriLoader.AssetFileDescriptorFactory|
|Uri|File|MediaStoreFileLoader.Factory|
|Uri|Uri|UnitModelLoader.Factory|
|URL|InputStream|UrlLoader.StreamFactory|
|GlideUrl|InputStream|HttpGlideUrlLoader.Factory|
|byte[]|InputStream|ByteArrayLoader.StreamFactory|
|byte[]|ByteBuffer|ByteArrayLoader.ByteBufferFactory|
|Drawable|Drawable|UnitModelLoader.Factory|

我们现在就用 `String` 类型的网络图片地址的 `Model` 来看一下：

寻找处理 `String` 的 `ModelLoader` :

```java
DataUrlLoader.StreamFactory.build=DataUrlLoader
  
StringLoader.StreamFactory.build
  =MultiModelLoaderFactory(Uri.class, InputStream.class)=MultiModelLoader
  ==>DataUrlLoader.StreamFactory=DataUrlLoader
  ==>HttpUriLoader.Factory=HttpUriLoader
  ==>==>MultiModelLoaderFactory(Uri.class, ParcelFileDescriptor.class)
  ==>==>==>AssetUriLoader.FileDescriptorFactory=AssetUriLoader
  ==>==>==>UriLoader.FileDescriptorFactory=AssetUriLoader
  ==>==>==>AssetUriLoader(去重)
  ==>AssetUriLoader.StreamFactory=AssetUriLoader
  ==>MediaStoreImageThumbLoader.Factory=MediaStoreImageThumbLoader
  ==>MediaStoreVideoThumbLoader.Factory=MediaStoreVideoThumbLoader
  ==>UriLoader.StreamFactory=AssetUriLoader
  ==>UrlUriLoader.StreamFactory
  ==>==>MultiModelLoaderFactory(GlideUrl.class, InputStream.class)
  ==>==>==>HttpGlideUrlLoader.Factory=HttpGlideUrlLoader
  
StringLoader.FileDescriptorFactory.build
  =MultiModelLoaderFactory(Uri.class, ParcelFileDescriptor.class)=MultiModelLoader
  ==>AssetUriLoader.FileDescriptorFactory=AssetUriLoader
  ==>UriLoader.FileDescriptorFactory=UriLoader
  
StringLoader.AssetFileDescriptorFactory.build
  =MultiModelLoaderFactory(Uri.class, AssetFileDescriptor.class)
  =UriLoader.AssetFileDescriptorFactory=UriLoader
```

<center>![在这里插入图片描述](https://img-blog.csdnimg.cn/20191220185656297.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)</center>

接下来对这一组 `ModelLoader` 进行处理，过滤掉无法处理 `Model` 数据的 `ModelLoader` 。其中 `MultiModelLoader` 中只要有一个 `ModelLoader` 能处理 `Model` 数据， 那么这个 `MultiModelLoader` 就可以不被过滤。

```java
StringLoader.StreamFactory=MultiModelLoader[7]
StringLoader.FileDescriptorFactory=MultiModelLoader[2]
StringLoader.AssetFileDescriptorFactory=UriLoader
```

<center>![在这里插入图片描述](https://img-blog.csdnimg.cn/20191220185718404.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)</center>

### Transformation

[Glid-Transformation文档：https://muyangmin.github.io/glide-docs-cn/doc/transformations.html](https://muyangmin.github.io/glide-docs-cn/doc/transformations.html)

**Transformation**：用于在实现资源上执行任意转换的类。

```java
//唯一标识某些数据放置的接口。
public interface Key {
    String STRING_CHARSET_NAME = "UTF-8";
    Charset CHARSET = Charset.forName(STRING_CHARSET_NAME);
    //将所有唯一标识信息添加到给定的摘要中。
    void updateDiskCacheKey(@NonNull MessageDigest messageDigest);
    @Override
    boolean equals(Object o);
    @Override
    int hashCode();
}
//用于在实现资源上执行任意转换的类
//equals和hashCode来标识内存缓存中的转换
//updateDiskCacheKey来标识磁盘中的转换缓存。
public interface Transformation<T> extends Key {
    //转换给定资源并返回转换后的资源
    @NonNull
    Resource<T> transform(@NonNull Context context, @NonNull Resource<T> resource, int outWidth, int outHeight);
}
```

### DataRewinder

**DataRewinder**：对数据的游标进行重置，也就是所谓的倒带。

这个逻辑在上一篇文章 [Android-Universal-Image-Loader源码分析](https://dandanlove.blog.csdn.net/article/details/103256724) 中也有讲到过，我们拿到数据流之后可能会从它的头部信息中获取一些图片本身的参数，然后我们再将数据流写入文件缓存的时候要重置数据流的游标保证写入的数据完整。

```java
public interface DataRewinder<T> {
    interface Factory<T> {
      @NonNull
      DataRewinder<T> build(@NonNull T data);
      @NonNull
      Class<T> getDataClass();
    }
    //将包装的数据回退到实例化此对象时的位置，然后返回重新包装的数据
    @NonNull
    T rewindAndGet() throws IOException;
    //当不再需要该复卷机并可以对其进行清理时调用。
    void cleanup();
}
```

`Glide` 中的 `DataRewinder` 模块设计也使用的是**工厂模式**。

| DataClass   | DataRewinder        | Factory                     | 添加时机         |
| ----------- | ------------------- | --------------------------- | ---------------- |
| InputStream | InputStreamRewinder | InputStreamRewinder.Factory | Glide构造中注册  |
| ByteBuffer  | ByteBufferRewinder  | ByteBufferRewinder.Factory  | Glide构造中注册  |
| ALL         | DefaultRewinder     | DEFAULT_FACTORY             | 默认值，不用添加 |

### Transition

`Transition` 不是由`Glide`进行注册的，而是我们在业务代码中按照需要添加的。它作为`Glide`组件的一种，所以我们放在这里来进行介绍。

[Glide-Transition文档：https://muyangmin.github.io/glide-docs-cn/doc/transitions.html](https://muyangmin.github.io/glide-docs-cn/doc/transitions.html)

`Glide` 提供了很多的过渡效果，用户可以手动地应用于每个请求。`Glide` 的内置过渡以一致的方式运行，并且将根据加载图像的位置在某些情况下避免运行。

<center>![在这里插入图片描述](https://img-blog.csdnimg.cn/201912201901022.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)</center>

```java
//包装视图的目标将能够提供所有必要的参数并开始过渡。
public interface Transition<R> {
    //包含视图的接口，该视图公开了运行各种类型的必需的方法
    //Glide中所有ViewTarget的子类都实现了该接口
    interface ViewAdapter {
        //返回包装的view
        View getView();
        //返回在视图中显示的当前可绘制对象；如果不存在这样的可绘制对象，则返回null（或无法检索）。
        @Nullable
        Drawable getCurrentDrawable();
        //设置当前可绘制对象（通常是动画可绘制对象）以在包装视图中显示。
        void setDrawable(Drawable drawable);
    }
    //从当前正在使用的上一个Drawable进行动画处理在给定视图中显示，
    //如果在运行过渡过程中将新资源放在视图中，则为True，
    //如果调用者需要手动将当前资源放在视图上，则为false。
    boolean transition(R current, ViewAdapter adapter);
}
//一个工厂类，可以根据请求的状态产生不同的Transition
public interface TransitionFactory<R> {
    //返回一个新的Transition
    //isFirstResource如果这是要加载到目标中的第一个资源，则为True。
    Transition<R> build(DataSource dataSource, boolean isFirstResource);
}
```

## Glide使用

我们上面从业务代码中自定义的`@GlideModule`注解讲到了`Glide`的构造，以及`Glide`中模块&组件的注册以及其对应的功能和处理逻辑。

接下来我们继续根据业务代码中`Glide`最常用的代码，来讲述`Glide`的其它部分。

```java
Glide.with(context)
    .load(myUrl)
    .into(imageView);
```

这段代码中有我们三个业务逻辑参数：`context`、`myUrl`、`imageView`。

> - `context` ：本次加载图片的上下文环境；
> - `myUrl` ：本次需要加载图片的地址，也叫数据；
> - `imageView` ：本次需要加载图片的`View` ，也叫目标；

## RequestManager

```java
Glide.with(context)
```

根据这一行代码我们进行分析：

```java
//com.bumptech.glide.Glide.java
@NonNull
public static RequestManager with(@NonNull Context context) {
    return getRetriever(context).get(context);
}
```

`Glide.with(context)`获取的是`RequestManager` 。

```java
//用于管理和启动对Glide的请求的类。可以使用活动，片段和连接性生命周期事件智能地停止，启动和重新启动请求。
//通过实例化一个新对象或利用Activity和Fragment生命周期内置的处理功能进行检索，请对Fragment或Activity使用静态Glide.load方法。
public class RequestManager
    implements ComponentCallbacks2, LifecycleListener, ModelTypes<RequestBuilder<Drawable>> {
    /***代码全都省略***/
}
```

> `ComponentCallbacks2` ：`Android` 自带的内存管理的接口，`Application` 和 `Activity` 都有实现。主要可以根据系统的内存状况及时调整App内存占用，提升用户体验或让App存活更久。

> `LifecycleListener` ：activity或者fragment的声明周期监听接口

```java
public interface LifecycleListener {
    void onStart();//fragmetn或者activity的onstart
    void onStop();//fragmetn或者activity的onstop
    void onDestroy();//fragmetn或者activity的ondestroy
}
```

> `ModelTypes` ：通俗的说就是`Glide`的**API**。

```java
interface ModelTypes<T> {
    @NonNull
    @CheckResult
    T load(@Nullable Bitmap bitmap);
    T load(@Nullable Drawable drawable);
    T load(@Nullable String string);
    T load(@Nullable Uri uri);
    T load(@Nullable File file);
    T load(@RawRes @DrawableRes @Nullable Integer resourceId);
    T load(@Nullable URL url);
    T load(@Nullable byte[] model);
    T load(@Nullable Object model);
}
```

### RequestManager的获取

```java
//com.bumptech.glide.manager.RequestManagerRetriever.java
//Application的生命周期单独管理
private volatile RequestManager applicationManager;
//一个是support包，一个是Android的api，RequestManagerFragment的临时存储
//RequestManagerFragment用来管理RequestManager和同步生命周期
final Map<android.app.FragmentManager, RequestManagerFragment> pendingRequestManagerFragments = new HashMap<>();
final Map<FragmentManager, SupportRequestManagerFragment> pendingSupportRequestManagerFragments = new HashMap<>();
public RequestManager get(@NonNull Context context) {
    if (context == null) {
        throw new IllegalArgumentException("You cannot start a load on a null Context");
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {
        if (context instanceof FragmentActivity) {//FragmentActivity
            return get((FragmentActivity) context);
        } else if (context instanceof Activity) {//Activity
            return get((Activity) context);
        } else if (context instanceof ContextWrapper//找到context的basecontext
            && ((ContextWrapper) context).getBaseContext().getApplicationContext() != null) {
            return get(((ContextWrapper) context).getBaseContext());
        }
    }
    return getApplicationManager(context);
}
```

我们用`Fragment` 来举例看看`RequestManager`的获取（`Activity`和`FragmentActivity`的实现都类似）：

```java
public RequestManager get(@NonNull Fragment fragment) {
    Preconditions.checkNotNull(fragment.getContext(),"fragment必须被attch，不能被destroy");
    if (Util.isOnBackgroundThread()) {//如果应用在后台，那么切换为Application
        return get(fragment.getContext().getApplicationContext());
    } else {
        FragmentManager fm = fragment.getChildFragmentManager();
        return supportFragmentGet(fragment.getContext(), fm, fragment, fragment.isVisible());
    }
}
private RequestManager supportFragmentGet(
      @NonNull Context context,
      @NonNull FragmentManager fm,
      @Nullable Fragment parentHint,
      boolean isParentVisible) {
    //获取当前上下文中的SupportRequestManagerFragment
    SupportRequestManagerFragment current =
        getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
    //获取SupportRequestManagerFragment的RequestManager
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
        //给新创建的SupportRequestManagerFragment添加RequestManager
        Glide glide = Glide.get(context);
        requestManager =
            factory.build(
                glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
        current.setRequestManager(requestManager);
    }
    return requestManager;
}
private SupportRequestManagerFragment getSupportRequestManagerFragment(
      @NonNull final FragmentManager fm, @Nullable Fragment parentHint, boolean isParentVisible) {
    //先看看当前的FragmentManager中有没有SupportRequestManagerFragment，有就复用
    SupportRequestManagerFragment current = (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
        //pendingSupportRequestManagerFragments，只是一个临时性的存储,因为每次put之后必定会remove
        //因为add之后立即进行findFragmentByTag是获取不到的，所以需要临时存储
        current = pendingSupportRequestManagerFragments.get(fm);
        if (current == null) {
            //创建新的SupportRequestManagerFragment
            current = new SupportRequestManagerFragment();
            current.setParentFragmentHint(parentHint);  
            //如果fragment已经展示，那么添加的SupportRequestManagerFragment也要同步生命周期
            if (isParentVisible) {
                current.getGlideLifecycle().onStart();
            }
            //添加临时存储
            pendingSupportRequestManagerFragments.put(fm, current);
            //创建的SupportRequestManagerFragment添加到FragmentManager中
            fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
            //删除临时存储
            handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
        }
    }
    return current;
}
```

### 其它功能

执行\取消一个请求、暂停所有请求、重启所有请求等对请求的标记，这部分会在`Request` 详细讲解。在说`Request` 之前先说一下`Request`需要为谁加载目标资源。

## Target

> `Target` ：在声明周期内Glide加载资源回调接口；
>
> `BaseTarget` ：用于加载`Resource`的基础 `Target` 大多数方法的基本或空实现；
>
> `TargetView` ：为Bitmap添加到View上提供了默认实现，

```java
public interface Target<R> extends LifecycleListener {
    //表示我们希望资源保持其原始的未修改宽度和/或高度
    int SIZE_ORIGINAL = Integer.MIN_VALUE;
    //开始加载
    void onLoadStarted(@Nullable Drawable placeholder);
    //加载失败
    void onLoadFailed(@Nullable Drawable errorDrawable);
    //资源加载完成时将调用的方法。
    void onResourceReady(@NonNull R resource, @Nullable Transition<? super R> transition);
    //在取消负载及其资源释放时调用
    void onLoadCleared(@Nullable Drawable placeholder);
    //一种检索此目标大小的方法
    void getSize(@NonNull SizeReadyCallback cb);
    //如果给定的回调仍保留，则将其从待处理集中删除
    void removeCallback(@NonNull SizeReadyCallback cb);
    //设置为此目标保留的当前请求
    void setRequest(@Nullable Request request);
    //检索对此目标的当前请求
    @Nullable
    Request getRequest();
}
```

## RequestBuilder

```java
Glide.with(context)
    .load(myUrl)
```

我们继续看下一行`load`代码：

```java
//com.bumptech.glide.RequestManager.java
public RequestBuilder<Drawable> load(@Nullable String string) {
    return asDrawable().load(string);
}
```

`RequestBuilder`通用类，可以处理通用资源类型的设置选项和启动负载。`GlideRequest`虽然继承了`RequestBuilder` 但内部的实现都是`super`调用父类的方法。

```java
//com.bumptech.glide.RequestManager.java
public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
}
public <ResourceType> RequestBuilder<ResourceType> as(
      @NonNull Class<ResourceType> resourceClass) {
    return new RequestBuilder<>(glide, this, resourceClass, context);
}
```

`asDrawable` 创建一个请求资源类型为 `Drawable.class` 的 `RequestBuilder` 。

```java
//com.bumptech.glide.RequestBuilder.java
public RequestBuilder<TranscodeType> load(@Nullable String string) {
    return loadGeneric(string);
}
private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    this.model = model;
    isModelSet = true;
    return this;
}
```

接下来设置这个 `RequestBuilder` 的 `Model` 为`String.class` 的 `myUrl` 。

```java
Glide.with(context)
    .load(myUrl)
    .into(imageView);
```

`RequestBuilder` 的 `into`  方法是 `Glide` 加载图片的最后一步。

```java
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    Util.assertMainThread();
    Preconditions.checkNotNull(view);
    BaseRequestOptions<?> requestOptions = this;
    //下面是根据view修改requestOptions的属性
    if (!requestOptions.isTransformationSet()
        && requestOptions.isTransformationAllowed()
        && view.getScaleType() != null) {
      switch (view.getScaleType()) {
        case CENTER_CROP:
          requestOptions = requestOptions.clone().optionalCenterCrop();
          break;
        case CENTER_INSIDE:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case FIT_CENTER:
        case FIT_START:
        case FIT_END:
          requestOptions = requestOptions.clone().optionalFitCenter();
          break;
        case FIT_XY:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case CENTER:
        case MATRIX:
        default:
          // Do nothing.
      }
    }
    return into(
        glideContext.buildImageViewTarget(view, transcodeClass),
        /*targetListener=*/ null,
        requestOptions,
        Executors.mainThreadExecutor());
}
```

`into` 方法主要根据 `View` 的属性重构造 `requestOptions` ， 并生成 `DrawableImageViewTarget` 。

### ViewTarget的生成

```java
public class GlideContext extends ContextWrapper {
    public <X> ViewTarget<ImageView, X> buildImageViewTarget(
        @NonNull ImageView imageView, @NonNull Class<X> transcodeClass) {
        return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
    }
    /***部分代码省略***/
}
public class ImageViewTargetFactory {
    @NonNull
    @SuppressWarnings("unchecked")
    public <Z> ViewTarget<ImageView, Z> buildTarget(
        @NonNull ImageView view, @NonNull Class<Z> clazz) {
        if (Bitmap.class.equals(clazz)) {
            return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);
        } else if (Drawable.class.isAssignableFrom(clazz)) {
            return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);
        } else {
            throw new IllegalArgumentException(
            "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
      }
    }
}
```

`transcodeClass` 实际就是上面的 `resourceClass` 也就是 `Drawable.class` 。

接下来我们继续看它的 `into` 方法：

```java
private <Y extends Target<TranscodeType>> Y into(
      @NonNull Y target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> options,
      Executor callbackExecutor) {
    Preconditions.checkNotNull(target);
    //检测是否设置了model
    if (!isModelSet) {
      throw new IllegalArgumentException("You must call #load() before calling #into()");
    }
    //创建请求,Executors.mainThreadExecutor()标识callback在主线程
    Request request = buildRequest(target, targetListener, options, callbackExecutor);
    Request previous = target.getRequest();
    //如果target上的当前请求与上一次的相同，且请求缓存磁盘并且上一个请求为完成时使用上一个请求
    if (request.isEquivalentTo(previous)
        && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
      //如果请求已完成，请重新开始，以确保重新发送结果，触发RequestListeners和Targets。
      //如果请求失败，将重新开始重新启动请求，使它有另一个完成的机会。
      //如果请求已经正在运行，我们可以让它继续运行而不会受到干扰。
      if (!Preconditions.checkNotNull(previous).isRunning()) {
        //使用上一个请求而不是新请求来进行优化，例如跳过设置占位符，跟踪和取消跟踪目标并获取视图尺寸在单独的请求中完成。 
        previous.begin();
      }
      return target;
    }
    //清除target之前的请求
    requestManager.clear(target);
    //给target设置新的请求
    target.setRequest(request);
    //target同步requestmanager的声明周期，并将request添加到请求列表
    requestManager.track(target, request);
    return target;
}
private boolean isSkipMemoryCacheWithCompletePreviousRequest(
    BaseRequestOptions<?> options, Request previous) {
    return !options.isMemoryCacheable() && previous.isComplete();
}
```

好吧，终于看见加载图片的请求了~！

这里会根据之前构建的一些参数来生成 `Request` ，然后和 `ViewTarget` 的 `Request` 进行比较决定是否重用之前的 `Request` ，或者清除之前的 `Request` 。

## Request

> `Request` ：为 `Target` 加载资源的请求。

```java
public interface Request {
    void begin();//开始异步加载
    //防止从以前的请求中加载任何位图，释放由此持有的任何资源请求，并将该请求标记为具有已取消。
    void clear();
    //暂停请求
    void pause();
    //如果此请求正在运行并且尚未完成或失败，则返回true。
    boolean isRunning();
    //如果请求成功完成，则返回true
    boolean isComplete();
    //如果请求已清除，则返回true
    boolean isCleared();
    //判断两个请求是否相同
    boolean isEquivalentTo(Request other);
}
```

### Request的创建

```java
Request request = buildRequest(target, targetListener, options, callbackExecutor);
```

> - `target` : 我们对 `ImageView` 包装之后的的 `ViewTarget`；
> - `targetListener` ：默认为`null`；
> - `options` ：为重新根据`view` 属性修改之后的 `requestOptions` ；
> - `callbackExecutor` : 回调的线程池，在主线程中执行回调；

生成的`Request` 实例为 `SingleRequest`，它是专门为了`Target`而加载资源的。

如果需要设置处理 `Error` 的请求那么会在 `Request` 外包一层生成 `ErrorRequestCoordinator`。

> `ErrorRequestCoordinator` ：运行单个主要的Request直到完成，然后仅在单个主要请求失败的情况下后退错误请求；

如果需要设置处理缩略图的请求那么会在 `Request` 外包一层生成 `ThumbnailRequestCoordinator`。

> `ThumbnailRequestCoordinator` ：一个协调器，用于协调两个单独的`Request`，它们同时加载图像的小缩略图版本和图像的完整尺寸版本。

整个`Glide` 中 `Request` 的实现类就`SingleRequest` 、`ErrorRequestCoordinator` 和 `ThumbnailRequestCoordinator` 一共三个。

### SingleRequest

`SingleRequest` 不仅实现了 `Request` ，还实现了`SizeReadyCallback` 和 `ResourceCallback` 接口。

> `SizeReadyCallback` ：当目标确定其大小时必须调用的回调。对于固定尺寸的目标可以同步调用。

```java
public interface SizeReadyCallback {
    void onSizeReady(int width, int height);
}
```

> `ResourceCallback` ：侦听资源加载成功完成或失败的回调。

```java
public interface ResourceCallback {
  //成功加载资源时调用。
  void onResourceReady(Resource<?> resource, DataSource dataSource);
  //当资源无法成功加载时调用。
  void onLoadFailed(GlideException e);
  //返回通知单个请求时要使用的锁
  Object getLock();
}
```

上面我们阅读`into`代码的时候知道了 `request` 的执行是调用了`RequestManager.track` 。

```java
//com.bumptech.glide.RequestManager
//对request进行追踪，默认的new RequestTracker()；
private final RequestTracker requestTracker；
//对target进行追踪
private final TargetTracker targetTracker = new TargetTracker();
synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
    //添加到RequestManger的target追踪容器中
    targetTracker.track(target);
    //运行request
    requestTracker.runRequest(request);
}
```

> - `TargetTracker` : 保留当前对有效的一组Target，并转发生命周期事件。
> - `RequestTracker` ：用于跟踪，取消和重新启动进行中，已完成和失败的请求的类。

`Request` 已经讲述了一部分了，之前 `RequestManger` 部分对于 `Request` 的请求、清除、暂停、重新启动没有详细讲述，现在可以开始。

### Request.begin

```java
//com.bumptech.glide.manager.RequestTracker
public void runRequest(@NonNull Request request) {
    //添加到请求容器中
    requests.add(request);
    //如果当前的RequestManager不处于暂停状态，那么直接开始请求
    if (!isPaused) {
        request.begin();
    } else {
        //否则延迟该请求
        request.clear();
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            Log.v(TAG, "Paused, delaying request");
        }
         pendingRequests.add(request);
    }
}
```

接下来看`SingleRequest ` 的 `begin` 方法，开始真正的获取资源。

```java
//com.bumptech.glide.manager.RequestTracker
@Override
public void begin() {
    synchronized (requestLock) {
        assertNotCallingCallbacks();//如果已经进行了回调，那么抛异常
        stateVerifier.throwIfRecycled();//如果任务已经被回收，那么抛异常
        startTime = LogTime.getLogTime();
        if (model == null) {
            //校验宽高是否有效
            if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
                width = overrideWidth;
                height = overrideHeight;
            }
            //详细的日志级别进行记录
            int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
            onLoadFailed(new GlideException("Received null model"), logLevel);
            return;
        }
        if (status == Status.RUNNING) {//如果任务已经处于运行状态，那么抛异常
          throw new IllegalArgumentException("Cannot restart a running request");
        }
        //如果任务已经执行完成，那么直接回调加载资源
        if (status == Status.COMPLETE) {
            onResourceReady(resource, DataSource.MEMORY_CACHE);
            return;
        }
        //重新启动既未完成也未运行的请求，可以视为新请求并可以从头开始再次运行。
        status = Status.WAITING_FOR_SIZE;
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
           onSizeReady(overrideWidth, overrideHeight);//内部有请求资源的实现方法调用
        } else {
           target.getSize(this);//重新回调onSizeReady方法
        }
        //加载占位图
        if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
            && canNotifyStatusChanged()) {
            target.onLoadStarted(getPlaceholderDrawable());
        }
    }
}
```

已知加载的`ViewTarget`的宽高才会进一步加载资源。

```java
//com.bumptech.glide.request.SingleRequest
@Override
public void onSizeReady(int width, int height) {
    stateVerifier.throwIfRecycled();//如果任务已经被回收，那么抛异常
    synchronized (requestLock) {//加锁，进行数据请求
        if (status != Status.WAITING_FOR_SIZE) {//必须size确定
            return;
        }
        status = Status.RUNNING;//进入运行状态
        //在加载资源之前，对Target的大小应用乘数。对于加载缩略图或避免加载大量资源非常有用。默认为1f。
        float sizeMultiplier = requestOptions.getSizeMultiplier();
        this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
        this.height = maybeApplySizeMultiplier(height, sizeMultiplier);
        loadStatus =
                engine.load(
                        glideContext,
                        model,//myUrl
                        requestOptions.getSignature(),//缓存签名，默认为EmptySignature
                        this.width,
                        this.height,
                        requestOptions.getResourceClass(),//不设置decode的话，默认为Object
                        transcodeClass,//Drawable.class
                        priority,
                        requestOptions.getDiskCacheStrategy(),
                        requestOptions.getTransformations(),
                        requestOptions.isTransformationRequired(),
                        requestOptions.isScaleOnlyOrNoTransform(),
                        requestOptions.getOptions(),
                        requestOptions.isMemoryCacheable(),
                        requestOptions.getUseUnlimitedSourceGeneratorsPool(),
                        requestOptions.getUseAnimationPool(),
                        requestOptions.getOnlyRetrieveFromCache(),
                        this,
                        callbackExecutor);
        /***部分代码省略***/
    }
}
```

## Engine.load

```java
//com.bumptech.glide.load.engine.Engine
public <R> LoadStatus load(/***部分代码省略***/) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;
    EngineKey key = //构造缓存的key
            keyFactory.buildKey(
                    model,//数据myUrl
                    signature,//签名
                    width,
                    height,
                    transformations,//转变
                    resourceClass,//Object.class
                    transcodeClass,//Drawable.class
                    options);
    EngineResource<?> memoryResource;
    synchronized (this) {
        //先从缓存中加载
        memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);
        if (memoryResource == null) {
            return waitForExistingOrStartNewJob(/***部分代码省略***/);
        }
    }
    //回调可以并发，不需要进行锁操作
    cb.onResourceReady(memoryResource, DataSource.MEMORY_CACHE);
    return null;
}
```

### 内存缓存加载

```java
private EngineResource<?> loadFromMemory(EngineKey key, boolean isMemoryCacheable, long startTime) {
    if (!isMemoryCacheable) {//如果不允许使用内存，那么直接返回null
        return null;
    }
    EngineResource<?> active = loadFromActiveResources(key);//获取活跃的资源
    if (active != null) {
        return active;
    }
    EngineResource<?> cached = loadFromCache(key);//获取内存缓存中的资源
    if (cached != null) {
        return cached;
    }
    return null;
}
```

#### ActiveResources

> `ActiveResources` ：利用持有`Resource` 弱引用的 `HashMap` 来进行数据保存。

```java
private EngineResource<?> loadFromActiveResources(Key key) {
    EngineResource<?> active = activeResources.get(key);
    if (active != null) {
        active.acquire();
    }
    return active;
}
```

- 添加操作：有两个触发点，一个是从 `cache` 中读取的时候会将资源添加到 `ActiveResources` 。另一个获取资源完成的时候会添加到 `ActiveResources` 。它们有一个相同的点都在使用资源的时候添加到 `ActiveRsources` 。
- 删除操作：`EngineResource` 在内部有一个引用计数器，每次被获取的时候都会进行 `acquire` 自加。被释放的时候也是会自减。当引用计数为0的时候，会被 `ActivityResources` 释放，并添加到 `MemoryCache` 中。

#### MemoryCache

> `MemoryCache` ：同样使用 **LRU 算法**，实现类为` LruResourceCache` ，它提供了一个 `ResourceRemovedListener `接口，当有资源从 `MemoryCache` 中被移除时会回调其中的方法，`Engine` 中接收到这个消息后就会进行 `Bitmap` 的回收操作。

```java
private EngineResource<?> loadFromCache(Key key) {
    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
        cached.acquire();
        activeResources.activate(key, cached);
    }
    return cached;
}
private EngineResource<?> getEngineResourceFromCache(Key key) {
    Resource<?> cached = cache.remove(key);
    final EngineResource<?> result;
    if (cached == null) {
        result = null;
    } else if (cached instanceof EngineResource) {
        result = (EngineResource<?>) cached;
    } else {
        result = new EngineResource<>(cached, true, true, key, this);
    }
    return result;
}
@Override
public void onResourceReleased(Key cacheKey, EngineResource<?> resource) {
    activeResources.deactivate(cacheKey);
    if (resource.isMemoryCacheable()) {
        cache.put(cacheKey, resource);
    } else {
        resourceRecycler.recycle(resource);
    }
}
```

- 添加操作：当资源被 `ActiveResources` 释放的时候；
- 删除操作：当资源被 `ActivityResource` 获取的时候；

以上两点保证了 `ActiveResource` 和 `MemoryCache` 中的资源是不会有重复的。

### DecodeJob

> `DecodeJob` ：负责从缓存的数据或原始源解码资源，并应用转换和转码。
接下来的部分会详细讲解上图的每个部分。

<center>![在这里插入图片描述](https://img-blog.csdnimg.cn/20191220190409636.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)</center>

### 磁盘缓存加载

等待创建或者是获取已经存在的加载状态。

```java
private <R> LoadStatus waitForExistingOrStartNewJob(/***部分代码省略***/) {
    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);//从hashmap中获取加载key的工作引擎
    if (current != null) {//如果已经存在那么直接返回
        current.addCallback(cb, callbackExecutor);
        return new LoadStatus(cb, current);
    }
    EngineJob<R> engineJob = //通过添加和删除负载的回调并在负载完成时通知回调来管理负载的类。
            engineJobFactory.build(/***部分代码省略***/);
    DecodeJob<R> decodeJob = //负责从缓存的数据或原始源解码资源，并应用转换和转码。
            decodeJobFactory.build(/***部分代码省略***/);
    jobs.put(key, engineJob);//添加到hashmap中
    engineJob.addCallback(cb, callbackExecutor);//添加回调信息
    engineJob.start(decodeJob);//开始工作
    return new LoadStatus(cb, engineJob);
}
```

选择执行线程，开始执行任务：

```java
//com.bumptech.glide.load.engine.EngineJob
public synchronized void start(DecodeJob<R> decodeJob) {
    this.decodeJob = decodeJob;
    //根据磁盘缓存策略，判断使用哪个线程池执行任务
    GlideExecutor executor = decodeJob.willDecodeFromCache() ? diskCacheExecutor : getActiveSourceExecutor();
    executor.execute(decodeJob);
}
```

在`diskCacheExecutor` 线程池中执行 `DecoceJob`。

```java
//com.bumptech.glide.load.engine.DecodeJob
public void run() {
    /***部分代码省略***/
    DataFetcher<?> localFetcher = currentFetcher;
    try {
        if (isCancelled) {//如果任务已经取消，那么进行失败回调
            notifyFailed();
            return;
        }
        runWrapped();//开始执行任务
    } catch (CallbackException e) {
        //如果不受Glide控制的回调引发异常，则应避免使用Glide下面的特定调试逻辑。 
    } catch (Throwable t) {
        //通知失败，进行失败回调，抛出异常
    } finally {
        if (localFetcher != null) {
            localFetcher.cleanup();
        }
        GlideTrace.endSection();
    }
}
```

`runWrapped` 为 `run` 实现的包装。

```java
//com.bumptech.glide.load.engine.DecodeJob
private enum RunReason {
    //初始化
    INITIALIZE,
    //切换执行服务
    SWITCH_TO_SOURCE_SERVICE,
    //获取数据之后的资源编解码
    DECODE_DATA,
}
private void runWrapped() {
    switch (runReason) {
        case INITIALIZE:
            stage = getNextStage(Stage.INITIALIZE);
            currentGenerator = getNextGenerator();
            runGenerators();
            break;
        case SWITCH_TO_SOURCE_SERVICE:
            runGenerators();
            break;
        case DECODE_DATA:
            decodeFromRetrievedData();
            break;
        default:
            throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
}
```

再讲下一步数据从磁盘的获取之前，先看一下我们默认的磁盘策略：

```java
public static final DiskCacheStrategy AUTOMATIC =
    new DiskCacheStrategy() {
        @Override
        public boolean isDataCacheable(DataSource dataSource) {
            return dataSource == DataSource.REMOTE;
        }
        @Override
        public boolean isResourceCacheable(
            boolean isFromAlternateCacheKey, DataSource dataSource, EncodeStrategy encodeStrategy) {
            return ((isFromAlternateCacheKey && dataSource == DataSource.DATA_DISK_CACHE)
                    || dataSource == DataSource.LOCAL)
                && encodeStrategy == EncodeStrategy.TRANSFORMED;
        }
        @Override
        public boolean decodeCachedResource() {
            return true;
        }
        @Override
        public boolean decodeCachedData() {
            return true;
        }
    };
```

- 如果数据时 `REMOTE` 类型那么可以进行缓存；
- 如果是解码资源而且`Key` 不是源数据的`Key` ，资源为 `DATA_DISK_CACHE` 的话，那么这个资源可以缓存；
- 可以缓存解码资源；
- 可以缓存解码数据；

从上面我们可以了解到缓存有`数据缓存` 和 `解码后的数据缓存` 。

我们接着分析 `runWrapped` 方法内调用的 `getNextStage` :

```java
//com.bumptech.glide.load.engine.DecodeJob
private Stage getNextStage(Stage current) {
    switch (current) {
        case INITIALIZE://AUTOMATIC默认为true，所以初始化的第一步应该检测Stage.RESOURCE_CACHE
            return diskCacheStrategy.decodeCachedResource()
                ? Stage.RESOURCE_CACHE
                : getNextStage(Stage.RESOURCE_CACHE);
        case RESOURCE_CACHE://AUTOMATIC默认为true，所以初始化的下一步应该检测Stage.DATA_CACHE
            return diskCacheStrategy.decodeCachedData()
                ? Stage.DATA_CACHE
                : getNextStage(Stage.DATA_CACHE);
        case DATA_CACHE:////如果用户选择仅从缓存中检索资源，则从源跳过加载。否则应该检测Stage.SOURCE
            return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
        case SOURCE:
        case FINISHED:
            return Stage.FINISHED;
        default:
            throw new IllegalArgumentException("Unrecognized stage: " + current);
    }
}
```

知道了`Stage` 类型接着分析 `runWrapped` 方法内调用的 `getNextStage` :

```java
//com.bumptech.glide.load.engine.DecodeJob
private DataFetcherGenerator getNextGenerator() {
    switch (stage) {
        case RESOURCE_CACHE://解码之后的资源缓存
            return new ResourceCacheGenerator(decodeHelper, this);
        case DATA_CACHE://源数据的缓存
            return new DataCacheGenerator(decodeHelper, this);
        case SOURCE://数据源
            return new SourceGenerator(decodeHelper, this);
        case FINISHED:
            return null;
        default:
            throw new IllegalStateException("Unrecognized stage: " + stage);
    }
}
```

> `DataFetcherGenerator` : 注册 `ModelLoaders` 和 `Model` 的数据提取构造器。

```java
interface DataFetcherGenerator {
    //尝试使用一个新的DataFetcher，则返回true已启动，否则为false。
    boolean startNext();
    //尝试取消当前正在运行的提取程序
    void cancel();
}
```

`ResourceCacheGenerator` 、`DataCacheGenerator` 和 `SourceGenerator` 除过实现了`DataFetcherGenerator` 还实现了 `DataCallback` 接口。

> `DataCallback` : 数据已加载且可用时或加载时必须调用的回调失败。

```java
interface DataCallback<T> {
    //如果加载成功，则使用加载的数据进行调用；如果加载失败，则使用null进行调用。
    void onDataReady(@Nullable T data);
    //加载失败时调用
    void onLoadFailed(@NonNull Exception e);
}
```

获取到`DataFetcherGenerator` 构造器之后执行 `startNext` ，判断是否下一步能开始加载数据。

```java
//com.bumptech.glide.load.engine.EngineJob
private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();
    boolean isStarted = false;
    while (!isCancelled && currentGenerator != null 
        && !(isStarted = currentGenerator.startNext())) {//执行当前数据构造的判断是否能加载数据
        stage = getNextStage(stage);//切换数据状态
        currentGenerator = getNextGenerator();//获取新的数据构造器
        if (stage == Stage.SOURCE) {//需要需要进行源数据加载那么切换执行线程
            reschedule();
            return;
        }
    }
    //如果已经完成或者取消，而且没开始获取数据。那么通知失败
    if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
        notifyFailed();
    }
    //否则，生成器将开始新的加载，我们希望将其调回onDataFetcherReady。
}
```

#### ResourceCacheGenerator

> `ResourceCacheGenerator` ：从包含降采样/转换后的资源缓存文件中获取数据。

```java
class ResourceCacheGenerator implements DataFetcherGenerator, DataFetcher.DataCallback<Object> {    
    /***部分代码省略***/
    @Override
    public boolean startNext() {
        List<Key> sourceIds = helper.getCacheKeys();//能处理Model（myUrl）的LoadData列表，返回其Key
        if (sourceIds.isEmpty()) {//如果没有，那么标示Model（myUrl）这个数据类型无法处理，直接返回false
            return false;
        }
        //获取glide中注册的，能加载Model（url），能解码resourceClass（Drawable），能转码transcodeClass(Object)的组件集合
        List<Class<?>> resourceClasses = helper.getRegisteredResourceClasses();
        if (resourceClasses.isEmpty()) {//如果没有适配的组件集合，而且转码的类型为文件的话那么抛异常。因为File都不能转码那么根本无法磁盘缓存
            if (File.class.equals(helper.getTranscodeClass())) {
                return false;
            }
            throw new IllegalStateException(/***部分代码省略***/);
        }
        while (modelLoaders == null || !hasNextModelLoader()) {//如果还有modelLoader
            resourceClassIndex++;
            //双重判断使LoadData列表进行完全遍历
            if (resourceClassIndex >= resourceClasses.size()) {
                sourceIdIndex++;
                if (sourceIdIndex >= sourceIds.size()) {
                    return false;
                }
                resourceClassIndex = 0;
            }
            Key sourceId = sourceIds.get(sourceIdIndex);
            Class<?> resourceClass = resourceClasses.get(resourceClassIndex);
            Transformation<?> transformation = helper.getTransformation(resourceClass);
            //ResourceCacheKey用于下采样和转换的资源数据的缓存密钥+任何请求的签名。
            currentKey =
                new ResourceCacheKey(
                    helper.getArrayPool(),
                    sourceId,//modelLoaddata
                    helper.getSignature(),//签名
                    helper.getWidth(),//宽
                    helper.getHeight(),//高
                    transformation,//转换
                    resourceClass,//解码
                    helper.getOptions());
            cacheFile = helper.getDiskCache().get(currentKey);
            //判断Resource是否有缓存，如果有那么加载这个缓存文件数据
            if (cacheFile != null) {
                sourceKey = sourceId;
                modelLoaders = helper.getModelLoaders(cacheFile);//获取能加载文件的ModelLoader列表
                modelLoaderIndex = 0;
            }
        }
        loadData = null;
        boolean started = false;
        while (!started && hasNextModelLoader()) {
            ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
            //ModelLoader根据cacheFile，width，height，options构造LoadData
            loadData = modelLoader.buildLoadData(cacheFile, helper.getWidth(), helper.getHeight(), helper.getOptions());
            //该LoadData是否被Glide进行加载过，是否可用
            if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
                started = true;//如果可用，那么使用该LoadData中的DataFetcher进行数据获取
                loadData.fetcher.loadData(helper.getPriority(), this);
            }
        }

        return started;
    }
    @Override
    public void cancel() {
        LoadData<?> local = loadData;
        if (local != null) {
            local.fetcher.cancel();
        }
    }
    @Override//数据已经准备好，标识数据源为RESOURCE_DISK_CACHE
    public void onDataReady(Object data) {
        cb.onDataFetcherReady(
            sourceKey, data, loadData.fetcher, DataSource.RESOURCE_DISK_CACHE, currentKey);
    }
    @Override//加载失败，标识数据源为RESOURCE_DISK_CACHE
    public void onLoadFailed(@NonNull Exception e) {
        cb.onDataFetcherFailed(currentKey, e, loadData.fetcher, DataSource.RESOURCE_DISK_CACHE);
    }
}
```

#### DataCacheGenerator

> `DataCacheGenerator` ： 构造来自包含原始未修改源数据的缓存文件。

```java
class DataCacheGenerator implements DataFetcherGenerator, DataFetcher.DataCallback<Object> {
    /***部分代码省略***/
    @Override
    public boolean startNext() {
        //如果还有modelLoader
        while (modelLoaders == null || !hasNextModelLoader()) {
            sourceIdIndex++;
            //cacheKyes（helper.getCacheKeys）在构造的时候传入，和ResourceCacheGenerator的相同
            if (sourceIdIndex >= cacheKeys.size()) {
                return false;
            }
            Key sourceId = cacheKeys.get(sourceIdIndex);
            //原始源数据的缓存键和任何请求的签名。
            Key originalKey = new DataCacheKey(sourceId, helper.getSignature());
            cacheFile = helper.getDiskCache().get(originalKey);
            //判断Resource是否有缓存，如果有那么加载这个缓存文件数据
            if (cacheFile != null) {
                this.sourceKey = sourceId;
                modelLoaders = helper.getModelLoaders(cacheFile);//获取能加载文件的ModelLoader列表
                modelLoaderIndex = 0;
            }
        }
        loadData = null;
        boolean started = false;
        while (!started && hasNextModelLoader()) {
            ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
            //ModelLoader根据cacheFile，width，height，options构造LoadData
            loadData = modelLoader.buildLoadData(cacheFile, helper.getWidth(), helper.getHeight(), helper.getOptions());
            //该LoadData是否被Glide进行加载过，是否可用
            if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
                started = true;//如果可用，那么使用该LoadData中的DataFetcher进行数据获取
                loadData.fetcher.loadData(helper.getPriority(), this);
            }
        }
        return started;
    }
    @Override
    public void onDataReady(Object data) {//数据已经准备好，标识数据源为DATA_DISK_CACHE
        cb.onDataFetcherReady(sourceKey, data, loadData.fetcher, DataSource.DATA_DISK_CACHE, sourceKey);
    }
    @Override
    public void onLoadFailed(@NonNull Exception e) {//加载失败，标识数据源为DATA_DISK_CACHE
        cb.onDataFetcherFailed(sourceKey, e, loadData.fetcher, DataSource.DATA_DISK_CACHE);
    }
}
```

可以看出来 `ResourceCacheGenerator` 的实现其实和 `DataCacheGenerator` 的实现逻辑很相似，只不过是缓存的 `Key` 不同而已。

### 源数据请求

> `SourceGenerator` ： 根据磁盘缓存策略，可以先将源数据编码后写入磁盘，然后再加载从缓存文件而不是直接返回。

```java
class SourceGenerator implements DataFetcherGenerator,DataFetcher.DataCallback<Object>,
				DataFetcherGenerator.FetcherReadyCallback {
    private DataCacheGenerator sourceCacheGenerator;//磁盘缓存的数据构造器
    private Object dataToCache;//需要缓存的数据
    private DataCacheKey originalKey;//磁盘缓存的key
    /***部分代码省略***/
    @Override
    public boolean startNext() {
        if (dataToCache != null) {//是否已经拥有缓存的数据
            Object data = dataToCache;
            dataToCache = null;
            cacheData(data);//进行据缓存，缓存后sourceCacheGenerator不为空
        }
        //由SourceGenerator转到DataCacheGenerator（网络转磁盘）
        if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
            return true;
        }
        sourceCacheGenerator = null;
        loadData = null;
        boolean started = false;
        while (!started && hasNextModelLoader()) {//如果还有LoadData(model,width，height，options构造LoadData)
            loadData = helper.getLoadData().get(loadDataListIndex++);
            //fetcher.getDataSource()是否可以进行磁盘缓存
            if (loadData != null && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
                    || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {//该LoadData是否被Glide进行加载过，是否可用
                started = true;//如果可用，那么使用该LoadData中的DataFetcher进行数据获取
                loadData.fetcher.loadData(helper.getPriority(), this);
            }
        }
        return started;
    }
    /***部分代码省略***/
    private void cacheData(Object dataToCache) {
        long startTime = LogTime.getLogTime();
        try {//获取dataToCache对应的解码器，StreamEncoder
            Encoder<Object> encoder = helper.getSourceEncoder(dataToCache);
            DataCacheWriter<Object> writer = new DataCacheWriter<>(encoder, dataToCache, helper.getOptions());
            originalKey = new DataCacheKey(loadData.sourceKey, helper.getSignature());//生成数据源的key
            helper.getDiskCache().put(originalKey, writer);//将writer写入到磁盘缓存中，对应的key为originalKey
        } finally {
            loadData.fetcher.cleanup();
        }
        //构造原始数据缓存构造器，转到磁盘读取数据
        sourceCacheGenerator = new DataCacheGenerator(Collections.singletonList(loadData.sourceKey), helper, this);
    }
    @Override//数据加载成功回调，date类型为InputStream
    public void onDataReady(Object data) {DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
        //如果数据不为空，而且可以缓存那么机进行SourceGenerator转DataCacheGenerator
        if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
            dataToCache = data;
            cb.reschedule();//切换线程，切换完之后currentGenerator还是SourceGenerator
        } else {//否早直接将数据进行回调
            cb.onDataFetcherReady(loadData.sourceKey,data,loadData.fetcher,loadData.fetcher.getDataSource(),originalKey);
        }
    }
    @Override//数据加载失败回调
    public void onLoadFailed(@NonNull Exception e) {
        cb.onDataFetcherFailed(originalKey, e, loadData.fetcher, loadData.fetcher.getDataSource());
    }
    @Override//DataCacheGenerator的数据加载成功回调，data类型为ByteBuffer
    public void onDataFetcherReady(Key sourceKey, Object data, DataFetcher<?> fetcher, DataSource dataSource, Key attemptedKey) {
        // 此数据提取程序将从文件中加载并提供错误的数据源，因此请覆盖使用原始提取程序的数据源 
        cb.onDataFetcherReady(sourceKey, data, fetcher, loadData.fetcher.getDataSource(), sourceKey);
    }
    @Override//DataCacheGenerator的数据加载失败回调
    public void onDataFetcherFailed(Key sourceKey, Exception e, DataFetcher<?> fetcher, DataSource dataSource) {
        cb.onDataFetcherFailed(sourceKey, e, fetcher, loadData.fetcher.getDataSource());
    }
}
```

`SourceGenerator` 、`DataCacheGenerator` 和 `ResourceCacheGenerator` 的 `startNext` 方法都会寻找合适的 `ModelLoader` 构建对应的 `LoadData` 来加载数据。

现在我们的 `Model` 是 `myUrl` 一个网络图片地址，在没有加载过之前`ResourceCacheGenerator` 和 `DataCacheGenerator` 当然是没有缓存的，所以我们这里用`SourceGenerator` 加载一个网络图片的过程在详细讲述一下 `startNext` 中怎么获取`LoadData` 进行数据加载（其他两个都实现都类似）。

> 需要注意的是`Data`已经是从磁盘缓存读出来的 `ByteBuffer` ，但是`DataSource` 还是 `REMOTE` 。

### 数据解码和转码

无论加载的缓存还是其他的数据成功后都会回调 `onDataFetcherReady` ，然后进行数据解码。

```java
//com.bumptech.glide.load.engine.DecodeJob
public void onDataFetcherReady(Key sourceKey, Object data, DataFetcher<?> fetcher, DataSource dataSource, Key attemptedKey) {
    this.currentSourceKey = sourceKey;//url
    this.currentData = data;//ByteBuffer
    this.currentFetcher = fetcher;//ByteBufferFetcher
    this.currentDataSource = dataSource;//REMOTE
    this.currentAttemptingKey = attemptedKey;//url
    //如果当前线程不是主线程，那么切换到主线程执行数据解码（调用decodeFromRetrievedData）
    if (Thread.currentThread() != currentThread) {
        runReason = RunReason.DECODE_DATA;
        callback.reschedule(this);
    } else {
        GlideTrace.beginSection("DecodeJob.decodeFromRetrievedData");
        try {
            decodeFromRetrievedData();//进行数据解码
        } finally {
            GlideTrace.endSection();
        }
    }
}
```

接下来的步骤：

1. `decodeFromRetrievedData` 中调用 `decodeFromData` ；
2. `decodeFromData` 中调用 `decodeFromFetcher` ；
3. `decodeFromFetcher` 获取 `Registry.getLoadPath`  以及 `Registry.getRewinder`；

最终是为了获得 `DecodePaht` 和 `LoadPath` 。

#### 解码&渐变&转码

> `LoadPath` ：一个或多个 `DecodePath` 来进行数据的解码；

```java
//com.bumptech.glide.Registry
@Nullable
public <Data, TResource, Transcode> LoadPath<Data, TResource, Transcode> getLoadPath(
      @NonNull Class<Data> dataClass,//ByteBuffer.class
      @NonNull Class<TResource> resourceClass,//Object.class
      @NonNull Class<Transcode> transcodeClass) {//Drawable.class
    LoadPath<Data, TResource, Transcode> result = loadPathCache.get(dataClass, resourceClass, transcodeClass);
    if (loadPathCache.isEmptyLoadPath(result)) {
        return null;
    } else if (result == null) {
        List<DecodePath<Data, TResource, Transcode>> decodePaths = getDecodePaths(dataClass, resourceClass, transcodeClass);
        //可能无法将给定的类型解码或转码为所需的类型数据类。 
        if (decodePaths.isEmpty()) {
            result = null;
        } else {
            result = new LoadPath<>(dataClass, resourceClass, transcodeClass, decodePaths, throwableListPool);
        }
        loadPathCache.put(dataClass, resourceClass, transcodeClass, result);
    }
    return result;
}
```

> `DecodePath` ：尝试从给定的数据类型解码和转码资源类型；

```java
//com.bumptech.glide.Registry
@NonNull
private <Data, TResource, Transcode> List<DecodePath<Data, TResource, Transcode>> getDecodePaths(
      @NonNull Class<Data> dataClass,
      @NonNull Class<TResource> resourceClass,
      @NonNull Class<Transcode> transcodeClass) {
    List<DecodePath<Data, TResource, Transcode>> decodePaths = new ArrayList<>();
    //GifDrawable,Bitmap,BitmapDrawable
    List<Class<TResource>> registeredResourceClasses = decoderRegistry.getResourceClasses(dataClass, resourceClass);
    for (Class<TResource> registeredResourceClass : registeredResourceClasses) {
        List<Class<Transcode>> registeredTranscodeClasses = transcoderRegistry.getTranscodeClasses(registeredResourceClass, transcodeClass);
        for (Class<Transcode> registeredTranscodeClass : registeredTranscodeClasses) {
            List<ResourceDecoder<Data, TResource>> decoders = decoderRegistry.getDecoders(dataClass, registeredResourceClass);
            ResourceTranscoder<TResource, Transcode> transcoder = transcoderRegistry.get(registeredResourceClass, registeredTranscodeClass);
            //transcoder=DrawableBytesTranscoder
            DecodePath<Data, TResource, Transcode> path =
                new DecodePath<>(
                    dataClass,//ByteBuffer
                    registeredResourceClass,//GifDrawable/Bitmap/BitmapDrawable
                    registeredTranscodeClass,//Drawable/Drawable/Drawable
              			//ByteBufferGifDecoder/ByteBufferBitmapDecoder/BitmapDrawableDecoder
                    decoders,
              			///UnitTranscoder/BitmapDrawableTranscoder/UnitTranscoder
                    transcoder,
                    throwableListPool);
            decodePaths.add(path);
        }
    }
    return decodePaths;
}
```

1. 解码的时候调用 `DecodePath.decode` 方法，该方法会循环遍历内部的`decoders`列表中的 `ResourceDecoder` 如果`ByteBufferBitmapDecoder.decode`解码成功(`Bitmap`)直接返回。
2. 继续回调判断是或否进行`transform` 如果需要那么 `DecodeJob.onResourceDecoded` 进行 `transform` 进行资源的渐变，渐变之后重新构造缓存的 `Key` 将再次进行 `Encoder` 。
3. 然后再进行 `BitmapDrawableTranscoder.transcode` 进行转码得到 `LazyBitmapDrawableResource` 。
4. 回到 `notifyEncodeAndRelease` 方法进行资源成功回调，以及判断判断是否需要 `Encoder` 进行磁盘缓存，回调编码成功释放资源。

<center>![在这里插入图片描述](https://img-blog.csdnimg.cn/20191220192134589.png#pic_center)</center>

至此整个资源的获取过程完成，最后我们获取出了`Drawable` 。

### 加载&释放资源

> `DecodeJob`

- 进行 `notifyComplete` : 回调 `EngineJob.onResourceReady` ;

- 进行 `onEncodeComplete` ：释放当前的 `DecodeJob` ;

> `EngineJob`

- 进行 `notifyCallbacksOfResult` ，回调 `incrementPendingCallbacks` 进行资源的引用+1 ；
- 回调 `Engine.onEngineJobComplete` ；
- 执行 `CallResourceReady` 任务，回调 `SingleRequest.onResourceReady` ，并进行资源的引用+1；
- 执行 `decrementPendingCallbacks` 进行资源的引用-1；

> `Engine`

- 将资源添加到 `activeResource` ;
- 移除当前`jobs` 中对应的 `EngineJob`;

> `SingleRequest`

- 判断&校验，获取资源；
- 进行 `target.onResourceReadt` 将资源加载到 `View` 上。 

# 总结

因为上一篇文章是 [Android-Universal-Image-Loader源码分析](https://dandanlove.blog.csdn.net/article/details/103256724)  ，所以这里主要是结合 `ImageLoader` 来和 `Glide` 进行比较。

> 功能

1. `Glide` 默认的网络请求和 `ImageLoader` 相同都是 `HttpUrlConnection`；
2. `Glide` 和 `ImageLoader` 都可以添加自定义的网络请求比如 `OkHttp` ;
3. `Glide` 和 `ImageLoader` 都支持在图片加载前获取图片的数据（图片的宽、高）。
4. `Glide` 会根据请求时会根据当前 `context` 的声明周期进行 `Request` 的管理（粒度为 `Application` 和 `Activity`）。这个功能是 `ImageLoader` 不具备的，`ImageLoader` 只能停止一个请求，或者停止所有请求。
5. `Glide` 和 `ImageLoader` 都具有加载默认图、加载失败备用图的功能。
6. `Glide` 具备加载缩略图的功能，这个功能是 `ImageLoader` 不具备的。
7. `Glide` 可以设置请求时候的优先级，虽然说这个请求优先级不是十分严格仅仅是指导来做请求的优化处理，但这个功能是 `ImageLoader` 不具备的。
8. `Glide` 加载图片的数据可支持多种类型，`ImageLoader` 只支持 `String` 。
9. `Glide` 和 `ImageLoader` 都可以自定义配置图片加载库使用的 `download` 、`decoder` 、`executor` 、`cache` 等，只不过 `Glide` 对自定义模块配置更加的方便以及粒度更细。

> 缓存

1. `ImageLoader` 缓存的 `key` 只能根据图片地址进行处理生成。 `Glide` 的原始数据的磁盘缓存的 `Key` 是由 `url` 和 `signature` 组成，资源图片的缓存（磁盘缓存和内存缓存）的 `Key` 是由图片的（宽、高、资源类型、资源转换类型、资源解码类型、签名、`model` 以及 `option`）组成而且签名可以由我们自己的 `request` 进行设置。
2.  `Gilde` 可以进行请求时的设置跳过缓存，或者进行 `signature` 设置防止缓存失效问题。这个是 `ImageLoader` 不具备的功能。
3. `Glide` 进行渐变处理的图片会被再次缓存，这个功能是 `ImageLoader` 不具备的。
4. `Glide` 的内存缓存分为两级`MemoryCache` 和 `ActiveResource` ，`MemoryCache` 和 `ImageLoader` 相同都是使用 `LruCache` ，`ActiveResource` 对正在使用的图片做了弱引用，防止使用中的 `资源` 被 `LRU` 算法回收掉。



文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我的公共号：

<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>