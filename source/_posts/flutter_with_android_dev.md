---
title: Flutter混合开发：Android接入Flutter
date: 2019-07-20 10:00:00
tags: [混合开发]
categories: Flutter
description: "虽然Flutter无法接入我们的项目，但是我们可以尝试者去模仿Flutter在项目中的使用场景。下边我讲讲我在Android 和Flutter的混合开发实践的躺坑之旅。
"
---


# 前言

<center>![Flutter.png](https://upload-images.jianshu.io/upload_images/1319879-de2f5f1d8891d703.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)</center>

`Flutter` Google推出已经已经一年多了，单个 `Flutter` 项目的开发流程已经很成熟了。对与个人开发者来说使用 `Flutter` 开发一个跨平台的App挺有意思。但是对于现有的项目改造来说还是不建议，`Flutter` 中的控件还没有完全能满足我们的要求，我们需要解决这个问题会消耗我们大量的研发资源。

虽然 `Flutter` 无法接入我们的项目，但是我们可以尝试者去模仿 `Flutter` 在项目中的使用场景。下边我讲讲我在 `Android` 和 `Flutter` 的混合开发实践的躺坑之旅。

## 官方指导

[Add Flutter to existing apps]([https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps](https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps)

## 实践：

### 创建Flutter模块

如果你存在一个 `Android app` 的路径是 `some/path/MyApp` ，你希望创建你的 `Flutter` 项目作为子模块：

```shell
$ cd some/path/
# flutter create my_flutter是创建纯Flutter项目的命令
$ flutter create -t module my_flutter
```

你能得到一个创建好的 `some/path/my_flutter` 的 `Flutter` 项目，它包含了一部分`Dart` 的代码。其中有一个 `.android/` 的隐藏的子文件夹，它包装了Android库中的模块项目。

### 使主app依赖Flutter模块

在主App的 `setting.gradle` 文件中包含 `Flutter` 模块作为子模块。

```java
// MyApp/settings.gradle
include ':app'                                     // assumed existing content
setBinding(new Binding([gradle: this]))                                 // new
evaluate(new File(                                                      // new
  settingsDir.parentFile,                                               // new
  'my_flutter/.android/include_flutter.groovy'                          // new
))
```

这个绑定和脚本评估允许 `Flutter` 模块可以包含自己（作为`:flutter`），在你自己的 `setting.gradle` 文件中， 任何 `Flutter` 插件可以作为模块使用（作为 `:package_info` , `:video_player` 等）。

在你的app采用 `implementation` 方式依赖 `Flutter` 模块：

```java
// MyApp/app/build.gradle
:
dependencies {
  implementation project(':flutter')
  :
}
```

### 在你的Java代码中使用Flutter模块

使用 `Flutter` 模块的Java接口  `Flutter.createView`  ，可以在你的app中添加 `Flutter View` 。

#### addView
<center>![add.png](https://upload-images.jianshu.io/upload_images/1319879-a22ee322f894f259.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/620)</center>

```java
// MyApp/app/src/main/java/some/package/MainActivity.java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        FrameLayout frameLayout = findViewById(R.id.flutter_root);
        View flutterView = Flutter.createView(MainActivity.this, getLifecycle(), "root1");
        FrameLayout.LayoutParams layout = new FrameLayout.LayoutParams(FrameLayout.LayoutParams.MATCH_PARENT, FrameLayout.LayoutParams.MATCH_PARENT);
        frameLayout.addView(flutterView, layout);
    }
}
```

#### setContentView
<center>![all.png](https://upload-images.jianshu.io/upload_images/1319879-beb144b6387966f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/620)</center>
```java
// MyApp/app/src/main/java/some/package/MainActivity.java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        View flutterView = Flutter.createView(MainActivity.this, getLifecycle(), "root1");
        FrameLayout.LayoutParams layout = new FrameLayout.LayoutParams(FrameLayout.LayoutParams.MATCH_PARENT, FrameLayout.LayoutParams.MATCH_PARENT);
        setContentView(flutterView, layout);
    }
}
```

#### fragment

也可以创建一个负责管理自己生命周期的FlutterFragment。

```java
//io.flutter.facade.Flutter.java
public static FlutterFragment createFragment(String initialRoute) {
    final FlutterFragment fragment = new FlutterFragment();
    final Bundle args = new Bundle();
    args.putString(FlutterFragment.ARG_ROUTE, initialRoute);
    fragment.setArguments(args);
    return fragment;
}
//io.flutter.facade.FlutterFragment.java
public class FlutterFragment extends Fragment {
    public static final String ARG_ROUTE = "route";
    private String mRoute = "/";

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        if (getArguments() != null) {
            mRoute = getArguments().getString(ARG_ROUTE);
        }
    }

    @Override
    public void onInflate(Context context, AttributeSet attrs, Bundle savedInstanceState) {
        super.onInflate(context, attrs, savedInstanceState);
    }

    @Override
    public FlutterView onCreateView(@NonNull LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        return Flutter.createView(getActivity(), getLifecycle(), mRoute);
    }
}
```

### dart代码交互

上面我们使用了 `"route1"` 字符串告诉 `Flutter` 模块中的  `Dart` 代码展示那个 `widget` 。在 `Flutter` 模块项目的模板文件 `lib/main.dart` 中的可以使用`window。defaultRouteName` 作为提供路由选择的字符串，通过 `runApp` 决定创建那个 `widget` 。

```dart
import 'dart:ui';
import 'package:flutter/material.dart';

void main() => runApp(_widgetForRoute(window.defaultRouteName));

Widget _widgetForRoute(String route) {
  switch (route) {
    case 'route1':
      return SomeWidget(...);
    case 'route2':
      return SomeOtherWidget(...);
    default:
      return Center(
        child: Text('Unknown route: $route', textDirection: TextDirection.ltr),
      );
  }
}
```

### 构建和运行你的app

一般在使用 `Android Studio` 中，你可以构建和运行 `Myapp` ，完全和在添加Flutter模块依赖项之前相同。也可以同样的进行代码的编辑、调试和分析。



## 报错和解决

整个接入的过程一般是不会有问题的，但是呢？我们不按照官方提供的文档上自己一顿操作可能会产生其他的问题。

### 关联项目报错

```shell
AILURE: Build failed with an exception.

* Where:
Settings file '/Users/tanzx/AndroidStudioWorkSapce/GitHub/MyApp/settings.gradle' line: 6

* What went wrong:
A problem occurred evaluating settings 'MyApp'.
> /Users/tanzx/AndroidStudioWorkSapce/GitHub/MyApp/my_flutter/.android/include_flutter.groovy (/Users/tanzx/AndroidStudioWorkSapce/GitHub/MyApp/my_flutter/.android/include_flutter.groovy)

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.

* Get more help at https://help.gradle.org

BUILD FAILED in 0s
```

#### 报错原因：

将创建的 `Flutter` 模块放在了 `MyApp` 文件夹的内部，地址搞错。

#### 解决方法：

- 将 `Flutter` 放在 `MyApp` 的外层；
- 将 `setting.gradle` 配置文件中的， `'my_flutter/.android/include_flutter.groovy'` 改为 `'MyApp/my_flutter/.android/include_flutter.groovy'` ；



作为Android开发人员学习 `Flutter` 的第一步我们已经完成了，虽然后续的需要了解和学习的还有很多 `良好的开始是成功的一半` ，加油~！



文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！