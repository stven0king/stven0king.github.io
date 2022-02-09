---
title: Dart和Java通信源码分析和实践
date: 2019-08-01 20:55:00
tags: [ChannelPlugin]
categories: Flutter
description: "在Flutter与Native混合开发的模式下，Platform Channel的应用场景非常多，理解Platform Channel的工作原理，有助于我们在从事这方面开发时能做到得心应手。文章内容是一个获取Android文件目录的Channel，从实践到源码执行的分析过程。"
---


# 前言

`Dart` 和 `Java` 通信这块的知识点涵盖了 `Dart&C` 以及 `Java&C` 的通信，我们先有简单的业务组件的定义再到底层实现原理进行分析，我们现在从Flutter定义的三种 `Channel` 中的 `MethodChannel ` 使用进行剖析。

<center>![面无表情的我.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xMzE5ODc5LWZmMzEzMDNjNTE2MWZjYWQucG5n)</center>


# Dart和Java通信的实践

## Java端ChannelPlugin的创建

```java
public class FileProviderPlugin implements MethodChannel.MethodCallHandler {
    public final static String NAME = "plugins.flutter.io/file_plugin";
    private final Registrar mRegistrar;
    private Context application;
    public FileProviderPlugin(Registrar mRegistrar, Context context) {
        this.mRegistrar = mRegistrar;
        this.application = context.getApplicationContext();
    }

    public static void registerWith(Registrar registrar, Context context) {
        MethodChannel channel =
                new MethodChannel(registrar.messenger(), NAME);
        FileProviderPlugin instance = new FileProviderPlugin(registrar, context);
        channel.setMethodCallHandler(instance);
    }

    @TargetApi(Build.VERSION_CODES.FROYO)
    @Override
    public void onMethodCall(MethodCall methodCall, Result result) {
        String path;
        switch (methodCall.method) {
            case "getExternalStoragePublicDirectory":
                path = Environment.getExternalStoragePublicDirectory(methodCall.arguments.toString()).getAbsolutePath();
                result.success(path);
                break;
        }
    }
}
```

## Java端ChannelPlugin的注册

```java
public class MainActivity extends FlutterActivity {
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    GeneratedPluginRegistrant.registerWith(this);
    FileProviderPlugin.registerWith(this.registrarFor(FileProviderPlugin.NAME), this);
  }
}
```

## Dart端ChannelPlugin的创建

```dart
import 'dart:async';
import 'dart:io';
import 'package:flutter/services.dart';
const MethodChannel _channel = const MethodChannel('plugins.flutter.io/file_plugin');
const String DIRECTORY_DCIM = "DCIM";
Future<Directory> getExternalStoragePublicDirectory(String type) async {
  if (Platform.isIOS)
    throw new UnsupportedError("Functionality not available on iOS");
  final String path = await _channel.invokeMethod('getExternalStoragePublicDirectory', type);
  if (path == null) {
    return null;
  }
  return new Directory(path);
}
```

## Dart端ChannelPlugin的使用

```dart
class Page extends StatelessWidget{
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Reading and Writing Files')),
      body: Center(
        child: Text("this page name is test"),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _click,
        tooltip: 'Increment',
        child: Icon(Icons.add),
      ),
    );
  }

  void _click() {
    _getExternalStoragePublicDirectory(FileUtils.DIRECTORY_DCIM).then((path) {
      print(path);
    });
  }
	//获取存储图片和视频文件目录
  Future<String> _getExternalStoragePublicDirectory(String s) async {
    final directory = await FileUtils.getExternalStoragePublicDirectory(s);
    return directory.path;
  }
}
```

## 结果输出

```java
I/flutter (16780): /storage/emulated/0/DCIM
```



# 源代码分析

## Dart端代码

### MethodChannel信息的封装

