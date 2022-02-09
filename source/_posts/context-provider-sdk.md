---
title: 非侵入试获取Context进行SDK初始化
date: 2020-7-7 20:25:00
tags: [Context]
categories: Android
description: "业务SDK集成可以不需要代码么？"
---

![Context](https://img-blog.csdnimg.cn/20200707194057310.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

# 非侵入试获取Context进行SDK初始化

当我们在使用第三方**SDK**，或者自己进行**SDK**封装时，如果需要需要用到 `Context` 进行初始化时，一般做法就是将初始化方法暴露给调用方，让调用方在初始化**SDK**时，传入上下文环境。

```java
publi class App extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        MultiDex.install(getApplication());
        if (Config.Debug)) {
            ARouter.openLog();
            ARouter.openDebug();
        }
        ARouter.init(this);
        SkinSdk.init(this);
        PaySDK.install(this);
        /***部分代码省略***/
    }
}
```

以上的 `SDK` 初始化代码是不是感觉很难维护。有没有一种直接拿来用而不需要进行 **显式** 初始化的SDK集成方式呢？

我们知道 `ContentProvider` 的生命周期，它是在 `Application.attach` 之后 `Application.onCreate` 之前进行 `installProvider` 。不需要我们 **显式** 通过代码启动。、

所以他满足了我们一般初始化 **SDK** 的条件：

- 拥有 `Context[Application]` 的上下文环境；
- 可以进行自动启动；

如果大家平时注意观察会发现我们平时使用的一些**SDK**也是不需要显示初始化的，而他们都是使用自定义的 `ContentProvider` 这种方式。

## [picasso](https://github.com/square/picasso) 初始化

相关知识可以阅读 [Picasso源码分析和对比](https://editor.csdn.net/md/?articleId=103770389) 这篇文章。

```java
@RestrictTo(LIBRARY)
public final class PicassoProvider extends ContentProvider {
  @SuppressLint("StaticFieldLeak") static Context context;
  @Override public boolean onCreate() {
    context = getContext();
    return true;
  }
  /***部分代码省略***/
}
```

无侵入试的 `Context` 获取，依靠 `ContentProvider` 通过注册在 `AndroidManifest.xml` 实现自动启。

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

`picasso` 的初始化：

```java
public class Picasso {
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
}
```

## InstantRun

相关知识可以阅读 [InstantRun从2.0到3.0，历史解毒](https://editor.csdn.net/md/?articleId=80365174) 这篇文章。

```xml
<application android:theme="@style/AppTheme" 
             android:label="@string/app_name" 
             android:icon="@mipmap/ic_launcher"
             android:name="com.example.tzx.changeskin.MyApplication" 
             android:debuggable="true" 
             android:testOnly="true" 
             android:allowBackup="true" 
             android:supportsRtl="true">
        <activity android:name="com.example.tzx.changeskin.InstantRunActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <provider android:name="com.android.tools.ir.server.InstantRunContentProvider"
                  android:multiprocess="true"          android:authorities="com.example.tzx.changeskin.com.android.tools.ir.server.InstantRunContentProvider"/>
</application>
```

`InstantRunContentProvider` 源码：

```java
public final class InstantRunContentProvider extends ContentProvider {
    public boolean onCreate() {
        if (isMainProcess()) {
            Log.i(Logging.LOG_TAG, "starting instant run server: is main process");
            Server.create(getContext());
        } else {
            Log.i(Logging.LOG_TAG, "not starting instant run server: not main process");
        }
        return true;
    }
    private boolean isMainProcess() {
        boolean isMainProcess = false;
        if (AppInfo.applicationId == null) {
            return false;
        }
        boolean foundPackage = false;
        int pid = Process.myPid();
        for (RunningAppProcessInfo processInfo : ((ActivityManager) getContext().getSystemService("activity")).getRunningAppProcesses()) {
            if (AppInfo.applicationId.equals(processInfo.processName)) {
                foundPackage = true;
                if (processInfo.pid == pid) {
                    isMainProcess = true;
                    break;
                }
            }
        }
        if (isMainProcess || foundPackage) {
            return isMainProcess;
        }
        Log.w(Logging.LOG_TAG, "considering this process main process:no process with this package found?!");
        return true;
    }
    /***部分代码省略***/
}
```

## Leakcanary

进行 `LeakCanary` 集成的 `apk` 文件中的 `AndroidManifest.xml` 会自动添加一下的配置：

```xml
<provider
		android:name="leakcanary.internal.AppWatcherInstaller$MainProcess"
    android:exported="false"
    android:authorities="com.tzx.androidcode.leakcanary-installer" />
```

`AppWatcherInstaller$MainProcess` 源码：

```kotlin
internal sealed class AppWatcherInstaller : ContentProvider() {
  /**
   * [MainProcess] automatically sets up the LeakCanary code that runs in the main app process.
   */
  internal class MainProcess : AppWatcherInstaller()
  /**
   * When using the `leakcanary-android-process` artifact instead of `leakcanary-android`,
   * [LeakCanaryProcess] automatically sets up the LeakCanary code
   */
  internal class LeakCanaryProcess : AppWatcherInstaller() {
    override fun onCreate(): Boolean {
      super.onCreate()
      AppWatcher.config = AppWatcher.config.copy(enabled = false)
      return true
    }
  }
  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    InternalAppWatcher.install(application)
    return true
  }
  /***部分代码省略***/
}
```

## App Startup

[Jetpack StartUp官网](https://developer.android.google.cn/topic/libraries/app-startup) 

我们可以通过使用 `ContentProvider` 初始化每个依赖关系来满足此需求，但是  `ContentProvider` 的实例化成本很高，并且可能不必要地减慢启动顺序。此外，  `ContentProvider`  的初始化是无序的。

`App Startup` 提供了一种更高效的方法，可在应用程序启动时初始化组件并显式定义其依赖关系。

[App Startup 源码分析](https://dandanlove.blog.csdn.net/article/details/107188896)

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！