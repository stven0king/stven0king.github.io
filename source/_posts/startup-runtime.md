---
title: App Startup 源码分析
date: 2020-7-7 20:30:00
tags: [App Startup]
categories: Android
description: "我们可以通过使用 ContentProvider 初始化每个依赖关系来满足此需求，但是 ContentProvider 的实例化成本很高，并且可能不必要地减慢启动顺序。此外， ContentProvider 的初始化是无序的。App Startup 提供了一种更高效的方法，可在应用程序启动时初始化组件并显式定义其依赖关系。"
---




![startup](https://img-blog.csdnimg.cn/20200707194954333.jpeg)

上篇文章 [非侵入试获取Context进行SDK初始化](https://dandanlove.blog.csdn.net/article/details/107188633) 讲述了通过`ContentProvider` 进行 `SDK` 的初始化，文章末尾引出了 `App Startup` 。如果一个 `app` 依赖了很多需要初始化的 `sdk` ，如果都放在一个 `ContentProvider` 中会导致此 `ContentProvider` 代码数量增加。而如果每个sdk都采用同样的方式将会带来性能问题。`App Startup`可以有效解决这个问题。

[Jetpack StartUp官网](https://developer.android.google.cn/topic/libraries/app-startup) 

## 集成

使用 `startup` 在你的 `Android App` 或者 `Android Library` ，需要在你 `build.gradle` 添加下边依赖。

```gradle
dependencies {
    implementation "androidx.startup:startup-runtime:1.0.0-alpha01"
}
```

## 接入

`Apps` 和 `Library` 通常依赖于应用程序启动时立即初始化组件。

我们可以通过使用 `ContentProvider` 初始化每个依赖关系来满足此需求，但是  `ContentProvider` 的实例化成本很高，并且可能不必要地减慢启动顺序。此外，  `ContentProvider`  的初始化是无序的。

`App Startup` 提供了一种更高效的方法，可在应用程序启动时初始化组件并显式定义其依赖关系。

### 实现初始化组件

我们定义的每一个初始化组件必现实现 [Initializer](https://developer.android.google.cn/reference/kotlin/androidx/startup/Initializer) 接口，这个接口定义了两个方法 ：

```java
public interface Initializer<T> {
    @NonNull
    T create(@NonNull Context paramContext);
    @NonNull
    List<Class<? extends Initializer<?>>> dependencies();
}
```

-  `create()` ，这个方法会包含组件初始话的所有的操作，最终会返回一个实例 `T`；
- `dependencies()`，这个方法返回一组实现了 `Initializer<T>` 的类，这些都是当前组件初始化需要依赖的其他组件。可以使用此方法来控制应用程序在启动时运行初始化程序的顺序。

```java
// Initializes WorkManager.
class WorkManagerInitializer extends Initializer<WorkManager> {
    @Override
    public WorkManager create(Context context) {
        Configuration configuration = Configuration.Builder().build();
        WorkManager.initialize(context, configuration);
        return WorkManager.getInstance(context);
    }
    @Override
    public List<Class<Initializer<?>>> dependencies() {
        // No dependencies on other libraries.
        return emptyList();
    }
}
```

`WorkManagerInitializer` 组件的初始化不需要依赖其他的组件。

```java
// Initializes ExampleLogger.
class ExampleLoggerInitializer extends Initializer<ExampleLogger> {
    @Override
    public ExampleLogger create(Context context) {
        // WorkManager.getInstance() is non-null only after
        // WorkManager is initialized.
        return ExampleLogger(WorkManager.getInstance(context));
    }
    @Override
    public List<Class<Initializer<?>>> dependencies() {
        // Defines a dependency on WorkManagerInitializer so it can be
        // initialized after WorkManager is initialized.
        return Arrays.asList(WorkManagerInitializer.class);
    }
}
```

`ExampleLoggerInitializer` 组件的初始化需要依赖 `WorkManagerInitializer` 组件。

### 设置AndroidManifest条目

`InitializationProvider ` 是被 `App Startup` 包含一组特殊的 `Content Provider` 。使用它能发现和调用组件的初始化。

`InitializationProvider `  可以通过在 `AndroidManifest` 中配置的 `<meta-data>` 发现初始化组件。

`App Startup ` 通过调用 `dependencies()` 方法我们能发现其他的初始化组件。

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <!-- This entry makes ExampleLoggerInitializer discoverable. -->
    <meta-data  android:name="com.example.ExampleLoggerInitializer"
          android:value="androidx.startup" />
</provider>
```

我们不需要在 `AndroidManifest.xml` 中添加 `WorkManagerInitializer`，因为 `ExampleLoggerInitializer` 是依赖于 `WorkManagerInitializer` 。

### 手动初始化组件

当您使用 `App Startup `时，`InitializationProvider `对象使用名为 `AppInitializer `的实体在应用程序启动时自动发现并运行组件初始化程序。

但如果不想应用程序启动的时候进行组件初始化，那么可以进行手动初始化。这称为延迟初始化，它可以帮助最小化启动成本。

您必须首先对要手动初始化的所有组件禁用自动初始化。

#### 禁用单个组件的自动初始化

要禁用单个组件的自动初始化，请从清单中删除该组件的初始化程序的 `<meta-data>` 条目。

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data android:name="com.example.ExampleLoggerInitializer"
              tools:node="remove" />
</provider>
```

您可以在条目中使用 `tools:node="remove"`而不是简单地删除条目，以确保合并工具还从所有其他合并清单文件中删除了条目。

**禁用组件的自动初始化，也会禁用该组件的依赖项的自动初始化。**

#### 禁用所有组件的自动初始化

要禁用所有自动初始化，请从清单中删除 `InitializationProvider` 的整个条目：

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    tools:node="remove" />
```

#### 手动调用组件初始化程序

如果为组件禁用了自动初始化，则可以使用 `AppInitializer` 手动初始化该组件及其依赖项。

例如，以下代码调用 `AppInitializer` 并手动初始化 `ExampleLogger` 。

```java
AppInitializer.getInstance(context)
    .initializeComponent(ExampleLoggerInitializer.class);
```

由于 `WorkManager` 是 `ExampleLogger` 的依赖项，因此 `App Startup` 也将初始化 `WorkManager` 。

### 运行Lint检查

`App Startup` 库包含一组 `lint` 规则，可用于检查是否已正确定义了组件初始化程序。您可以通过从命令行运行 `./gradlew：app：lintDebug` 来执行这些 `lint` 检查。

## 源码分析

![startup-runtime-source-files](/Users/tanzx/Note/Android/第三方库/startup-runtime/startup-runtime-source-files.png)

先看一下源码的 ` aar` 结构。

### lint.jar

提供 `App Startup` 进行语义检查，本次不做分析。

### Androidmanifest.xml

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="androidx.startup" >
    <uses-sdk
        android:minSdkVersion="14"
        android:targetSdkVersion="29" />
    <application>
        <provider
            android:name="androidx.startup.InitializationProvider"
            android:authorities="${applicationId}.androidx-startup"
            android:exported="false"
            tools:node="merge" />
    </application>
</manifest>
```

我们可以看到该库兼容最小的 `Android` 版本为 `14`，该库当前适配的版本为 `19` 。

另一个就是自己注册的 `InitializationProvider` 。

### InitializationProvider

`App Startup` 的开始就是 `InitializationProvider` 的启动，我们从 `InitializationProvider` 这个进行分析就可以。

```java
public final class InitializationProvider extends ContentProvider {
    public boolean onCreate() {
        Context context = getContext();
        if (context != null) {
            AppInitializer.getInstance(context).discoverAndInitialize();
        } else {
            throw new StartupException("Context cannot be null");
        }
        return true;
    }
    /***其他代码省略***/
}
```

这里我们可以到 `InitializationProvider` 内部最终还是调用的 `AppInitializer` 进行初始化，这里只不过是利用了 `ContentProvider` 的自动启动而已。

### AppInitializer

这个类算不算是 `App Startup` 这个库的核心我不是很清楚。

- 他是整个库的代码核心；
- 他不是核心因为实现真的很简单，`App Startup` 这个库再我看来`InitializationProvider` 更有可取之处 ；

```java
public final class AppInitializer {
    private static final String SECTION_NAME = "Startup";
    private static AppInitializer sInstance;//单例
    private static final Object sLock = new Object();
    @NonNull
    final Map<Class<?>, Object> mInitialized;//组件只有一次初始化
    @NonNull
    final Context mContext;//application的上下文环境
    AppInitializer(@NonNull Context context) {
        this.mContext = context.getApplicationContext();
        this.mInitialized = new HashMap<>();
    }
    @NonNull
    public static AppInitializer getInstance(@NonNull Context context) {//DCL单例
        synchronized (sLock) {
            if (sInstance == null)
                sInstance = new AppInitializer(context);
            return sInstance;
        }
    }
    @NonNull
    public <T> T initializeComponent(@NonNull Class<? extends Initializer<T>> component) {
        return doInitialize(component, new HashSet<>());
    }
  	//进行组件初始化
    @NonNull
    <T> T doInitialize(@NonNull Class<? extends Initializer<?>> component, @NonNull Set<Class<?>> initializing) {
        //防止多线程并发
        synchronized (sLock) {
            boolean isTracingEnabled = Trace.isEnabled();
            try {
                Object result;
                if (isTracingEnabled)
                    Trace.beginSection(component.getSimpleName());
								//首先判断该组件是正在进行初始化，如果是那么抛异常
                if (initializing.contains(component)) {
                    String message = String.format("Cannot initialize %s. Cycle detected.", new Object[] { component
                            .getName() });
                    throw new IllegalStateException(message);
                }
                //首先判断该组件是否进行过初始化，如果已经初始化那么直接返回
                if (!this.mInitialized.containsKey(component)) {
                    initializing.add(component);//加入正在初始化的容器做记录
                    try {
                        Object instance = component.getDeclaredConstructor(new Class[0]).newInstance(new Object[0]);//反射进行构造
                        Initializer<?> initializer = (Initializer)instance;
                        List<Class<? extends Initializer<?>>> dependencies = initializer.dependencies();//获取依赖的初始化组件
                        if (!dependencies.isEmpty())//如果依赖不为空，那么先初始化依赖组件
                            for (Class<? extends Initializer<?>> clazz : dependencies) {
                                if (!this.mInitialized.containsKey(clazz))
                                    doInitialize(clazz, initializing);
                            }
                        //调用create获取组件初始化之后的实例
                        result = initializer.create(this.mContext);
                        initializing.remove(component);//从正在初始化的容器中移除
                        this.mInitialized.put(component, result);//加入已经初始化的容器做记录
                    } catch (Throwable throwable) {
                        throw new StartupException(throwable);
                    }
                } else {
                    result = this.mInitialized.get(component);
                }
                return (T)result;
            } finally {
                Trace.endSection();
            }
        }
    }
		//获取InitializationProvider中注册的组件进行初始化
    void discoverAndInitialize() {
        try {
            Trace.beginSection("Startup");
            ComponentName provider = new ComponentName(this.mContext.getPackageName(), InitializationProvider.class.getName());
            ProviderInfo providerInfo = this.mContext.getPackageManager().getProviderInfo(provider, 128);
            Bundle metadata = providerInfo.metaData;
            String startup = this.mContext.getString(R.string.androidx_startup);
            if (metadata != null) {
                Set<Class<?>> initializing = new HashSet<>();
                Set<String> keys = metadata.keySet();
                for (String key : keys) {
                    String value = metadata.getString(key, null);
                    if (startup.equals(value)) {
                        Class<?> clazz = Class.forName(key);
                        if (Initializer.class.isAssignableFrom(clazz)) {
                            Class<? extends Initializer<?>> component = (Class)clazz;
                            doInitialize(component, initializing);
                        }
                    }
                }
            }
        } catch (android.content.pm.PackageManager.NameNotFoundException|ClassNotFoundException exception) {
            throw new StartupException(exception);
        } finally {
            Trace.endSection();
        }
    }
}
```

## App Startup总结

优点：

- 解决了多个 `sdk ` 初始化导致 `Application`  文件和 `Mainfest` 文件需要频繁改动，维护困难的问题。
- 方便了 `sdk` 开发者在内部处理 `sdk` 的初始化问题，并且可以和调用者共享一个 `ContentProvider`。
- 处理了 `sdk` 之间的依赖关系，有效解耦，方便协同开发；

缺点：

- `ContentProvider` 的启动和反射构造 `Initializer` 在低版本系统中会有一定的性能损耗。
- 目前有些 `sdk` 的集成使用的就是 `ContentProvider` 这种无侵入试，多个 `ContentProvider` 此时有些浪费。

- 导致类文件增多，特别是有大量需要初始化的 `sdk` 存在时。
- 可能目前的版本还不是正式版，所以对 **多线程** 和 **多进程** 的考虑比较少。


文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！