```dart
class MethodChannel {
  const MethodChannel(this.name, [this.codec = const StandardMethodCodec()]);
  final String name;
  final MethodCodec codec;
  @optionalTypeArgs
  Future<T> invokeMethod<T>(String method, [dynamic arguments]) async {
    assert(method != null);
    final ByteData result = await BinaryMessages.send(
      name,
      //构造一个MethodCall对象，并调用StandardMethodCodec的encode将其写入一个ByteData中。
      codec.encodeMethodCall(MethodCall(method, arguments)),
    );
    if (result == null) {
      throw MissingPluginException('No implementation found for method $method on channel $name');
    }
    final T typedResult = codec.decodeEnvelope(result);
    return typedResult;
  }
  /******部分代码省略******/
}
//MethodCall对象
class MethodCall {
  const MethodCall(this.method, [this.arguments]): assert(method != null);
  final String method;
  final dynamic arguments;
  @override
  String toString() => '$runtimeType($method, $arguments)';
}
//默认的对函数进行编解码工具
class StandardMethodCodec implements MethodCodec {
  const StandardMethodCodec([this.messageCodec = const StandardMessageCodec()]);
  final StandardMessageCodec messageCodec;
  @override
  ByteData encodeMethodCall(MethodCall call) {
    final WriteBuffer buffer = WriteBuffer();
    messageCodec.writeValue(buffer, call.method);
    messageCodec.writeValue(buffer, call.arguments);
    return buffer.done();
  }
  /******部分代码省略******/
}
//默认的对数据信息编解码工具
class StandardMessageCodec implements MessageCodec<dynamic> {
  const StandardMessageCodec();
  static const int _valueNull = 0;
  static const int _valueTrue = 1;
  static const int _valueFalse = 2;
  static const int _valueInt32 = 3;
  static const int _valueInt64 = 4;
  static const int _valueLargeInt = 5;
  static const int _valueFloat64 = 6;
  static const int _valueString = 7;
  static const int _valueUint8List = 8;
  static const int _valueInt32List = 9;
  static const int _valueInt64List = 10;
  static const int _valueFloat64List = 11;
  static const int _valueList = 12;
  static const int _valueMap = 13;
  //数据转化
  void writeValue(WriteBuffer buffer, dynamic value) {
    if (value == null) {
      buffer.putUint8(_valueNull);
    } else if (value is bool) {
      buffer.putUint8(value ? _valueTrue : _valueFalse);
    }/******部分代码省略******/
}
```

### 信息传到C层

```dart
class BinaryMessages {
  BinaryMessages._();
  static final Map<String, _MessageHandler> _handlers =
      <String, _MessageHandler>{};
  static final Map<String, _MessageHandler> _mockHandlers =
      <String, _MessageHandler>{};
  static Future<ByteData> send(String channel, ByteData message) {
    final _MessageHandler handler = _mockHandlers[channel];
    if (handler != null)
      return handler(message);
    return _sendPlatformMessage(channel, message);
  }

  static Future<ByteData> _sendPlatformMessage(String channel, ByteData message) {
    final Completer<ByteData> completer = Completer<ByteData>();
    ui.window.sendPlatformMessage(channel, message, (ByteData reply) {
      try {
        completer.complete(reply);
      } catch (exception, stack) {
        FlutterError.reportError(FlutterErrorDetails(
          exception: exception,
          stack: stack,
          library: 'services library',
          context: 'during a platform message response callback',
        ));
      }
    });
    return completer.future;
  }
  /******部分代码省略******/
}

class Window {
  Window._();
  /******部分代码省略******/
  void sendPlatformMessage(String name,
                           ByteData data,
                           PlatformMessageResponseCallback callback) {
    final String error =
        _sendPlatformMessage(name, _zonedPlatformMessageResponseCallback(callback), data);
    if (error != null)
      throw new Exception(error);
  }
  String _sendPlatformMessage(String name,
                              PlatformMessageResponseCallback callback,
                              ByteData data) native 'Window_sendPlatformMessage';
  /******部分代码省略******/
}
```

## C层代码分析

###  Window_sendPlatformMessage方法

