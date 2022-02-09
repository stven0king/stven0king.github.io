---
title: Flutter路由管理和页面参数的传递（获取和返回）
date: 2019-07-20 10:11:00
tags: [flutter传参,路由管理]
categories: Flutter
description: "我们做 Android开发的人员都知道 Android应用程序在进行页面跳转的时候可以利用Intent进行参数传递，那么再开发 Flutter 的时候有类似的方式可以进行参数传递么？
"
---


# 前言

在做 `Flutter` 开发的时候所有的页面以及页面上的元素都变成了 `Widget` ，创建一个页面或者视图直接 `new` 一个新的 `widget` 就可以，相关的参数我们可以直接通过构造函数直接传递。

我们做 `Android` 开发的人员都知道 `Android` 应用程序在进行页面跳转的时候可以利用Intent进行参数传递，那么再开发 `Flutter` 的时候有类似的方式可以进行参数传递么？答案当然是有。



[Flutter中文网](https://flutterchina.club/cookbook/navigation/navigation-basics/) 中有一段话，大多数应用程序包含多个页面。例如，我们可能有一个显示产品的页面，然后，用户可以点击产品，跳到该产品的详情页。

在Android中，页面对应的是Activity，在iOS中是ViewController。而在Flutter中，页面只是一个widget！

在Flutter中，我们那么我们可以使用[`Navigator`](https://docs.flutter.io/flutter/widgets/Navigator-class.html)在页面之间跳转。

所以我们下边讲述 `widget` 的参数传递，从简单到简便：

> widget构造参数传递
>
> route参数传递
>
> 上面两种方式进混合（onGenerateRoute）



# widget构造参数传递

```dart
class Page extends StatelessWidget{
  Page({this.arguments});
  final Map arguments;

  @override
  Widget build(BuildContext context) {
    return Material(
      child: Center(
        child: Text("this page name is ${arguments != null ? arguments['name'] : 'null'}"),
      ),
    );
  }
}
```

上面是一个简单的 `Flutter` 的视图组件，我们在使用参数 `arguments` 的时候只需要将其传入到 `Page({this.arguments})` 的构造函数中。

```dart
void main() => runApp(MyApp());
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      home: Page(arguments: {"name": 'Flutter Demo Home Page'}),
    );
  }
}
```

这种方式进行的参数传递只能单向往下一个页面传递，不能像Android的 `setResult` 一样往上一级页面传递数据。

# Route

在讲 `Route` 传参的时候，我们先讲讲 `Flutter` 中 `Route` 相关的知识点。

路由( `Route` )在移动开发中通常指页面（ `Page` ），这跟 `web` 开发中单页应用的 `Route` 概念意义是相同的，`Route` 在` Android` 中通常指一个 `Activity` ，在 `iOS` 中指一个 `ViewController` 。所谓路由管理，就是管理页面之间如何跳转，通常也可被称为导航管理。这和原生开发类似，无论是 `Android` 还是 `iOS` ，导航管理都会维护一个路由栈，路由入栈( `push` )操作对应打开一个新页面，路由出栈( `pop`)操作对应页面关闭操作，而路由管理主要是指如何来管理路由栈。

## MaterialPageRoute

`MaterialPageRoute` 是我们使用最为广泛的路由类，它继承自 `PageRoute` 类， `PageRoute` 类是一个抽象类继承抽象类 `ModalRoute`，下面我们介绍一下 `MaterialPageRoute`  构造函数的各个参数的意义：

```dart
MaterialPageRoute({
  @required this.builder,
  RouteSettings settings,
  this.maintainState = true,
  bool fullscreenDialog = false,
}) : assert(builder != null),
      assert(maintainState != null),
      assert(fullscreenDialog != null),
      assert(opaque),
      super(settings: settings, fullscreenDialog: fullscreenDialog);
```

- `builder` 是一个WidgetBuilder类型的回调函数，它的作用是构建路由页面的具体内容，返回值是一个widget。我们通常要实现此回调，返回新路由的实例。
- `settings` 包含路由的配置信息，如路由名称、路由参数、是否初始路由（首页）。
- `maintainState`：默认情况下，当入栈一个新路由时，原来的路由仍然会被保存在内存中，如果想在路由没用的时候释放其所占用的所有资源，可以设置`maintainState`为false。
- `fullscreenDialog`表示新的路由页面是否是一个全屏的模态对话框，在iOS中，如果`fullscreenDialog`为`true`，新页面将会从屏幕底部滑入（而不是水平方向）。

> 如果想自定义路由切换动画，可以自己继承PageRoute来实现，我们将在后面介绍动画时，实现一个自定义的路由Widget。

## 命名路由

所谓命名路由（Named Route）即给路由起一个名字，然后可以通过路由名字直接打开新的路由。这为路由管理带来了一种直观、简单的方式。和 `Android` 中的 `ARrouter` 页面跳转框架所定义的 `path` 非常的类似。

## 路由表

要想使用命名路由，我们必须先提供并注册一个路由表（routing table），这样应用程序才知道哪个名称与哪个路由Widget对应。路由表的定义是一个 `Map<String, WidgetBuilder>` 结构的 `Map` ， key 为路由的名称，是个字符串；value是个builder回调函数，用于生成相应的路由Widget。我们在通过路由名称入栈新路由时，应用会根据路由名称在路由表中找到对应的WidgetBuilder回调函数，然后调用该回调函数生成路由widget并返回。

我们在创建 `MaterialApp` 的时候就有一个 `routes` 构造参数：

```dart
const MaterialApp({
  Key key,
  this.navigatorKey,
  this.home,
  this.routes = const <String, WidgetBuilder>{},
  this.initialRoute,
  this.onGenerateRoute,
  this.onUnknownRoute,
  this.navigatorObservers = const <NavigatorObserver>[],
  /***********/
})
```

## Navigator

 `Navigator`是一个路由管理的widget，它通过一个栈来管理一个路由widget集合。通常当前屏幕显示的页面就是栈顶的路由。`Navigator`提供了一系列方法来管理路由栈，我们主要使用 `push` 和 `pop` 连个操作进行页面的入栈和出栈。

### push

将给定的路由入栈（即打开新的页面），返回值是一个`Future`对象，用以接收新路由出栈（即关闭）时的返回数据。

`push`  我们主要使用两个方法一个是直接 `push` 一个路由，另外一个是 `pushNamed` 一个命名路由地址（PS：要想使用命名路由必须提供并注册一个路由表，这后面会讲到）。

#### push方法源码

下边是 `Navigator.push` 的源码，入参的 `Route` 对象中有一个 `RouteSettings` 成员变量，我们可以在构造 `Route` 对象的时候将需要传递的参数放在 `RouteSettings` 中。

```dart
@optionalTypeArgs
static Future<T> push<T extends Object>(BuildContext context, Route<T> route) {
  return Navigator.of(context).push(route);
}
```

#### push方法使用

我们可以将参数放在 `SecondScreen` 的构造函数中，也可以放在构造的 `MaterialPageRoute` 的 `RouteSettings` 中。

```dart
Navigator.push(
  context,
  new MaterialPageRoute(builder: (context) => new SecondScreen()),
).then((data){
  //接受返回的参数
  print(data.toString());
};
```

### pushNamed方法源码

第二种方式最终的实现也是调用的 `push` 方法，这中方法直接暴露了参数 `Object arguments` 。

```dart
@optionalTypeArgs
static Future<T> pushNamed<T extends Object>(
  BuildContext context,
  String routeName, {
  Object arguments,
  }) {
  return Navigator.of(context).pushNamed<T>(routeName, arguments: arguments);
}
@optionalTypeArgs
Future<T> pushNamed<T extends Object>(
  String routeName, {
  Object arguments,
}) {
  return push<T>(_routeNamed<T>(routeName, arguments: arguments));
}
```

#### pushNamed方法使用

使用前提是 `/route1` 已经被注册到路由表中：

```dart
Navigator.of(context)
  .pushNamed(
    '/route1',
    arguments: {
      "name": 'hello'
    }
	).then((data){
  	//接受返回的参数
  	print(data.toString());
	};
```

### pop

将栈顶路由出栈，入参为一个 `object` 类型的对象为当前页面关闭时返回给上一个页面的数据。

```dart
@optionalTypeArgs
static bool pop<T extends Object>(BuildContext context, [ T result ]) {
  return Navigator.of(context).pop<T>(result);
}
```

使用非常简单：

```dart
Navigator.of(context).pop("ok~!");
```

### 页面参数的传输、获取以及结果返回

#### 参数传输

```dart
Navigator.of(context).pushNamed('/route1', arguments: {"name": 'hello'});
```

#### 参数获取

```dart
class Page extends StatelessWidget{
  String name;
  @override
  Widget build(BuildContext context) {
    dynamic obj = ModalRoute.of(context).settings.arguments;
    if (obj != null && isNotEmpty(obj["name"])) {
      name = obj["name"];
    }
    return Material(
      child: Center(
        child: Text("this page name is ${name}"),
      ),
    );
  }
}
```

#### 参数返回

```dart
//页面返回参数
Navigator.of(context).pop("ok~!");
//上一个页面接收参数
Navigator.of(context)
  .pushNamed(
    '/route1',
    arguments: {
      "name": 'hello'
    }
	).then((data){
  	//接受返回的参数
  	print(data.toString());
	};
```

#  onGenerateRoute构建路由

在说 `onGenerateRoute` 构建路由之前，我们得先了解他。前面 `MaterialApp` 的的构造函数中我们看到过它出现， `MaterialApp` 有一个参数类型为 `Function` 类型的 `onGenerateRoute` 。

```dart
void main() => runApp(MyApp());
Map<String, WidgetBuilder> routers = {'/route1': (context, {arguments}) => Page(arguments: arguments)}
class MyApp extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      // 处理Named页面跳转 传递参数
      onGenerateRoute: (RouteSettings settings) {
        // 统一处理
        final String name = settings.name;
        final Function pageContentBuilder = routers[name];
        if (pageContentBuilder != null) {
          final Route route =
          MaterialPageRoute(
                builder: (context) {
                  //将RouteSettings中的arguments参数取出来，通过构造函数传入
                  return pageContentBuilder(context, arguments: settings.arguments);
                },
                settings: settings,
            );
          return route;
        }
      },
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: MyHomePage(name: 'Flutter Demo Home Page'),
      //routes优先执行，所以必须注释掉，否则onGenerateRoute方法不会调用
      //routes: routers,
    );
  }
}
class Page extends StatelessWidget{
  Page({this.arguments});
  final Map arguments;
  @override
  Widget build(BuildContext context) {
    return Material(
      child: Center(
        child: Text("this page name is ${arguments != null ? arguments['name'] : 'null'}"),
      ),
    );
  }
}
```

>-  这种方式统一处理了页面的 `arguments` 参数，所以必须保证 `Map<String, WidgetBuilder> routers` 当中注册的所有 `Widget` 的构造函数中都有一个 `Map` 类型并且名为 `arguments` 的参数。
>-  这种方法同时也传递了 `RouteSettings` ，所以在下一个页面我们也可以通过 `ModalRoute.of(context).settings.arguments` 方式获取参数。

>- 这种方式可以自定义 `PageRoute` 的类型，比如自带 `IOS` 侧滑返回效果的 `CupertinoPageRoute` 等。之前通过在  `WidgetsApp` 注册`routes` 的方式默认生成的 `PageRoute` 类型为 `MaterialPageRoute` 。

源码分析传送门：[Flutter路由管理和页面参数的传递（源码分析）](http://dandanlove.com/2019/07/20/flutter_router_source/)



文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！