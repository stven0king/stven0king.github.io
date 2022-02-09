---
title: Android8.0隐式广播和自定义签名权限
date: 2021-01-23 17:17:21
tags: [Android8.0,签名权限]
categories: Android
description: "记录一下今天同事给我分享的比较有意思的Bug，在已有的已经在AndroidManifest.xml中注册的广播在部分手机上无法通过Action隐式启动。上网搜搜资料自己写了个Demo，Mark一下~！~！ "
---

![我思故我在](https://img-blog.csdnimg.cn/img_convert/6346ddc0ece7034f9b69d509ed67751e.png#pic_center)


## 前言

记录一下今天同事给我分享的比较有意思的Bug，在已有的已经在`AndroidManifest.xml`中注册的广播在部分手机上无法通过`Action`隐式启动。上网搜搜资料自己写了个Demo，Mark一下~！~！

[Android官网：Oreo后台执行限制](https://developer.android.com/about/versions/oreo/background)

我们这里主要看对于广播的影响，摘抄一段官网上的介绍：

## 广播限制

如果应用注册为接收广播，则在每次发送广播时，应用的接收器都会消耗资源。 如果多个应用注册为接收基于系统事件的广播，则会引发问题：触发广播的系统事件会导致所有应用快速地连续消耗资源，从而降低用户体验。 为了缓解这一问题，`Android 7.0`（API 级别 24）对广播施加了一些限制，如[后台优化](https://developer.android.com/topic/performance/background-optimization)中所述。 `Android 8.0`（API 级别 26）让这些限制更为严格。

- 适配 `Android 8.0` 或更高版本的应用无法继续在其清单中为隐式广播注册广播接收器。 *隐式广播*是一种不专门针对该应用的广播。 例如，`ACTION_PACKAGE_REPLACED` 就是一种隐式广播，因为该广播将被发送给所有已注册侦听器，让后者知道设备上的某些软件包已被替换。 不过，`ACTION_MY_PACKAGE_REPLACED` 不是隐式广播，因为不管已为该广播注册侦听器的其他应用有多少，它都会只被发送给软件包已被替换的应用。
- 应用可以继续在它们的清单中注册显式广播。
- 应用可以在运行时使用 `Context.registerReceiver()` 为任意广播（不管是隐式还是显式）注册接收器。
- 需要[签名权限](https://developer.android.com/guide/topics/manifest/permission-element#plevel)的广播不受此限制所限，因为这些广播只会发送到使用相同证书签名的应用，而不是发送到设备上的所有应用。

在许多情况下，之前注册隐式广播的应用使用 `JobScheduler` 作业可以获得类似的功能。 例如，一款社交照片应用可能需要不时地执行数据清理，并且倾向于在设备连接到充电器时执行此操作。 之前，应用已经在清单中为 `ACTION_POWER_CONNECTED` 注册了一个接收器；当应用接收到该广播时，它会检查清理是否必要。 为了迁移到 `Android 8.0` 或更高版本，应用将该接收器从其清单中移除。 应用将清理作业安排在设备处于空闲状态和充电时运行。

**请注意：**很多隐式广播当前已不受此限制所限。 应用可以继续在其清单中为这些广播注册接收器，不管应用适配哪个 API 级别。 有关已豁免广播的列表，请参阅[隐式广播例外](https://developer.android.com/guide/components/broadcast-exceptions)。 

更具上面的描述，我们可以得到一下几点：
1. 适配`Android 8.0`或更高版本的应用无法继续在其清单中为隐式广播注册广播接收器；
2. 应用可以继续在它们的清单中注册显式广播；
3. 推荐运行时使用`Context.registerReceiver()`注册广播；
4. 需要**签名权限**的广播不受此约束；

## 自定义权限

[Android官网：permission](https://developer.android.com/guide/topics/manifest/permission-element#plevel)


```xml
<permission android:description="string resource"
    android:icon="drawable resource"
    android:label="string resource"
    android:name="string"
    android:permissionGroup="string"
    android:protectionLevel=["normal" | "dangerous" |"signature" | ...] />
```

### android:protectionLevel
说明权限中隐含的潜在风险，并指示系统在确定是否将权限授予请求授权的应用时应遵循的流程。

每个保护级别都包含基本权限类型以及零个或多个标记。例如，`dangerous`保护级别没有标记。相反，保护级别 `signature|privileged`是`signature`基本权限类型和`privileged`标记的组合。

下表列出了所有基本权限类型。如需查看标记列表，请参阅 [protectionLevel](https://developer.android.com/reference/android/R.attr#protectionLevel)。

|值|	含义|
|:--|:--|
|`normal`|	默认值。具有较低风险的权限，此类权限允许请求授权的应用访问隔离的应用级功能，对其他应用、系统或用户的风险非常小。系统会自动向在安装时请求授权的应用授予此类权限，无需征得用户的明确许可（但用户始终可以选择在安装之前查看这些权限）。|
|`dangerous`|	具有较高风险的权限，此类权限允许请求授权的应用访问用户私人数据或获取可对用户造成不利影响的设备控制权。由于此类权限会带来潜在风险，因此系统可能不会自动向请求授权的应用授予此类权限。例如，应用请求的任何危险权限都可能会向用户显示并且获得确认才会继续执行操作，或者系统会采取一些其他方法来避免用户自动允许使用此类功能。|
|`signature`|	只有在请求授权的应用使用与声明权限的应用相同的证书进行签名时系统才会授予的权限。如果证书匹配，则系统会在不通知用户或征得用户明确许可的情况下自动授予权限。|
|`signatureOrSystem`	|`signature\privileged` 的旧同义词。在**API级别23**中已弃用。系统仅向位于`Android`系统映像的专用文件夹中的应用或使用与声明权限的应用相同的证书进行签名的应用授予的权限。不要使用此选项，因为 `signature` 保护级别应足以满足大多数需求，无论应用安装在何处，该保护级别都能正常发挥作用。`signatureOrSystem`权限适用于以下特殊情况：多个供应商将应用内置到一个系统映像中，并且需要明确共享特定功能，因为这些功能是一起构建的。|

### 自定义签名权限并使用
```xml
<permission
        android:protectionLevel="signature"
        android:name="com.xx.xx.receiver" />
<uses-permission android:name="com.xx.xx.receiver"/>
```

## 测试
我写个Demo测试一下，测试机`MI 8`，系统为`Android 10`。

声明两个`Broadcast`,一个带权限，一个不带权限。
```kotlin
public open class CustomReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
        if(intent != null) {
            var param = intent.getStringExtra("message")
            var result:String?
            result = String.format("get the receiver for %s", param)
            XToast.show(result)
        }
    }
}
public class CustomReceiver2 : CustomReceiver() {}
```
```xml
<!--带签名权限的广播-->
<receiver android:name=".main.receiver.CustomReceiver"
    android:permission="com.xx.xx.receiver">
    <intent-filter>
        <action android:name="com.xx.xx.message"/>
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</receiver>
<!--普通的广播-->
<receiver android:name=".main.receiver.CustomReceiver2">
    <intent-filter>
        <action android:name="com.xx.xx.message2"/>
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</receiver>
```

1. > 发送权限隐式广播-不加权限
```kotlin
var intent = Intent("com.xx.xx.message" + num++)
intent.putExtra("message","custom test" + num++)
sendBroadcast(intent)
```
结果：吐司无法展现。
```java
15:59:23.420#1635#1758#W#BroadcastQueue #Background execution not allowed: receiving Intent { act=com.xx.xx.message flg=0x10 (has extras) } to com.xx.xx.demo/.main.receiver.CustomReceiver
```

2. > 发送权限隐式广播-加权限
```kotlin
var intent = Intent("com.xx.xx.message")
intent.putExtra("message","custom test")
sendOrderedBroadcast(intent, "com.xx.xx.receiver")
```
结果：吐司展现。

3. > 发送权限隐式广播-加权限-错误
```kotlin
var intent = Intent("com.xx.xx.message")
intent.putExtra("message","custom test")
sendOrderedBroadcast(intent, "com.xx.xx.receiver.error")
```
结果：吐司无法展现。
```java
15:44:54.514#1696#1774#W#BroadcastQueue #Permission Denial: receiving Intent { act=com.xx.xx.message flg=0x1000010 (has extras) } to com.xx.xx.ztemplate/.main.receiver.CustomReceiver requires com.xx.xx.receiver.error due to sender com.xx.xx.demo (uid 10547)
```

4. > 发送隐式广播
```kotlin
var intent = Intent("com.xx.xx.message2")
intent.putExtra("message","custom test")
sendBroadcast(intent)
```
结果：吐司无法展现。

```java
15:59:23.420#1635#1758#W#BroadcastQueue #Background execution not allowed: receiving Intent { act=com.xx.xx.message flg=0x10 (has extras) } to com.xx.xx.demo/.main.receiver.CustomReceiver
```

5. > 发送隐式广播-添加package
```kotlin
var intent = Intent("com.xx.xx.message2")
intent.`package` = "com.xx.xx.demo"
intent.putExtra("message","custom test")
sendBroadcast(intent)
```
结果：吐司展现。

6. > 发送隐式广播-setClass(等同于添加component)
```kotlin
var intent = Intent("com.xx.xx.message2")
intent.setClass(this, CustomReceiver2::class.java)
intent.putExtra("message","custom test")
sendBroadcast(intent)
```
结果：吐司展现。

其实第`5`和第`6`个case已经不算隐式广播了，他们都为`Intent`设置了`package`指明了当前的环境。

## 错误分析

>  `BroadcastQueue #Permission Denial:`

![BroadcastQueue-permission-denial.png](https://img-blog.csdnimg.cn/img_convert/c161a43371831e84fbe2db7c36a2f342.png#pic_center)


这里提示权限有问题，需要添加或修改权限。

> `BroadcastQueue #Background execution not allowed:`

![BroadcastQueue-not-allow.png](https://img-blog.csdnimg.cn/img_convert/16522b7278dc8b09f5b84e9cf029c5ed.png#pic_center)


```java
//android.content.Intent.java
public static final int FLAG_RECEIVER_INCLUDE_BACKGROUND = 0x01000000;
public static final int FLAG_RECEIVER_EXCLUDE_BACKGROUND = 0x00800000;
```

这个代码块有一个`||`操作，我们不想让其进入到改逻辑需要使前面判断为`false`，后面判断为`false`。

判断`r.intent.getFlags()&Intent.FLAG_RECEIVER_EXCLUDE_BACKGROUND` 中携带了`FLAG_RECEIVER_EXCLUDE_BACKGROUND`标志位。我们一般都不会携带，所以前面逻辑为`false`。

后面逻辑有三个`&&`操作，那么只需要让其中一个为`false`即可。


2. `r.intent.getComponent() == null`， 会进入此逻辑（设置**component**）。

3. `r.intent.getPackage() == null`，会进入此逻辑（设置**component**）。

4. `r.intent.getFlags() & Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND) == 0` 不能带有`FLAG_RECEIVER_INCLUDE_BACKGROUND`这个标志位，否则会进入此逻辑（设置**FLAG_RECEIVER_INCLUDE_BACKGROUND**）。
5. 如果启动广播的时候携带了权限，那么如果不是签名权限会进入此逻辑(设置**签名权限**)。

其实`1`和`2`我们上面已经测试过了（第`5`个和第`6`个case）；

`3`设置`Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND`，但改常量是`hide`无法通过`Intent`访问。我们只能写`0x01000000`，但不建议这么做；

`4`其实就是文档中说明的**签名权限**不受`Android 8.0`后台执行优化的控制；

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！