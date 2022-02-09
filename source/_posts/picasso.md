---
title: Picasso源码分析和对比
date: 2019-12-30 17:57:35
tags: [Picasso]
categories: Android
description: "前面的Android-Universal-Image-Loader源码分析和Glide源码阅读理解一小时分别讲述了五年前和现在最受欢迎的Android图片加载库。今天讲述的picasso是Square公司开源的一个Android图片加载 "
---

前面的 [Android-Universal-Image-Loader源码分析](https://dandanlove.blog.csdn.net/article/details/103256724) 和 [Glide源码阅读理解一小时](https://dandanlove.blog.csdn.net/article/details/103564516) 分别讲述了五年前和现在最受欢迎的 `Android` 图片加载库。今天讲述的`picasso`是`Square`公司开源的一个`Android`图片加载库，可以实现图片下载和缓存功能。它 `ImageLoader` 和 `Glide` 的都有些相同和和不同点以及自己独特的点。


本文参考的 `Picasso` 源码的版本为 `2.71828` 。

[官网地址：https://square.github.io/picasso](https://square.github.io/picasso)

[GitHub地址：https://github.com/square/picasso](https://github.com/square/picasso)

# Picasso组成部分

<center>
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191230174525919.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)</center>

这幅图对应的是 `Picasso` 的主要组成部分。

- `Picasso` ：图片加载、转换、缓存的管理类。提供 `get` 方法获取默认的单例对象，也提供了 `Builder` 供业务方自定义配置生成自己的单例对象。
- `RequestCreator` ：提供易用的 `API ` ，内部维护了一个 `Request.Builder` 对象。用于构建图片下载的`Request` 。
- `Request` ：一个不可变的数据，用于控制图片使用之前的加载和变化。提供 `Builder` 进行数据的参数设置。
- `Action` ：图片架加载任务的请求包装，内部有 `picasso` 、`Request` 、`key` 和 `tag` 等。
- `Dispatcher` ：执行任务的分发器，以及任务的暂停、重复、回复等事件的处理。内部注册了 `BroadcastReceiver` 用来监测网络变化，从而进一步修改线程池的大小。
- `BitmapHunter` ：核心类负责任务执行具体操作，获取数据，解码数据为 `Bitmp` 。处理生成的 `Bitmap` 以及负责当前请求的 `transformation` 操作。
- `RequestHandler` ：用于自定义的请求处理类，需要重写 `canHandleRequest` 和 `load` 方法。`Picasso` 内部默认添加了7个 `RequestHandler` 子类。

# Picasso的获取

`Picasso` 的官网实例中 `Picasso.get()` 方式可以获取默认的 `Picasso` 的单例对象进行图片加载。

```java
//com.squareup.picasso.Picasso.java
static volatile Picasso singleton = null;
public static Picasso get() {
    if (singleton == null) {
        synchronized (Picasso.class) {
        if (singleton == null) {
            if (PicassoProvider.context == null) {
                throw new IllegalStateException("context == null");
            }
            singleton = new Builder(PicassoProvider.context).build();
            }
        }
    }
    return singleton;
}
```

`Picasso` 的单例使用了**双重校验锁（ DCL：double-checked locking)**，相关资料可以参考[Java版的7种单例模式](https://dandanlove.blog.csdn.net/article/details/101759634) 。

其中的 `Context` 不需要外部注入使用的是 `ContentProvider` 的 `Context` ，这个 `ContentProvider` 是`Picasso` 自己注册在 `AndroidManifest.xml` 中的。

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.squareup.picasso" >
    <uses-sdk android:minSdkVersion="14" />
    <application>
        <provider
            android:name="com.squareup.picasso.PicassoProvider"
            android:authorities="${applicationId}.com.squareup.picasso"
            android:exported="false" />
    </application>
</manifest>
```

# Picasso的配置

根据 `Picasso.Builder` 的我们可以知道我们能自定义那些 `Picasso` 的配置。

```java
//com.squareup.picasso.Picasso$Builder.java
public Picasso build() {
    Context context = this.context;
    if (downloader == null) {//默认的下载为okhttp
        downloader = new OkHttp3Downloader(context);
    }
    if (cache == null) {//默认的内存缓存使用的是LruCache
        cache = new LruCache(context);
    }
    if (service == null) {//默认的线程池
        service = new PicassoExecutorService();
    }
    if (transformer == null) {//请求转化接口
        transformer = RequestTransformer.IDENTITY;
    }
    Stats stats = new Stats(cache);
    Dispatcher dispatcher = new Dispatcher(context, service, HANDLER, downloader, cache, stats);
    //listener:检测Picasso加载图片过程中的失败以及异常
    //requestHandlers:自定义请求处理模块
    //defaultBitmapConfig:自定义生成Bitmap的配置
    //indicatorsEnabled：是否显示图片来源的指示器
    //loggingEnabled：是否打印Picasso的日志
    return new Picasso(context, dispatcher, cache, listener, transformer, requestHandlers, stats,
        defaultBitmapConfig, indicatorsEnabled, loggingEnabled);
}
```

显示指示器的图片加载完成之后会变为：

```java
public enum LoadedFrom {
    MEMORY(Color.GREEN),
    DISK(Color.BLUE),
    NETWORK(Color.RED);
    final int debugColor;
    LoadedFrom(int debugColor) {
      this.debugColor = debugColor;
    }
}
```
<center>
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191230174458691.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)</center>

# Picasso加载数据类型

`Picasso` 一共提供了4中 `load` 方法：

```java
public RequestCreator load(@Nullable Uri uri);
public RequestCreator load(@Nullable String path);
public RequestCreator load(@NonNull File file);
public RequestCreator load(@DrawableRes int resourceId);
```

其中 `String` 和 `File` 类型都会转化为 `Uri` 类型，所以 `Picasso` 支持 `Uri` 和 `ResourceId` 类型。

```java
Picasso.get().load(R.drawable.landing_screen).into(imageView1);
Picasso.get().load("file:///android_asset/DvpvklR.png").into(imageView2);
Picasso.get().load(new File(...)).into(imageView3);
```

# Request

`Picasso` 的简单实用：

```java
Picasso.get().load("http://i.imgur.com/DvpvklR.png").into(imageView);
```

`Picasso` 的对图片的一些变化实用：

```java
Picasso.get()
  .load(url)
  .resize(50, 50)
  .centerCrop()
  .into(imageView)
```

`Picasso` 的默认图以及错误处理默认图设置：

```java
Picasso.get()
    .load(url)
    .placeholder(R.drawable.user_placeholder)
    .error(R.drawable.user_placeholder_error)
    .into(imageView);
```

以上的这些设置都是在修改 `Request` 的成员变量的属性。

## RequestCreator的构造

我们知道 `Picasso` 支持加载 `Uri` 和 `ResourceId` ，所以我们先来看看这两个数据类型的 `load` 的方法。

```java
//com.squareup.picasso.Picasso.java
public RequestCreator load(@Nullable Uri uri) {
   return new RequestCreator(this, uri, 0);
}
public RequestCreator load(@DrawableRes int resourceId) {
   if (resourceId == 0) {
      throw new IllegalArgumentException("Resource ID must not be zero.");
   }
   return new RequestCreator(this, null, resourceId);
}
```

## RequestCreateor对Request的配置

- 可以设置 `tag` 标签，`Picasso` 可以在暂停、恢复请求的时候操作具有相同的 `tag` 标签的请求。比如 `Activity` 或者 `Context` 我们可以根据这个 `tag` 标签，做请求的**生命周期管理**，但是需要注意内存泄漏；
- 可以设置缓存的额外的 `Key` ，从而对同一个请求资源做不同的缓存处理；
- 设置请求的优先级；
- 设置内存缓存策略，以及网络请求缓存策略；
- 设置禁用从磁盘缓存或网络加载的图像的进行淡入浅出动画；
- 设置对图片的转化，转化前的图片必须在转化后手动回收；
- 设置可以等到图片加载完成确定宽、高之后再进行资源的加载；
- 设置对图片的宽、高、裁剪方式，旋转角度和解码配置等；

## Request的构造和Action的提交

我们通过 `Picasso.load` 方法构造 `RequestCreateor` ，`Request` 的构造再下一步调用 `RequestCreateor.into` 真正出发请求的时候产生。

```java
//com.squareup.picasso.RequestCreator.java
public void into(ImageView target) {
    into(target, null);
}
public void into(ImageView target, Callback callback) {
    long started = System.nanoTime();
    checkMain();
    if (target == null) {
      throw new IllegalArgumentException("Target must not be null.");
    }
    if (!data.hasImage()) {//判断uri为空，或者resourceid等于0
      picasso.cancelRequest(target);//取消在target上的请求
      if (setPlaceholder) {//如果设置默认占位图，那么默认图展现在target上
        setPlaceholder(target, getPlaceholderDrawable());
      }
      return;
    }
    if (deferred) {//判断是否需要延迟初始化
        if (data.hasSize()) {//如果已经设置了宽高，那么不能进行延迟请求
            throw new IllegalStateException("Fit cannot be used with resize.");
        }
        int width = target.getWidth();
        int height = target.getHeight();
        if (width == 0 || height == 0) {
            if (setPlaceholder) {//如果设置默认占位图，那么默认图展现在target上
                setPlaceholder(target, getPlaceholderDrawable());
            }//设置延迟加载，具体实现后续有讲解
            picasso.defer(target, new DeferredRequestCreator(this, target, callback));
            return;
        }
        //如果target已经确定边界那么不需要延迟请求
        data.resize(width, height);
    }
    Request request = createRequest(started);//创建Request,如果请求已更改，请复制原始ID和时间戳。
    String requestKey = createKey(request);//创建请求的key，用来标识请求和内存缓存
    if (shouldReadFromMemoryCache(memoryPolicy)) {//是否读取内存缓存（默认内存缓存，可读、可写）
        Bitmap bitmap = picasso.quickMemoryCacheCheck(requestKey);//从urlcache中获取
        if (bitmap != null) {//如果有缓存的bitmap
            picasso.cancelRequest(target);//取消请求，target设置bitmap同时标识数据来源为内存缓存
            setBitmap(target, picasso.context, bitmap, MEMORY, noFade, picasso.indicatorsEnabled);
            if (picasso.loggingEnabled) {
                log(OWNER_MAIN, VERB_COMPLETED, request.plainId(), "from " + MEMORY);
            }
            if (callback != null) {
                callback.onSuccess();
            }
            return;
        }
    }
    if (setPlaceholder) {//如果设置默认占位图，那么默认图展现在target上
        setPlaceholder(target, getPlaceholderDrawable());
    }//构建请求任务的Action，进行任务提交
    Action action = new ImageViewAction(picasso, target, request, memoryPolicy, networkPolicy, errorResId,errorDrawable, requestKey, tag, callback, noFade);
    picasso.enqueueAndSubmit(action);
}
```

上面这段代码主要包含了一下以及逻辑:

- 是否需要延迟请求，以便于确定加载图片的边界；
- 创建 `Request` 和 `requestkey` ；
  - `Request` 的创建主要是先调用 `RequestCreator` 对象中的 `Request.Builder` 的 `builder()` 方法构造，并且调用 `Picasso` 的请求转化操作进行 `请求` 的处理.
  - `requestKey` 的创建主要是根据当前 `Request` 的 `uri` 或`stableKey` 以及旋转角度、宽高、裁剪样式和转变操作等构造。
- 是否需要从内存缓存中读取并加载；
- 构建请求的任务 `Action` ，并进行提交；

```java
void enqueueAndSubmit(Action action) {
    Object target = action.getTarget();
    if (target != null && targetToAction.get(target) != action) {
        //检测是否在主线程，去掉target上的之前的action操作
        cancelExistingRequest(target);
        targetToAction.put(target, action);
    }
    //提交任务
    submit(action);
}
void submit(Action action) {
    dispatcher.dispatchSubmit(action);
}
```

`enqueueAndSubmit` 方法移除了 `ImageView` 上之前的操作，以便于加载新的图片。

# Action

上面说到了构造 `Action` ，我们这里来分析一下 `Picasso` 提供的 `Action` 类型。

> `GetAction` ：仅仅用来加载资源以及进行缓存，无任何回调；
>
> `FetchAction` ：用来加载资源以及进行缓存，只有成功失败回调，没有资源信息回调；
>
> `TargetAction` ：用来加载资源以及进行缓存，可以有有资源信息的成功、失败回调；
>
> `ImageViewAction` ：用来加载资源以及进行缓存，然后将产生的 `Bitmap` 加载在 `ImageView` 上。
>
> `RemoteViewAction` ：抽象类，用来加载资源以及进行缓存，然后将产生的 `Bitmap` 加载在 `RemoteView` 上。有两个实现类`NotificationAction` 和 `AppWidgetAction` ，分别对应通知栏和桌面小部件。

# Dispatcher

`Dispatcher` 作为任务的分发器会将提交任务、恢复任务、暂停任务、取消任务、任务完成、任务失败以及网络状态变化。

我先接着上一步讲述任务的提交：

```java
//com.squareup.picasso.Dispatcher.java
void dispatchSubmit(Action action) {
    //切换任务执行的线程(主线程到异步线程)
   handler.sendMessage(handler.obtainMessage(REQUEST_SUBMIT, action));
}
```

这里的 `handler` 是有着一个非主线程的 `Looper` ，为 `DispatcherHandler `。

```java
//com.squareup.picasso.Dispatcher.java
this.handler = new DispatcherHandler(dispatcherThread.getLooper(), this);
static class DispatcherThread extends HandlerThread {
    DispatcherThread() {
        //THREAD_PRIORITY_BACKGROUND=10
    	super(Utils.THREAD_PREFIX + DISPATCHER_THREAD_NAME, THREAD_PRIORITY_BACKGROUND);
    }
}
```

切换线程执行 `Action ` ：

```java
//com.squareup.picasso.Dispatcher.java
void performSubmit(Action action) {
    performSubmit(action, true);
}
void performSubmit(Action action, boolean dismissFailed) {
    //如果当前的tag处于paused那么将这个action加入到pausedActions容器
    if (pausedTags.contains(action.getTag())) {
        pausedActions.put(action.getTarget(), action);
        if (action.getPicasso().loggingEnabled) {
            log(OWNER_DISPATCHER, VERB_PAUSED, action.request.logId(),"because tag '" + action.getTag() + "' is paused");
        }
        return;
    }
    //判断是否已经有对应的Bitmap捕捉器
    BitmapHunter hunter = hunterMap.get(action.getKey());
    if (hunter != null) {//如果有那么将当前的action进行依附（情景：多个imageview加载同一个图片资源）
        hunter.attach(action);
        return;
    }
    if (service.isShutdown()) {//任务线程池是否关闭
        if (action.getPicasso().loggingEnabled) {
            log(OWNER_DISPATCHER, VERB_IGNORED, action.request.logId(), "because shut down");
        }
        return;
    }
    //从Picasso中获取对应的Bitmap捕捉器
    hunter = forRequest(action.getPicasso(), this, cache, stats, action);
    //提交Bitmap的捕捉任务
    hunter.future = service.submit(hunter);
    hunterMap.put(action.getKey(), hunter);
    if (dismissFailed) {//失败不做处理，从failedActions移除
        failedActions.remove(action.getTarget());
    }
    if (action.getPicasso().loggingEnabled) {
        log(OWNER_DISPATCHER, VERB_ENQUEUED, action.request.logId());
    }
}
```

# 线程池大小

`Dispatcher`  在构造函数中注册了广播，监听网络状态的变化从而修改线程池执行线程数量的大小。

```java
//com.squareup.picasso.PicassoExecutorService.java
private static final int DEFAULT_THREAD_COUNT = 3;
void adjustThreadCount(NetworkInfo info) {
    if (info == null || !info.isConnectedOrConnecting()) {
      setThreadCount(DEFAULT_THREAD_COUNT);
      return;
    }
    switch (info.getType()) {
      case ConnectivityManager.TYPE_WIFI://WIFI
      case ConnectivityManager.TYPE_WIMAX:
      case ConnectivityManager.TYPE_ETHERNET:
        setThreadCount(4);
        break;
      case ConnectivityManager.TYPE_MOBILE:
        switch (info.getSubtype()) {
            case TelephonyManager.NETWORK_TYPE_LTE:  // 4G
            case TelephonyManager.NETWORK_TYPE_HSPAP:
            case TelephonyManager.NETWORK_TYPE_EHRPD:
                setThreadCount(3);
                break;
            case TelephonyManager.NETWORK_TYPE_UMTS: // 3G
            case TelephonyManager.NETWORK_TYPE_CDMA:
            case TelephonyManager.NETWORK_TYPE_EVDO_0:
            case TelephonyManager.NETWORK_TYPE_EVDO_A:
            case TelephonyManager.NETWORK_TYPE_EVDO_B:
                setThreadCount(2);
                break;
            case TelephonyManager.NETWORK_TYPE_GPRS: // 2G
            case TelephonyManager.NETWORK_TYPE_EDGE:
                setThreadCount(1);
                break;
            default:
                setThreadCount(DEFAULT_THREAD_COUNT);
        }
        break;
      default:
        setThreadCount(DEFAULT_THREAD_COUNT);
    }
}
private void setThreadCount(int threadCount) {
    setCorePoolSize(threadCount);
    setMaximumPoolSize(threadCount);
}
```

# BitmapHunter

`BitmapHunter` 实现了 `Runnable ` 接口处理资源的加载，它内部有一个 `RequestHandler` 来 `load` 对应的 `Uri` 或者 `ResourceId` 。

## BitmapHunter的创建

```java
//com.squareup.picasso.BitmapHunter.java
static BitmapHunter forRequest(Picasso picasso, Dispatcher dispatcher, Cache cache, Stats stats, Action action) {
    Request request = action.getRequest();
    List<RequestHandler> requestHandlers = picasso.getRequestHandlers();
    //基于索引的循环,分配迭代器。
    for (int i = 0, count = requestHandlers.size(); i < count; i++) {
        RequestHandler requestHandler = requestHandlers.get(i);
        if (requestHandler.canHandleRequest(request)) {
            return new BitmapHunter(picasso, dispatcher, cache, stats, action, requestHandler);
        }
    }
    return new BitmapHunter(picasso, dispatcher, cache, stats, action, ERRORING_HANDLER);
}
```

`RequestHandler` 前面讲述过 `Piacasso` 自己默认添加了7种，而且我们也可以自定义。

```java
Picasso(Context context, //上线文环境
        Dispatcher dispatcher, //任务分发器
        Cache cache, //内存缓存
        Listener listener,//Picasso的异常回调
        RequestTransformer requestTransformer, //请求拦截处理类
        List<RequestHandler> extraRequestHandlers, //自定义的请求扩展处理
        Stats stats,//内存的状态
        Bitmap.Config defaultBitmapConfig, //Bitmap的解码配置
        boolean indicatorsEnabled, //图片右上角是否显示指示器标识图片来源
        boolean loggingEnabled) {//是否打印日志
    /***部分代码省略***/
    int builtInHandlers = 7; //默认7个，可添加自定义扩展
    //allRequestHandlers的总数
    int extraCount = (extraRequestHandlers != null ? extraRequestHandlers.size() : 0);
    List<RequestHandler> allRequestHandlers = new ArrayList<>(builtInHandlers + extraCount);
    //ResourceRequestHandler必须是列表中的第一个以避免强制其他RequestHandler对request.uri执行null检查涵盖（request.resourceId！= 0）的情况。 
    allRequestHandlers.add(new ResourceRequestHandler(context));
    if (extraRequestHandlers != null) {
        allRequestHandlers.addAll(extraRequestHandlers);
    }
    allRequestHandlers.add(new ContactsPhotoRequestHandler(context));
    allRequestHandlers.add(new MediaStoreRequestHandler(context));
    allRequestHandlers.add(new ContentStreamRequestHandler(context));
    allRequestHandlers.add(new AssetRequestHandler(context));
    allRequestHandlers.add(new FileRequestHandler(context));
    allRequestHandlers.add(new NetworkRequestHandler(dispatcher.downloader, stats));
    /***部分代码省略***/
}
```

我们从 `Picasso` 的构造中可以看出 `RequestHandler` 都有哪些：

- `ResourceRequestHandler` ：资源类型处理，`ResourceId` 类型的独有处理类；
- `ContactsPhotoRequestHandler` ：`ContactsPhoto` 请求处理器，加载`com.android.contacts/`下的图片。
- `MediaStoreRequestHandler` ：`MediaStore` 请求处理器，如果图片是存在`MediaStore`上的则用这个处理器处理。
- `ContentStreamRequestHandler` ：`content` 开头的类型，加载`ContentProvider` 提供的图片。
- `AssetRequestHandler` ：`file://android_asset/` 开头的类型，加载`asset`目录下的图片。
- `FileRequestHandler` ：`file` 开头的类型，处理文件类型的图片。
- `NetworkRequestHandler` ：`http` 或者 `https` 开头的类型，加载网络图片。

可能一个数据类型有多个处理的 `RequestHandler` ，但从生成 `BitmapHunter` 的方法我们可以看出来最终由 `RequestHandler` 的添加顺序确定。

## BitmapHunter.run

执行请求任务，以及进行相应的异常捕获处理。

```java
//com.squareup.picasso.BitmapHunter.java
@Override public void run() {
    try {
        updateThreadName(data);//更新线程名称以方便与打印日志
        if (picasso.loggingEnabled) {
            log(OWNER_HUNTER, VERB_EXECUTING, getLogIdsForHunter(this));
        }
        result = hunt();//获取Bitmap
        if (result == null) {//处理失败回调
            dispatcher.dispatchFailed(this);
        } else {//处理成功回调
            dispatcher.dispatchComplete(this);
        }
    } catch (NetworkRequestHandler.ResponseException e) {//处理网络响应异常
        if (!NetworkPolicy.isOfflineOnly(e.networkPolicy) || e.code != 504) {
            exception = e;
        }
        dispatcher.dispatchFailed(this);
    } catch (IOException e) {//处理io异常
        exception = e;
        dispatcher.dispatchRetry(this);
    } catch (OutOfMemoryError e) {//处理内存异常
        StringWriter writer = new StringWriter();
        //创建当前的内存数据快照，将其数据进行打印
        stats.createSnapshot().dump(new PrintWriter(writer));
        exception = new RuntimeException(writer.toString(), e);
        dispatcher.dispatchFailed(this);
    } catch (Exception e) {//处理其他异常
        exception = e;
        dispatcher.dispatchFailed(this);
    } finally {//线程结束
        Thread.currentThread().setName(Utils.THREAD_IDLE_NAME);
    }
}
```

# Bitmap的获取

```java
//com.squareup.picasso.BitmapHunter.java
Bitmap hunt() throws IOException {
    Bitmap bitmap = null;
    //是否允许从内存缓存中读取Bitmap
    if (shouldReadFromMemoryCache(memoryPolicy)) {
        bitmap = cache.get(key);
        if (bitmap != null) {//如果内存缓存有那么直接返回
            stats.dispatchCacheHit();
            loadedFrom = MEMORY;
            if (picasso.loggingEnabled) {
                log(OWNER_HUNTER, VERB_DECODED, data.logId(), "from cache");
            }
            return bitmap;
        }
    }
    //获取对应的网络数据处理状态
    networkPolicy = retryCount == 0 ? NetworkPolicy.OFFLINE.index : networkPolicy;
    //对应的RequestHandler加载数据得到Result
    RequestHandler.Result result = requestHandler.load(data, networkPolicy);
    if (result != null) {
        loadedFrom = result.getLoadedFrom();//获取数据来源（内存、磁盘、文件）
        exifOrientation = result.getExifOrientation();//获取旋转角度
        bitmap = result.getBitmap();//获取Bitmap
        //如果没有位图，则需要从流中对其进行解码。
        if (bitmap == null) {
            Source source = result.getSource();
            try {
                bitmap = decodeStream(source, data);
            } finally {
                try {//关闭数据流
                    source.close();
                } catch (IOException ignored) {
                }
            }
        }
    }
    if (bitmap != null) {//Bitmap不为空
        if (picasso.loggingEnabled) {
            log(OWNER_HUNTER, VERB_DECODED, data.logId());
        }//更新内存数据状态
        stats.dispatchBitmapDecoded(bitmap);
        //如果Bitmap需要变换或者需要进行方向调整
        if (data.needsTransformation() || exifOrientation != 0) {
            synchronized (DECODE_LOCK) {
                //是否可以进行方向的调整
                if (data.needsMatrixTransform() || exifOrientation != 0) {
                    //Bitmap使用矩阵进行方向的变换
                    bitmap = transformResult(data, bitmap, exifOrientation);
                    if (picasso.loggingEnabled) {
                        log(OWNER_HUNTER, VERB_TRANSFORMED, data.logId());
                    }
                }
                if (data.hasCustomTransformations()) {//是否自定义了Bitmap的转换
                    //处理Bitmap的变换，如果Bitmap有的新的对象，那么之前的Bitmap必须主动回收
                    bitmap = applyCustomTransformations(data.transformations, bitmap);
                    if (picasso.loggingEnabled) {
                        log(OWNER_HUNTER, VERB_TRANSFORMED, data.logId(), "from custom transformations");
                    }
                }
            }
            if (bitmap != null) {//有了新的Bitmap产生，更新内存数据状态
                stats.dispatchBitmapTransformed(bitmap);
            }
        }
    }
    return bitmap;
}
```

这里用一张网络图片举例说明怎么进行的数据获取：

我们根据 `Picasso` 中默认添加的  `RequestHandler`  了解到处理网络任务的 `NetworkRequestHandler` 。

```java
//com.squareup.picasso.NetworkRequestHandler.java
@Override public Result load(Request request, int networkPolicy) throws IOException {
    okhttp3.Request downloaderRequest = createRequest(request, networkPolicy);
    Response response = downloader.load(downloaderRequest);//OKHTTP进行下载返回Response
    ResponseBody body = response.body();//获取Request的Body
    if (!response.isSuccessful()) {//如果接口失败那么跑出异常并关闭响应
        body.close();
        throw new ResponseException(response.code(), request.networkPolicy);
    }
    //判断响应的数据是否为http的缓存
    Picasso.LoadedFrom loadedFrom = response.cacheResponse() == null ? NETWORK : DISK;
    //判断http的缓存是否无效
    if (loadedFrom == DISK && body.contentLength() == 0) {
        body.close();
        throw new ContentLengthException("Received response with 0 content-length header.");
    }
    //有了新的数据产生，更新内存数据状态
    if (loadedFrom == NETWORK && body.contentLength() > 0) {
      stats.dispatchDownloadFinished(body.contentLength());
    }//返回响应数据和数据来源
    return new Result(body.source(), loadedFrom);
}
```

# 磁盘缓存

从文章一开始的结构图和代码都没有体现**磁盘缓存** ，那么 `Picasso` 的 **磁盘缓存**怎么拥有的？为了回答这个问题我们先将上面的 `downloader.load` 来再次看一下，在一开始我们看到 `Picasso` 默认的 `downloader` 是 `OkHttp3Downloader` 。

> `Downloader` ：一种从外部资源（例如磁盘缓存和网络）加载图像的机制。

```java
public interface Downloader {
  //从互联网下载指定的图像。如果无法成功加载请求的URL，则抛出IOException。 
  @NonNull Response load(@NonNull okhttp3.Request request) throws IOException;
  //允许对此进行清理，包括关闭磁盘缓存和其他资源
  void shutdown();
}
```

这里我们就发现一个很有意思的现象，`Downloader` 的接口包含了 `Okhtt3` 的 `Request` 。所以这就限定了 `Picasso` 的请求只能使用 `Okhttp3` （毕竟都是 `Square`  公司的当然使用自己产品）。

```java
public final class OkHttp3Downloader implements Downloader {
    @VisibleForTesting final Call.Factory client;
    private final Cache cache;
    private boolean sharedClient = true;
    //创建使用OkHttp的新下载器。这会将图像缓存安装到您的应用程序中缓存目录。
    public OkHttp3Downloader(final Context context) {
        this(Utils.createDefaultCacheDir(context));
    }
    public OkHttp3Downloader(final File cacheDir) {
        this(cacheDir, Utils.calculateDiskCacheSize(cacheDir));
    }
    public OkHttp3Downloader(final Context context, final long maxSize) {
        this(Utils.createDefaultCacheDir(context), maxSize);
    }
    public OkHttp3Downloader(final File cacheDir, final long maxSize) {
        this(new OkHttpClient.Builder().cache(new Cache(cacheDir, maxSize)).build());
        sharedClient = false;
    }
    public OkHttp3Downloader(OkHttpClient client) {
        this.client = client;
        this.cache = client.cache();
    }
    public OkHttp3Downloader(Call.Factory client) {
        this.client = client;
        this.cache = null;
    }
    //同步执行
    @NonNull @Override public Response load(@NonNull Request request) throws IOException {
        return client.newCall(request).execute();
    }
    //关闭缓存
    @Override public void shutdown() {
        if (!sharedClient && cache != null) {
            try {
                cache.close();
            } catch (IOException ignored) {
            }
        }
    }
}
```

从 `Downloader` 的实现我们可以看出来我们 `Picasso` 的磁盘缓存是利用的 `Http` 协议做的磁盘缓存。

# 图片数据的呈现

我们在将 `Bitmap` 获取的之后，下一步就应该展现在 `ImageView` 上。除此之前还应该处理内存缓存、成功失败等回调。

## 内存缓存写入

```java
//com.squareup.picasso.Dispatcher.java
void performComplete(BitmapHunter hunter) {
    //如果允许写入内存缓存，那么将bitmap写入cache
    if (shouldWriteToMemoryCache(hunter.getMemoryPolicy())) {
        cache.set(hunter.getKey(), hunter.getResult());
    }
    //移除key对应的BitmpHunter
    hunterMap.remove(hunter.getKey());
    batch(hunter);//处理BitmapHunter
    if (hunter.getPicasso().loggingEnabled) {
        log(OWNER_DISPATCHER, VERB_BATCHED, getLogIdsForHunter(hunter), "for completion");
    }
}
private void batch(BitmapHunter hunter) {
    if (hunter.isCancelled()) {//如果Bitmap捕捉任务已经被取消
        return;
    }
    if (hunter.result != null) {//重建所有与待画位图相关的缓存。
        hunter.result.prepareToDraw();
    }
    batch.add(hunter);//将BitmapHunter添加都batch中进行批量的回调处理，500ms一批
    if (!handler.hasMessages(HUNTER_DELAY_NEXT_BATCH)) {
         handler.sendEmptyMessageDelayed(HUNTER_DELAY_NEXT_BATCH, BATCH_DELAY);
    }
}
```

> `prepareToDraw` ：绘制准备：重建该`bitmap`相关联的缓存来绘制。在可清除的`bitmap`中，此方法会尝试确保像素已经被解码。

## 交付数据

```java
//com.squareup.picasso.Dispatcher.java
void complete(BitmapHunter hunter) {
    Action single = hunter.getAction();//获取BitmapHunter的Action
    List<Action> joined = hunter.getActions();//获取获取BitmapHunter的Actions
    boolean hasMultiple = joined != null && !joined.isEmpty();//需要处理多个Action
    boolean shouldDeliver = single != null || hasMultiple;//是否有交付对象
    if (!shouldDeliver) {
        return;
    }
    Uri uri = hunter.getData().uri;//数据源的Uri
    Exception exception = hunter.getException();//数据加载过程中的异常
    Bitmap result = hunter.getResult();//数据对应Bitmap
    LoadedFrom from = hunter.getLoadedFrom();//数据来源
    if (single != null) {//Action的数据交付
      deliverAction(result, from, single, exception);
    }
    if (hasMultiple) {
        for (int i = 0, n = joined.size(); i < n; i++) {
            Action join = joined.get(i);//Action的数据交付
            deliverAction(result, from, join, exception);
        }
    }
    //如有有异常监听，并且有异常，那么进行回调
    if (listener != null && exception != null) {
        listener.onImageLoadFailed(this, uri, exception);
    }
}
//com.squareup.picasso.Picasso.java
private void deliverAction(Bitmap result, LoadedFrom from, Action action, Exception e) {
    if (action.isCancelled()) {//请求已经被取消
        return;
    }
    if (!action.willReplay()) {//是否需要等待监听网络状态重新加载
        targetToAction.remove(action.getTarget());
    }
    if (result != null) {
        if (from == null) {
            throw new AssertionError("LoadedFrom cannot be null.");
        }
        action.complete(result, from);//将Bitmap赋给View
    } else {
        action.error(e);
    }
}
//com.squareup.picasso.ImageViewAction.java
@Override 
public void complete(Bitmap result, Picasso.LoadedFrom from) {
    if (result == null) {
        throw new AssertionError(String.format("Attempted to complete action with no result!\n%s", this));
    }
    ImageView target = this.target.get();
    if (target == null) {
        return;
    }
    Context context = picasso.context;
    boolean indicatorsEnabled = picasso.indicatorsEnabled;//在target上显示Bitmap
    PicassoDrawable.setBitmap(target, context, result, from, noFade, indicatorsEnabled);
    if (callback != null) {
        callback.onSuccess();
    }
}
```

# Picasso延迟加载

为什么需要**延迟加载**呢？因为我们在`View` 上进行图片加载的时候不确定 `View` 是否已经被绘制完确定了宽、高。只有确定宽高我们才能从数据中解码出响应大小的 `Bitmap` 。所以**延迟加载**只是为了等待 `View` 被绘制完。

```java
class DeferredRequestCreator implements OnPreDrawListener, OnAttachStateChangeListener {
    private final RequestCreator creator;
    @VisibleForTesting final WeakReference<ImageView> target;
    @VisibleForTesting Callback callback;
    /***部分代码被省略***/
    @Override public void onViewAttachedToWindow(View view) {
        view.getViewTreeObserver().addOnPreDrawListener(this);
    }
    @Override public void onViewDetachedFromWindow(View view) {
        view.getViewTreeObserver().removeOnPreDrawListener(this);
    }
    @Override public boolean onPreDraw() {
        ImageView target = this.target.get();
        if (target == null) {
            return true;
        }
        ViewTreeObserver vto = target.getViewTreeObserver();
        if (!vto.isAlive()) {
            return true;
        }
        int width = target.getWidth();
        int height = target.getHeight();
        if (width <= 0 || height <= 0) {//宽、高不合法继续监听
            return true;
        }//移除监听，取消请求延迟，设置size大小，重写请求
        target.removeOnAttachStateChangeListener(this);
        vto.removeOnPreDrawListener(this);
        this.target.clear();
        this.creator.unfit().resize(width, height).into(target, callback);
        return true;
    }
    void cancel() {
        creator.clearTag();//清除tag
        callback = null;
        ImageView target = this.target.get();
        if (target == null) {
            return;
        }
        this.target.clear();//清除引用
        target.removeOnAttachStateChangeListener(this);//移除监听
        ViewTreeObserver vto = target.getViewTreeObserver();
        if (vto.isAlive()) {//移除监听
            vto.removeOnPreDrawListener(this);
        }
    }
    /***部分代码被省略***/
}
```

我们通过监听 `View` 在 `Window` 上的状态变化，以及监听 `View` 绘制来进行宽高的获取。

# 统计监控

```java
class Stats {
    private static final int CACHE_HIT = 0;//缓存命中
    private static final int CACHE_MISS = 1;//缓存没命中
    private static final int BITMAP_DECODE_FINISHED = 2;//图片解码
    private static final int BITMAP_TRANSFORMED_FINISHED = 3;//图片转码
    private static final int DOWNLOAD_FINISHED = 4;//数据下载完成
    /***部分代码省略***/
}
```

`Picasso` 通过 `Stats` 监控了内存和使用情况包括内存命中率，内存占用大小，流量消耗等。在产生 `OutOfMemoryError` 的时候会对当前内存的使用做一份快照并进行日志输出。

# 总结

前面的 [Android-Universal-Image-Loader源码分析](https://dandanlove.blog.csdn.net/article/details/103256724) 和 [Glide源码阅读理解一小时](https://dandanlove.blog.csdn.net/article/details/103564516) 有过 `Glide` 和 `ImageLoader` 的对比，这次我们将 `Picasso` 与这两个图片加载库再次进行对比。

>  `WEBP` ：在 `Android 4.0` **(API level 14)**中支持有损的`WebP`图像，在`Android 4.3`**(API level 18)**和更高版本中支持无损和透明的 `WebP` 图像。
<center>
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019123017442183.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)</center>

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！