```c++
//engine/lib/ui/window/window.cc
void Window::RegisterNatives(tonic::DartLibraryNatives *natives) { 
    natives->Register({
        {"Window_defaultRouteName", DefaultRouteName, 1, true},
        {"Window_scheduleFrame", ScheduleFrame, 1, true},
        {"Window_sendPlatformMessage", _SendPlatformMessage, 4, true},
        {"Window_respondToPlatformMessage", _RespondToPlatformMessage, 3, true},
        {"Window_render", Render, 2, true},
        {"Window_updateSemantics", UpdateSemantics, 2, true},
        {"Window_setIsolateDebugName", SetIsolateDebugName, 2, true},
        {"Window_reportUnhandledException", ReportUnhandledException, 2, true},
        {"Window_setNeedsReportTimings", SetNeedsReportTimings, 2, true},
    }); 
}
void _SendPlatformMessage(Dart_NativeArguments args){
    //由DART_NATIVE_CALLBACK_STATIC宏生成的方法调用；
  	//详见：/fuchsia/tonic/dart_binding_macros.h
    tonic::DartCallStatic(&SendPlatformMessage, args);
}
Dart_Handle SendPlatformMessage(Dart_Handle window, const std::string &name, Dart_Handle callback, Dart_Handle data_handle){
    UIDartState *dart_state = UIDartState::Current();
    //判断当前的window对象
    if (!dart_state->window()){
        return tonic::ToDart("Platform messages can only be sent from the main isolate");
    }
    //如果需要返回值构造返回体(PlatformMessageResponseDart),再向dart端做数据返回的时候需要
    fml::RefPtr<PlatformMessageResponse> response;
    if (!Dart_IsNull(callback)){
        response = fml::MakeRefCounted<PlatformMessageResponseDart>(tonic::DartPersistentValue(dart_state, callback), dart_state->GetTaskRunners().GetUITaskRunner());
    }
    //调用window的client对象的HandlePlatformMessage方法处理消息
  	//window=>WindowClient=>RuntimeController(RuntimeDelegate=>Engine).
    //所以最终调用的是Engine.HandlePlatformMessage
    if (Dart_IsNull(data_handle)){
        dart_state->window()->client()->HandlePlatformMessage(fml::MakeRefCounted<PlatformMessage>(name, response));
    }else{
        tonic::DartByteData data(data_handle);
        const uint8_t *buffer = static_cast<const uint8_t *>(data.data());
        dart_state->window()->client()->HandlePlatformMessage(fml::MakeRefCounted<PlatformMessage>(name, std::vector<uint8_t>(buffer, buffer + data.length_in_bytes()), response));
    }
    return Dart_Null();
}
```

### Engine.HandlePlatformMessage

```c++
//engine/shell/common/engine.h
private:Engine::Delegate& delegate_;
//engine/shell/common/engine.cc
static constexpr char kAssetChannel[] = "flutter/assets";
void Engine::HandlePlatformMessage(fml::RefPtr<PlatformMessage> message){
    //判断当前的channel是不是flutter/assets
    if (message->channel() == kAssetChannel){
        HandleAssetPlatformMessage(std::move(message));
    }else{
        delegate_.OnEngineHandlePlatformMessage(std::move(message));
    }
}
```

### Engine对象的delegate_

`Shell` 作为 `Engine::Delegate` 的子类：

```c++
//engine/shell/common/shell.h
class Shell final : public PlatformView::Delegate, 
                    public Animator::Delegate, 
                    public Engine::Delegate, 
                    public Rasterizer::Delegate, 
                    public ServiceProtocol::Handler{

    /*******部分代码省略********/
    // |Engine::Delegate|
    void OnEngineHandlePlatformMessage(fml::RefPtr<PlatformMessage> message) override;
}
```

### OnEngineHandlePlatformMessage方法实现

```c++
// sengine/hell/common/shell.cc
constexpr char kSkiaChannel[] = "flutter/skia";
// |Engine::Delegate|
void Shell::OnEngineHandlePlatformMessage(
    fml::RefPtr<PlatformMessage> message){
    //判断Shell是不是进行过setup
    FML_DCHECK(is_setup_);
    FML_DCHECK(task_runners_.GetUITaskRunner()->RunsTasksOnCurrentThread());
    //判断当前的channel是不是flutter/skia
    if (message->channel() == kSkiaChannel){
        HandleEngineSkiaMessage(std::move(message));
        return;
    }
    //在主线程中post一个task
    task_runners_.GetPlatformTaskRunner()->PostTask(
        //platform_view_是shell创建时候传过来的，也就是在Flutterview进行attach的时候出现的
        //在这里的PlatformView是PlatformViewAndroid
        [view = platform_view_->GetWeakPtr(), message = std::move(message)]() {
            if (view){
                view->HandlePlatformMessage(std::move(message));
            }
        });
}
```

### PlatformView

```c++
/// engine/shell/platform/android/platform_view_android.cc
// |PlatformView|
void PlatformViewAndroid::HandlePlatformMessage(
    fml::RefPtr<flutter::PlatformMessage> message){
    JNIEnv *env = fml::jni::AttachCurrentThread();
    fml::jni::ScopedJavaLocalRef<jobject> view = java_object_.get(env);
    if (view.is_null())
        return;
    int response_id = 0;
  	//如果需要相应
    if (auto response = message->response()){
        response_id = next_response_id_++;
      	//将当前的response_id存储在pending_responses_
        pending_responses_[response_id] = response;
    }
    auto java_channel = fml::jni::StringToJavaString(env, message->channel());
    //带参数
    if (message->hasData()){
        fml::jni::ScopedJavaLocalRef<jbyteArray> message_array(
            env, env->NewByteArray(message->data().size()));
        env->SetByteArrayRegion(
            message_array.obj(), 0, message->data().size(),
            reinterpret_cast<const jbyte *>(message->data().data()));
        message = nullptr;
        // This call can re-enter in InvokePlatformMessageXxxResponseCallback.
        FlutterViewHandlePlatformMessage(env, view.obj(), java_channel.obj(),
                                         message_array.obj(), response_id);
    } else{ // 不带参数
        message = nullptr;
        // This call can re-enter in InvokePlatformMessageXxxResponseCallback.
        FlutterViewHandlePlatformMessage(env, view.obj(), java_channel.obj(), nullptr, response_id);
    }
}
```

