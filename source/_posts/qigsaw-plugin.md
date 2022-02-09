---
title: Qigsaw源码之Gradle插件解析
date: 2021-01-25 20:01:24
tags: [Qigsaw]
categories: Android
description: "学习Qigsaw，阅读源码得到的理解，请指正"
---

![hi,2021](https://img-blog.csdnimg.cn/img_convert/5cdbb1b0ba4f46bbe9d8ef0f167ae337.png#pic_center)


[Android App Bundle](https://developer.android.com/platform/technology/app-bundle)为`Qigsaw`的前置依赖知识点。

[Android App Bundle](https://developer.android.com/platform/technology/app-bundle) 是`Android`新推出的一种官方发布格式`.aab`，可让您以更高效的方式开发和发布应用。借助 `Android App Bundle`，您可以更轻松地以更小的应用提供优质的使用体验，从而提升安装成功率并减少卸载量。转换过程轻松便捷。您无需重构代码即可开始获享较小应用的优势。改用这种格式后，您可以体验模块化应用开发和可自定义功能交付，并从中受益（PS：必须依赖于`GooglePlay`）。

`qigsaw`基于`AAB`实现，同时完全仿照`AAB`提供的`play core library`接口加载插件，开发查阅官方文档即可开始开发。如果有国际化需求的公司可以在国内版和国际版上无缝切换。同时`Qigsaw`实现`0 hook`，仅有少量私有 API 访问，保证其兼容性和稳定性。

[Github:Qigsaw](https://github.com/stven0king/Qigsaw)

本篇文章主要讲述`Qigsaw`相关的`plugin`。

## Qigsaw插件

主工程进行进行`apply plugin: 'com.iqiyi.qigsaw.application'`插件的依赖；

`feature`工程进行以下依赖：

```groovy
apply plugin: 'com.android.dynamic-feature'
apply plugin: 'com.iqiyi.qigsaw.dynamicfeature'
```

`gradle.properties`文件中配置`QIGSAW_BUILD=true`，才会有`feature`包的一些信息生成。

> com.iqiyi.qigsaw.application

`com.iqiyi.qigsaw.application.properties`文件内容为：

```java
implementation-class=com.iqiyi.qigsaw.buildtool.gradle.QigsawAppBasePlugin
```

`QigsawAppBasePlugin`默认会注册一个 `SplitComponentTransform`，在开启`QIGSAW_BUILD=true`之后还会注册`SplitResourcesLoaderTransform`。通过 `Transform`实现对插件内容的`AOP`。

`QigsawAppBasePlugin`除过注册两个`Transform`之外，为主要的是处理插件和基础包信息生成`Qigsaw`产物。

> com.iqiyi.qigsaw.dynamicfeature

`com.iqiyi.qigsaw.dynamicfeature.properties`文件内容为：

```java
implementation-class=com.iqiyi.qigsaw.buildtool.gradle.QigsawDynamicFeaturePlugin
```

`QigsawDynamicFeaturePlugin`在开启`QIGSAW_BUILD=true`之后会注册`SplitResourcesLoaderTransform`以及`SplitLibraryLoaderTransform`实现对插件内容的`AOP`。

### SplitResourcesLoaderTransform

主要是向`Activity`、`Service`和`Receiver`类中的`getResources`注入`SplitInstallHelper.loadResources(this, super.getResources())`。

```java
interface SplitComponentWeaver {
    /**
     * 链接目标
     */
    String CLASS_WOVEN = "com/google/android/play/core/splitinstall/SplitInstallHelper"
    /**
     * 链接方法
     */
    String METHOD_WOVEN = "loadResources"
    byte[] weave(InputStream inputStream)
}
```

相关注入类为：

```java
class SplitResourcesLoaderInjector {
    WaitableExecutor waitableExecutor
    /**
     * 预埋的 Activity
     */
    Set<String> activities
    Set<String> services
    Set<String> receivers
    SplitActivityWeaver activityWeaver
    SplitServiceWeaver serviceWeaver
    SplitReceiverWeaver receiverWeaver
    /**部分代码省略**/
}
```

其中基础包和插件的区别主要是注册的目标不同：

> 基础包只是读取`build.gradle`文件中的`qigsawSplit.baseContainerActivities`配置的`Activity`。

> 而插件需要读取`AndroidManifest.xml`文件中的`Activity`、`Service`和`Receiver`。

`SplitInstallHelper.loadResources(this, super.getResources());`的作用是将所有插件资源路径添加到`AssetManager`中，这样各个插件就可以访问所有的资源，关键实现代码如下:

```java
static Method getAddAssetPathMethod() throws NoSuchMethodException {
    if (addAssetPathMethod == null) {
        addAssetPathMethod = HiddenApiReflection.findMethod(AssetManager.class, "addAssetPath", String.class);
    }
    return addAssetPathMethod;
}
```

### SplitComponentTransform

该`Transform`主要进行了两个操作 ：

- 读取各个插件`apk`的`Manifest`文件，创建`ComponentInfo`类并将将各个插件`apk`的`Application`,`Activity`,`Service`,`Recevier`记录在该类的字段中，字段名称**以工程名+组件**类型命名，值为各个插件`apk`包含的组件，如过包含多个用逗号隔开。

```java
//com.iqiyi.android.qigsaw.core.extension.ComponentInfo
public class ComponentInfo {
   public static final String native_ACTIVITIES = "com.iqiyi.qigsaw.sample.ccode.NativeSampleActivity";
   public static final String java_ACTIVITIES = "com.iqiyi.qigsaw.sample.java.JavaSampleActivity";
   public static final String java_APPLICATION = "com.iqiyi.qigsaw.sample.java.JavaSampleApplication";
}
```

- 为每个`provider`创建代理类 类名为`String providerClassName=providerName+"Decorated"+splitName`，其中`providerName`为原始`provider`类名，`splitName`为插件`apk`对应的名称，并且该类继承`SplitContentProvider`。

```java
public class JavaContentProvider_Decorated_java extends SplitContentProvider {}
```

![provider_deccorated.png](https://img-blog.csdnimg.cn/img_convert/e30da5460cdeb47dafd2ad5eaf28a291.png#pic_center)


为啥这么做呢?

因为在`app`启动时`provider`的执行时机是比较靠前的，
`Application->attachBaseContext ==>ContentProvider->onCreate ==>Application->onCreate ==>Activity->onCreate`在这个过程中我们的插件apk并没有加载进来，一定会报`ClassNotFound`。所以我们将插件`apk`的`provider`生成一个代理类，然后替换掉，如果插件没有加载进来，代理类什么也不执行就可以了。很好的解决了我们的问题。

### SplitLibraryLoaderTransform

`SplitLibraryLoaderTransform`类进行的操作是向`dynamic-feature`构建`apk`的过程中，创建以 `"com.iqiyi.android.qigsaw.core.splitlib." + project.name + "SplitLibraryLoader"`的类。

```java
// com.iqiyi.android.qigsaw.core.splitlib.assetsSplitLibraryLoader
// com.iqiyi.android.qigsaw.core.splitlib.javaSplitLibraryLoader
// com.iqiyi.android.qigsaw.core.splitlib.nativeSplitLibraryLoader
package com.iqiyi.android.qigsaw.core.splitlib;
public class javaSplitLibraryLoader {
    public void loadSplitLibrary(String str) {
        System.loadLibrary(str);
    }
}
```

这个类的作用是啥呢？

下面我们来解释一下，你会发现很有趣的。

- `Qigsaw`是基于对于`com.google.android.play.core`对外暴露的方法，进行了自定义实现。因为`aab`目前只能对`google play`上发布应用起作用，所以开发者重新实现了一套`com.google.android.play.core`包名的第三方库，这样就可以做到在国内市场，与国外应用市场无缝迁移。
- `Qigsaw`提供两种加载方式加载插件`apk`，单`Classloader`和多`Classloader`模式，单`Classloader`涉及私有`api`访问，而多`Classloader`不涉及私有`api`访问。

该类的存在就是为了解决多`Classloader`模式下的**so加载**问题
`System.loadLibrary(str); `该方法会使用调用方的classloader从中获取so信息并加载。

```java
//java.lang.System.java
@CallerSensitive
public static void loadLibrary(String libname) {
  Runtime.getRuntime().loadLibrary0(Reflection.getCallerClass(), libname);
}
```

由于多`Classloader`模式下，每个插件都要各自的`Classloader`,`so`与`dex`都在各自的`Classloader`中记录，所以在多`Classloader`模式下， `System.loadLibrary`应由插件`apk`各自的`Classloader`调用。具体实现可参考`SplitLibraryLoaderHelper`类。

```java
//com.iqiyi.android.qigsaw.core.splitload.SplitLibraryLoaderHelper.java
private static boolean loadSplitLibrary0(ClassLoader classLoader, String splitName, String name) {
    try {
        Class<?> splitLoaderCl = classLoader.loadClass("com.iqiyi.android.qigsaw.core.splitlib." + splitName + "SplitLibraryLoader");
        Object splitLoader = splitLoaderCl.newInstance();
        Method method = HiddenApiReflection.findMethod(splitLoaderCl, "loadSplitLibrary", String.class);
        method.invoke(splitLoader, name);
        return true;
    } catch (Throwable ignored) {

    }
    return false;
}
```

## Qigsaw编译解析

### Qigsaw打包流程

![qigsaw_plugin_flow_chart.png](https://img-blog.csdnimg.cn/img_convert/4d922d7aa1f5ba0954ccf43743ca65ce.png#pic_center)


### copySplitManifestDebug

实现`feature`包下生成的`AndroidManifest.xml`文件的拷贝。

目标文件和地址：`featureName/build/intermediates/merged_manifests/debug/AndroidManifest.xml`。

拷贝后的地址：`app/build/intermediates/qigsaw/split-outputs/manifests/debug`。

拷贝后的文件名：`$featureName.xml`

### ProcessTaskDependenciesBetweenBaseAndSplitsWithQigsaw

触发**copySplitManifestDebug**任务，将`feature包`生成的产物和数据输出到`qigsawProcessDebugManifest`任务中。

### extractTargetFilesFromOldApk

将`app_debug.apk`解压 将`assets/`目录下所有内容释放到`app/build/intermediates/qigsaw/old-apk/target-files/xxx`中

### qigsawProcessDebugManifest

`SplitComponentTransform`创建的`$ContentProviderName_Decorated_$featureName`继承`SplitContentProvider`代替原有的`Provider`。

因为`Provider` 在应用启动的时候就需要加载，避免这个时候`feature`包没有下载下来，先加载一个代理的`Provider`。

![provider.png](https://img-blog.csdnimg.cn/img_convert/dbbbea28377246fd42fa062e276d8523.png#pic_center)


### generateDebugQigsawConfig

生成以下文件：

```java
@Keep
public final class QigsawConfig {
    public static final String DEFAULT_SPLIT_INFO_VERSION = "1.0.0_1.0.0";
    public static final String[] DYNAMIC_FEATURES = {"java", "assets", "native"};
    public static final String QIGSAW_ID = "1.0.0_c40ab5d";
    public static final boolean QIGSAW_MODE = Boolean.parseBoolean("true");
    public static final String VERSION_NAME = "1.0.0";
}
```

`QIGSAW_ID`回先获取基础包的`id`，如果没有那么为当前的`QigsawId`。

### processSplitApkDebug

每个`feature`都需要执行的任务，分别处理自己的的`apk`并生成对应的`json`文件。

- 将`feature`包的`apk`文件解压到`app/build/intermediates/qigsaw/split-outputs/unzip/debug/$featureName`文件；

- 遍历解压`apk`中的`lib`文件目录，找到支持的**ABI**;

- 如果有`lib`文件有`so`文件，那么在该目录生成一个`AndroidManifest.xml`文件；

  - 将`lib`文件和生成的`AndroidManifest.xml`压缩为`protoAbiApk`;

  - 利用`aapt2`工具将 `protoAbiApk`到 `binaryAbiApk`中;

  - 将`binaryAbiApk`进行签名生成`app/build/intermediates/qigsaw/split-outputs/apks/debug/$feature-$abi.apk`;

  - 生成`SplitInfo.SplitApkData`数据；

    ```json
    {
      "abi": "x86",
      "url": "assets://qigsaw/native-x86.zip",
      "md5": "03a29962b87c6ed2a7961b6dbe45f532",
      "size": 8539
    }
    ```

- 遍历解压`apk`中**除过lib**之前的文件目录，压缩为`$fearure-master-unsigned.apk`。签名生成`app/build/intermediates/qigsaw/split-outputs/apks/debug/$feature-master.apk`;

- 生成`SplitInfo.SplitApkData`数据；

  ```json
  {
    "abi": "master",
    "url": "assets://qigsaw/native-master.zip",
    "md5": "3b89066aeaf7d2c2a59b4f3a10fef345",
    "size": 12824
  }
  ```

- 更具`lib`文件下的数据生成`SplitInfo.SplitLibData`数据；

  ```json
  {
    "abi": "arm64-v8a",
    "jniLibs": [
      {
        "name": "libhello-jni.so",
        "md5": "2938d8b40825e82715422dbdba479e4f",
        "size": 5896
      }
    ]
  }
  ```

- 最后生成每个`feature`的`SplitInfo`数据，写入`/app/build/intermediates/qigsaw/split-outputs/split-info/debug/$featureName.json`文件；

  ```java
  public class SplitInfo implements Cloneable, GroovyObject {
      private String splitName;//feature包名称
      private boolean builtIn;//!onDemand||!releaseSplitApk(releaseSplitApk是gradle中配置项)
      private boolean onDemand;//取自AndroidManifest.xml中的onDemand
      private String applicationName;//feature应用名
      private String version;//feature包中的versionname@versioncode
      private int minSdkVersion;//feature最低版本
      private int dexNumber;//feature包中的dex数量
      private Set<String> dependencies;//feature包的依赖；
      private Set<String> workProcesses;//feature包AndroidManifest.xml中的Activity、Service、Receiver、provider配置的进程；
      private List<SplitInfo.SplitApkData> apkData;//SplitInfo.SplitApkData数据
      private List<SplitInfo.SplitLibData> libData;//SplitInfo.SplitLibData数据
  }
  ```


### qigsawAssembleDebug

- 将`build/intermediates/qigsaw/split-outputs/split-info/debug`中的每个`feature`包生成的`json`合并；
- 将**合并之后**的文件与基础包中的`Qigsaw`配置文件进行对比，生成新的`增量Qigsaw`配置文件；
  - 对比规则是`verisonName`相等的时候对比`split.version`，有一个不同就表示有更新;
  - 如果有更新，那么`QigsawId`为基础包的`QigsawId`，并分析和修改`split`信息；
    - 修改`split`信息的时候，相同的`splitName`对比`split.version`。如果相同那么`split`使用基础包的`split`信息，如果**不同**那么该`split`的`builtIn=false`，`onDemand=true`。并将有更新的`split`做记录（`updatesplits`字段值）。此时`updateMode`值为**VERSION_CHANGED=1**；
    - 没有任何修改，那么`updateMode`值为**VERSION_NO_CHANGED=2**；
    - 如果没有基础包，那么`updateMode`值为**DEFAULT=0**；
- 分别判断如果`feature`包的`builtIn`是**false**；
  - 判断是否有上传服务，如有有那么上传`feature`包。上传成功后将对应的`url`地址修改为可下载的`http`地址。如果地址为空，或者不是`http`开头会跑异常。
  - 如果没有实现上传服务那么`builtIn`置为**true**；
- 格式化`split`内容，写到`build/intermediates/qigsaw/split-details/debug`文件目录下。
- 将`updateMode`值写到`build/intermediates/qigsaw/split-details/debug/_update_record_.json`文件。
- 如果`updateMode`值为**VERSION_NO_CHANGED**，那么将`intermediates/qigsaw/old-apk/target-files/debug/assets/qigsaw/qigsaw_*.json`文件拷贝到`app/build/intermediates/merged_assets/debug/out/qigsaw/qigsaw_*.json`;
  - 否则将`app/build/intermediates/qigsaw/split-details/debug/qigsaw_*.json`文件拷贝到`app/build/intermediates/merged_assets/debug/out/qigsaw/qigsaw_*.json`;
- 向`app/build/intermediates/qigsaw/split-details/debug/base.app.cpu.abilist.properties `写入支持的`abi`，并将其拷贝到`app/build/intermediates/merged_assets/debug/out/`下面；
- 遍历`feature`生成的`splitinfo`信息，如果`builtIn`是**true**;
  - 如果`updateMode`值为**DEFAULT=0**，将将`app/build/intermediates/qigsaw/split-outputs/apks/debug/*.apk`拷贝到`app/build/intermediates/merged_assets/debug/out/qigsaw/*.zip`；
  - 如果`updateMode`值为**DEFAULT!=0**，判断该`feature`是否是在`updateSplits`中;
    - 如果是那么将`app/build/intermediates/qigsaw/split-outputs/apks/debug/*.apk`拷贝到`app/build/intermediates/merged_assets/debug/out/qigsaw/*.zip`；
    - 如果不是将`app/build/intermediates/qigsaw/old-apk/target-files/debug/assets/qigsaw/*.zip`拷贝到`app/build/intermediates/merged_assets/debug/out/qigsaw/*.zip`；

### 产物

`Qigsaw`配置文件

```json
{
  "qigsawId": "1.0.0_c40ab5d",
  "appVersionName": "1.0.0",
  "updateSplits": [
    "java"
  ],
  "splits": [
    {
      "splitName": "java",
      "builtIn": true,
      "onDemand": false,
      "applicationName": "com.iqiyi.qigsaw.sample.java.JavaSampleApplication",
      "version": "1.1@1",
      "minSdkVersion": 14,
      "dexNumber": 2,
      "workProcesses": [
        ""
      ],
      "apkData": [
        {
          "abi": "master",
          "url": "assets://qigsaw/java-master.zip",
          "md5": "658bc419a9d3c7812a36e61f6c5be4c4",
          "size": 12822
        }
      ]
    }
    {
      "splitName": "native",
      "builtIn": true,
      "onDemand": true,
      "version": "1.0@1",
      "minSdkVersion": 14,
      "dexNumber": 2,
      "apkData": [
        {
          "abi": "arm64-v8a",
          "url": "assets://qigsaw/native-arm64-v8a.zip",
          "md5": "b01ad63db38a4ec5fad3284c573a02d3",
          "size": 8545
        },
        {
          "abi": "master",
          "url": "assets://qigsaw/native-master.zip",
          "md5": "3c41745a16a31e967cde8247009463f1",
          "size": 12824
        }
      ],
      "libData": [
        {
          "abi": "arm64-v8a",
          "jniLibs": [
            {
              "name": "libhello-jni.so",
              "md5": "2938d8b40825e82715422dbdba479e4f",
              "size": 5896
            }
          ]
        }
      ]
    }
  ]
}
```

`Qigsaw`加载的压缩包

![app-debug.png](https://img-blog.csdnimg.cn/img_convert/4aae1c3a10d593b721822c4c78332351.png#pic_center)


### 下期研究知识点

- 混淆相关使用操作；
- `Tinker`热修改相关使用操作；

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！