### JNI_OnLoad

在 `Android` 程序中  `so` 的加载都会调用 `so` 中的 `JNI_OnLoad` 方法， 详细的知识点可以从 [从JNI_OnLoad看so的加载](https://www.jianshu.com/p/4c0f72233f65) 这篇文章中学习。

```c++
//engine/shell/platform/android/library_loader.cc
JNIEXPORT jint JNI_OnLoad(JavaVM *vm, void *reserved){
    // Initialize the Java VM.
    fml::jni::InitJavaVM(vm);
    JNIEnv *env = fml::jni::AttachCurrentThread();
    bool result = false;
    // Register FlutterMain.
    result = flutter::FlutterMain::Register(env);
    FML_CHECK(result);
    // Register PlatformView
    result = flutter::PlatformViewAndroid::Register(env);
    FML_CHECK(result);
    // Register VSyncWaiter.
    result = flutter::VsyncWaiterAndroid::Register(env);
    FML_CHECK(result);
    return JNI_VERSION_1_4;
}
```

#### FlutterMain::Register

```c++
// engine/shell/platform/android/platform_view_android_jni.cc
void FlutterViewHandlePlatformMessage(JNIEnv *env,
                                      jobject obj,
                                      jstring channel,
                                      jobject message,
                                      jint responseId){
    //即调用FlutterJNI.handlePlatformMessage方法
    env->CallVoidMethod(obj, g_handle_platform_message_method, channel, message,
                        responseId);
    FML_CHECK(CheckException(env));
}

bool RegisterApi(JNIEnv *env){
    /*******部分代码省略********/
    //这里的g_handle_platform_message_method为io/flutter/embedding/engine/FlutterJNI
    g_handle_platform_message_method =
        env->GetMethodID(g_flutter_jni_class->obj(), "handlePlatformMessage",
                         "(Ljava/lang/String;[BI)V");
    return true;
}
```

#### PlatformViewAndroid::Register

```c++
//engine/shell/platform/android/flutter_main.cc
static std::unique_ptr<FlutterMain> g_flutter_main;
void FlutterMain::Init(JNIEnv *env,
                       jclass clazz,
                       jobject context,
                       jobjectArray jargs,
                       jstring kernelPath,
                       jstring appStoragePath,
                       jstring engineCachesPath){
    /*******部分代码省略********/
    g_flutter_main.reset(new FlutterMain(std::move(settings)));
    g_flutter_main->SetupObservatoryUriCallback(env);
}

void FlutterMain::SetupObservatoryUriCallback(JNIEnv *env){
    /*******部分代码省略********/
    g_flutter_jni_class = new fml::jni::ScopedJavaGlobalRef<jclass>(
        env, env->FindClass("io/flutter/embedding/engine/FlutterJNI"));
}
```



## Java端代码

从上面我们看到了最终调用到了 `FlutterJNI.handlePlatformMessage` 方法：

```c++
//engine/shell/platform/android/io/flutter/embedding/engine/FlutterJNI.java
// Called by native.
// TODO(mattcarroll): determine if message is nonull or nullable
@SuppressWarnings("unused") 
private void handlePlatformMessage(@NonNull final String channel, byte[] message, final int replyId){
    if (platformMessageHandler != null){
        platformMessageHandler.handleMessageFromDart(channel, message, replyId);
    }
}

@UiThread 
public void setPlatformMessageHandler(@Nullable PlatformMessageHandler platformMessageHandler){
    ensureRunningOnMainThread();
    this.platformMessageHandler = platformMessageHandler;
}
```

我们现在找到 `FlutterJNI` 中的 `platformMessageHandler` 实例就行。

### FlutterActivity中FlutterView的创建

```java
public class FlutterActivity extends Activity implements Provider, PluginRegistry, ViewFactory {
    private static final String TAG = "FlutterActivity";
    private final FlutterActivityDelegate delegate = new FlutterActivityDelegate(this, this);
    private final FlutterActivityEvents eventDelegate;
    private final Provider viewProvider;
    private final PluginRegistry pluginRegistry;

    public FlutterActivity() {
        this.eventDelegate = this.delegate;
        this.viewProvider = this.delegate;
        this.pluginRegistry = this.delegate;
    }
    /*******部分代码省略********/
}
```

`FlutterActivityDelegate` 中的委托实现：

```java
public final class FlutterActivityDelegate implements FlutterActivityEvents, Provider, PluginRegistry {
    /*******部分代码省略********/
    public void onCreate(Bundle savedInstanceState) {
        /*******部分代码省略********/
        String[] args = getArgsFromIntent(this.activity.getIntent());
        FlutterMain.ensureInitializationComplete(this.activity.getApplicationContext(), args);
        this.flutterView = this.viewFactory.createFlutterView(this.activity);
        if (this.flutterView == null) {
            FlutterNativeView nativeView = this.viewFactory.createFlutterNativeView();
            this.flutterView = new FlutterView(this.activity, (AttributeSet)null, nativeView);
            this.flutterView.setLayoutParams(matchParent);
            this.activity.setContentView(this.flutterView);
            this.launchView = this.createLaunchView();
            if (this.launchView != null) {
                this.addLaunchView();
            }
        }
        /*******部分代码省略********/
    }
    /*******部分代码省略********/
}
```

### FlutterNativeView中platformMessageHandler的注册

在 `FlutterNativeView` 构造函数中我们直接看到 了往 `FlutterJNI` 中注册的是 `FlutterNativeView.PlatformMessageHandlerImpl`。

```java
public class FlutterNativeView implements BinaryMessenger {
    /*******部分代码省略********/
    public FlutterNativeView(Context context, boolean isBackgroundView) {
        this.mNextReplyId = 1;
        this.mPendingReplies = new HashMap();
        this.mContext = context;
        this.mPluginRegistry = new FlutterPluginRegistry(this, context);
        this.mFlutterJNI = new FlutterJNI();
        this.mFlutterJNI.setRenderSurface(new FlutterNativeView.RenderSurfaceImpl());
        this.mFlutterJNI.setPlatformMessageHandler(new FlutterNativeView.PlatformMessageHandlerImpl());
        this.mFlutterJNI.addEngineLifecycleListener(new FlutterNativeView.EngineLifecycleListenerImpl());
        this.attach(this, isBackgroundView);
        this.assertAttached();
        this.mMessageHandlers = new HashMap();
    }
    /*******部分代码省略********/
}
```

### FlutterNativeView.PlatformMessageHandlerImpl

```java
private final class PlatformMessageHandlerImpl implements PlatformMessageHandler {
    /*******部分代码省略********/
    public void handleMessageFromDart(final String channel, byte[] message, final int replyId) {
        FlutterNativeView.this.assertAttached();
        //通过channel获取注册的BinaryMessageHandler
        BinaryMessageHandler handler = (BinaryMessageHandler)FlutterNativeView.this.mMessageHandlers.get(channel);
        if (handler != null) {
            try {
                ByteBuffer buffer = message == null ? null : ByteBuffer.wrap(message);
                //调用BinaryMessageHandler的onMessage方法
                handler.onMessage(buffer, new BinaryReply() {
                    /*******部分代码省略********/
                });
            } 
            /*******部分代码省略********/
        }
    }
    /*******部分代码省略********/
}
```

上面根据 `channel`  获取对应的 ` BinaryMessagehandler` 实例，那么这个实现是通过什么方式在 `FlutterNativeView` 中的 `mMessageHandlers` 注册的呢？

```java
//FlutterNativeView
public void setMessageHandler(String channel, BinaryMessageHandler handler) {
    if (handler == null) {
      this.mMessageHandlers.remove(channel);
    } else {
      this.mMessageHandlers.put(channel, handler);
    }
}
```

我们发现是在调用 `FlutterNativeView` 的 `setMessageHandler` 方法往 `mMessageHandlers` 添加 `BinaryMessageHandler` 。

### BinaryMessagehandler的注册

我们先回头看一下我们自定的 `FileProviderPlugin` 的实现：

```java
public class FileProviderPlugin implements MethodChannel.MethodCallHandler {
    public final static String NAME = "plugins.flutter.io/file_plugin";
 		/*******部分代码省略********/
    public static void registerWith(Registrar registrar, Context context) {
        MethodChannel channel =
                new MethodChannel(registrar.messenger(), NAME);
        FileProviderPlugin instance = new FileProviderPlugin(registrar, context);
        channel.setMethodCallHandler(instance);
    }
}
//FlutterActivity
public final Registrar registrarFor(String pluginKey) {
  	//pluginRegistry的实例是FlutterActivityDelegate对象
  	return this.pluginRegistry.registrarFor(pluginKey);
}
//FlutterActivityDelegate
public Registrar registrarFor(String pluginKey) {
  	return this.flutterView.getPluginRegistry().registrarFor(pluginKey);
}
//FlutterNativeView
public FlutterPluginRegistry getPluginRegistry() {
  	return this.mPluginRegistry;
}
//FlutterPluginRegistry
public Registrar registrarFor(String pluginKey) {
    if (this.mPluginMap.containsKey(pluginKey)) {
      throw new IllegalStateException("Plugin key " + pluginKey + " is already in use");
    } else {
      this.mPluginMap.put(pluginKey, (Object)null);
      return new FlutterPluginRegistry.FlutterRegistrar(pluginKey);
    }
}
//FlutterPluginRegistry.FlutterRegistrar
private class FlutterRegistrar implements Registrar {
  	public BinaryMessenger messenger() {
            return FlutterPluginRegistry.this.mNativeView;
        }
}
//MethodChannel
public final class MethodChannel {
  	public void setMethodCallHandler(@Nullable MethodChannel.MethodCallHandler handler) {
      	//FlutterNativeView.setMessageHandler
        this.messenger.setMessageHandler(this.name, handler == null ? null : new MethodChannel.IncomingMethodCallHandler(handler));
    }
}
```

### BinaryMessageHandler.onMessage

我们找到了 `BinaryMessageHandler` 的实例对象为 `MethodChannel.IncomingMethodCallHandler` :

```java
private final class IncomingMethodCallHandler implements BinaryMessageHandler{
  	//这里的Handler为我们定义的FileProviderPlugin
    private final MethodChannel.MethodCallHandler handler;
    IncomingMethodCallHandler(MethodChannel.MethodCallHandler handler) {
      	this.handler = handler;
    }
    public void onMessage(ByteBuffer message, final BinaryReply reply) {
	    MethodCall call = MethodChannel.this.codec.decodeMethodCall(message);
  	    try {
          //FileProviderPlugin的onMethodCall方法
    	    this.handler.onMethodCall(call, new MethodChannel.Result() {
        	  	public void success(Object result) {
                    reply.reply(MethodChannel.this.codec.encodeSuccessEnvelope(result));
        		}
			    public void error(String errorCode, String errorMessage, Object errorDetails) {
                    reply.reply(MethodChannel.this.codec.encodeErrorEnvelope(errorCode, errorMessage, errorDetails));
          		}
                public void notImplemented() {
                    reply.reply((ByteBuffer)null);
                }
              });
        } catch (RuntimeException var5) {
            Log.e("MethodChannel#" + MethodChannel.this.name, "Failed to handle method call", var5);
            reply.reply(MethodChannel.this.codec.encodeErrorEnvelope("error", var5.getMessage(), (Object)null));
        }
    }
}
```

## 消息回传到Dart端

我们 `Java` 在进行事件响应后执行 `MethodChannel.Result` 的 `success` 或者 `error` 方法将结果传递给 `Dart` 。

```java
//FileProviderPlugin的onMethodCall方法
this.handler.onMethodCall(call, new MethodChannel.Result() {
	  public void success(Object result) {
  	  reply.reply(MethodChannel.this.codec.encodeSuccessEnvelope(result));
  	}
  	public void error(String errorCode, String errorMessage, Object errorDetails) {
    	reply.reply(MethodChannel.this.codec.encodeErrorEnvelope(errorCode, errorMessage, errorDetails));
  	}
  	public void notImplemented() {
    	reply.reply((ByteBuffer)null);
  	}
});
```

上面代码中我们看到最终调用的是 `BinaryReply` 的 `reply` 方法：

而 `BinaryReply` 是我们在消息传递过程中 `FlutterNativeView.PlatformMessageHandlerImpl` 调用 `handleMessageFromDart` 方法中产生的实例：

```java
public void handleMessageFromDart(final String channel, byte[] message, final int replyId) {
    FlutterNativeView.this.assertAttached();
    BinaryMessageHandler handler = (BinaryMessageHandler)FlutterNativeView.this.mMessageHandlers.get(channel);
    if (handler != null) {
        try {
            ByteBuffer buffer = message == null ? null : ByteBuffer.wrap(message);
            handler.onMessage(buffer, new BinaryReply() {
                private final AtomicBoolean done = new AtomicBoolean(false);

                public void reply(ByteBuffer reply) {
                    if (!FlutterNativeView.this.isAttached()) {
                        Log.d("FlutterNativeView", "handleMessageFromDart replying ot a detached view, channel=" + channel);
                    } else if (this.done.getAndSet(true)) {
                        throw new IllegalStateException("Reply already submitted");
                    } else {
                        if (reply == null) {
                            FlutterNativeView.this.mFlutterJNI.invokePlatformMessageEmptyResponseCallback(replyId);
                        } else {
                            FlutterNativeView.this.mFlutterJNI.invokePlatformMessageResponseCallback(replyId, reply, reply.position());
                        }
                    }
                }
            });
        }
    /*******部分代码省略********/
}
```

可以看到当 `replay` 不为空的时候我们调用的是 `FlutterNativeView.this.mFlutterJNI.invokePlatformMessageResponseCallback` 。

```java
 @UiThread
public void invokePlatformMessageResponseCallback(int responseId) {
  	this.ensureAttachedToNative();
  	this.nativeInvokePlatformMessageResponseCallback(this.nativePlatformViewId, responseId);
}
```

从而调用 `native` 的 `nativeInvokePlatformMessageEmptyResponseCallback`  ，这个方法在 `flutter` 的 `so` 加载的时候已经被注册了。

```c++
//engine/shell/platform/android/platform_view_android_jni.cc
{
  .name = "nativeInvokePlatformMessageResponseCallback",
  .signature = "(JI)V",
  .fnPtr = reinterpret_cast<void*>(
    &InvokePlatformMessageResponseCallback),
}
```

对应的 `C++` 的方法是 `InvokePlatformMessageEmptyResponseCallback` 。

```c++
//engine/shell/platform/android/platform_view_android_jni.cc
static void InvokePlatformMessageResponseCallback(JNIEnv* env,
                                                  jobject jcaller,
                                                  jlong shell_holder,
                                                  jint responseId,
                                                  jobject message,
                                                  jint position) {
  ANDROID_SHELL_HOLDER->GetPlatformView()
      ->InvokePlatformMessageResponseCallback(env,         //
                                              responseId,  //
                                              message,     //
                                              position     //
      );
}
```

这个方法有调用 `PlatfromView ` 的 `InvokePlatformMessageResponseCallback` 方法，也就是:

```c++
//engine/shell/platform/android/platform_view_android.cc
void PlatformViewAndroid::InvokePlatformMessageResponseCallback(
    JNIEnv* env,
    jint response_id,
    jobject java_response_data,
    jint java_response_position) {
  if (!response_id)
    return;
  //找到存储在pending_responses_当中的PlatformMessageResponse
  auto it = pending_responses_.find(response_id);
  if (it == pending_responses_.end())
    return;
  uint8_t* response_data =
      static_cast<uint8_t*>(env->GetDirectBufferAddress(java_response_data));
  std::vector<uint8_t> response = std::vector<uint8_t>(
      response_data, response_data + java_response_position);
  auto message_response = std::move(it->second);
  //删除pending_responses_当中的PlatformMessageResponse
  pending_responses_.erase(it);
  //调用PlatformMessageResponse的Complete方法
  message_response->Complete(
      std::make_unique<fml::DataMapping>(std::move(response)));
}
```

上面降到过我们在调用 `Engine.HandlePlatformMessage`  方法时构造的是 `PlatformMessageResponse` 对象是的 `PlatformMessageResponseDart` 。

```c++
//engine/lib/ui/window/platform_message_response_dart.h
class PlatformMessageResponseDart : public PlatformMessageResponse {
  FML_FRIEND_MAKE_REF_COUNTED(PlatformMessageResponseDart);

 public:
  // Callable on any thread.
  void Complete(std::unique_ptr<fml::Mapping> data) override;
  void CompleteEmpty() override;

 protected:
  explicit PlatformMessageResponseDart(
      tonic::DartPersistentValue callback,
      fml::RefPtr<fml::TaskRunner> ui_task_runner);
  ~PlatformMessageResponseDart() override;

  tonic::DartPersistentValue callback_;
  fml::RefPtr<fml::TaskRunner> ui_task_runner_;
};
//engine/lib/ui/window/platform_message_response_dart.cc
void PlatformMessageResponseDart::Complete(std::unique_ptr<fml::Mapping> data) {
  //判断在构造PlatformMessageResponseDart时的callback是否为空
  if (callback_.is_empty())
    return;
  FML_DCHECK(!is_complete_);
  is_complete_ = true;
  ui_task_runner_->PostTask(fml::MakeCopyable(
      [callback = std::move(callback_), data = std::move(data)]() mutable {
        std::shared_ptr<tonic::DartState> dart_state =
            callback.dart_state().lock();
        if (!dart_state)
          return;
        tonic::DartState::Scope scope(dart_state);
        Dart_Handle byte_buffer = WrapByteData(std::move(data));
        //带着byte_buffer参数调用callback方法，回调到dart端
        tonic::DartInvoke(callback.Release(), {byte_buffer});
      }));
}
```

# 总结

事件由 `dart` 到 `C` 再到 `Java` ，相应由 `Java` 到 `C` 再到 `dart` 的过程可以简单用一下步骤叙述：

> 1、Application启动的时候加载flutter的so文件；
> 2、在加载so的时候注册了一系列的相关平台的函数以及操作类；
> 3、dart调用C层的方法顺便将数据传递给C层；
> 4、C层调用相关平台的注册的类的对应方法，
> 5、对应平台进行数据处理并返回数据；
> 6、事件到达系统底层之后找到事件的相应的句柄进行回调；

在整个源码分析过程不免想了解到系统的更底层，结果引出我也解决不了的问题。哈哈尴尬^_^~!

真的很头痛。。。。

## 问题一：tonic

为什么要弄这样的宏定义？

```c++
//tonic/dart_class_library.h
#define DART_NATIVE_CALLBACK_STATIC(CLASS, METHOD)          \
static void CLASS##_##METHOD(Dart_NativeArguments args) { \
    tonic::DartCallStatic(&CLASS::METHOD, args);            \
}
//tonic/dart_args.h
template <typename Sig>
void DartCallStatic(Sig func, Dart_NativeArguments args) {
  DartArgIterator it(args, 0);
  using Indices = typename IndicesForSignature<Sig>::type;
  DartDispatcher<Indices, Sig> decoder(&it);
  if (it.had_exception())
    return;
  decoder.Dispatch(func);
}
```

## 问题二：Dart_handle&DartInvoke

`DartInvoke` 调用 `Dart_Handle` 方法就能执行 `dart` 的函数？`Dart_handle` 到底在 `C` 这一层是一个什么样的结构体，它的作用有什么？ 

```dart
//tonic/logging/dart_invoke.cc
Dart_Handle DartInvoke(Dart_Handle closure,
                       std::initializer_list<Dart_Handle> args) {
  int argc = args.size();
  Dart_Handle* argv = const_cast<Dart_Handle*>(args.begin());
  Dart_Handle handle = Dart_InvokeClosure(closure, argc, argv);
  LogIfError(handle);
  return handle;
}
//sdk/runtime/include/dart_api.h
DART_EXPORT DART_WARN_UNUSED_RESULT Dart_Handle
Dart_InvokeClosure(Dart_Handle closure,
                   int number_of_arguments,
                   Dart_Handle* arguments);

//dart/sdk/runtime/vm/dart_api_impl.cc
DART_EXPORT Dart_Handle Dart_InvokeClosure(Dart_Handle closure,
                                           int number_of_arguments,
                                           Dart_Handle* arguments) {
  DARTSCOPE(Thread::Current());
  API_TIMELINE_DURATION(T);
  CHECK_CALLBACK_STATE(T);
  const Instance& closure_obj = Api::UnwrapInstanceHandle(Z, closure);
  if (closure_obj.IsNull() || !closure_obj.IsCallable(NULL)) {
    RETURN_TYPE_ERROR(Z, closure, Instance);
  }
  if (number_of_arguments < 0) {
    return Api::NewError(
        "%s expects argument 'number_of_arguments' to be non-negative.",
        CURRENT_FUNC);
  }

  // Set up arguments to include the closure as the first argument.
  const Array& args = Array::Handle(Z, Array::New(number_of_arguments + 1));
  Object& obj = Object::Handle(Z);
  args.SetAt(0, closure_obj);
  for (int i = 0; i < number_of_arguments; i++) {
    obj = Api::UnwrapHandle(arguments[i]);
    if (!obj.IsNull() && !obj.IsInstance()) {
      RETURN_TYPE_ERROR(Z, arguments[i], Instance);
    }
    args.SetAt(i + 1, obj);
  }
  // Now try to invoke the closure.
  return Api::NewHandle(T, DartEntry::InvokeClosure(args));
}
```



文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！