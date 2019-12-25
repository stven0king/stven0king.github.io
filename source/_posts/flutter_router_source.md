---
title: Flutter路由管理和页面参数的传递（源码分析）
date: 2019-07-20 10:14:00
tags: [flutter传参,路由管理,源码分析]
categories: Flutter
description: "上一篇 Flutter路由管理和页面参数的传递（获取&返回）文章中我们讲述了这么用代码实现Flutter中页面参数的传递，这一篇我们用源码分析一下Navigator为什么可以进行页面参数传递。。
"
---


# 前言

上一篇 [Flutter路由管理和页面参数的传递（获取&返回）](http://dandanlove.com/2019/07/20/flutter_router_fix_param/) 文章中我们讲述了这么用代码实现 `Flutter` 中页面参数的传递，这一篇我们用源码分析一下 `Navigator` 为什么可以进行页面参数传递。



从页面跳转入口的代码进行分析：

`Navigator.of(context).pushNamed('/route1');`

 # Navigator 的获取


`Navigator` 对应的 `State` 是 `NavigatorState` ，所以实际上我们需要获取的是 `NavigatorState` 。

```dart
class Navigator extends StatefulWidget {
  /******部分代码省略*****/
  static NavigatorState of(
    BuildContext context, {
      bool rootNavigator = false,
      bool nullOk = false,
    }) {
    final NavigatorState navigator = rootNavigator
      ? context.rootAncestorStateOfType(const TypeMatcher<NavigatorState>())
      : context.ancestorStateOfType(const TypeMatcher<NavigatorState>());
    assert(() {
      if (navigator == null && !nullOk) {
        throw FlutterError(
          'Navigator operation requested with a context that does not include a Navigator.\n'
          'The context used to push or pop routes from the Navigator must be that of a '
          'widget that is a descendant of a Navigator widget.'
        );
      }
      return true;
    }());
    return navigator;
  }
}
```

我们从源看到 `NavigatorState` 的获取实际是获取的 `context.ancestorStateOfType` 。

```dart
abstract class Element extends DiagnosticableTree implements BuildContext {
  /******部分代码省略*****/
  @override
  State ancestorStateOfType(TypeMatcher matcher) {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    Element ancestor = _parent;
    while (ancestor != null) {
      //从当前的Element节点一直向上寻找到匹配的StatefulElement
      if (ancestor is StatefulElement && matcher.check(ancestor.state))
        break;
      ancestor = ancestor._parent;
    }
    final StatefulElement statefulAncestor = ancestor;
    //返回匹配的StatefulElement的state
    return statefulAncestor?.state;
  }
}
```

循环遍历向上寻找 `Navigato`r 的 `state` ，这里就是 `NavigatorState` 。

 # Navigator的生成

`Navigator` 的 `Widget` 是是什么时候添加到视图树中的呢？我们从 `Flutter` 应用程序的入口开始一步一步跟进代码的执行：

```dart
void main() => runApp(MyApp());
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(...);
  }
}
```

`MaterialApp` 传入 `routes` 和 `onGenerateRoute` 等参数，`MaterialApp` 的 `state` 是 `_MaterialAppState`  它构建的是 `WidgetsApp`  类型的 `Widget` ，同时 `routes` 和 `onGenerateRoute` 等参数也进行了透传。

```dart
class MaterialApp extends StatefulWidget {
  const MaterialApp({
    Key key,
    this.navigatorKey,
    this.home,
    this.routes = const <String, WidgetBuilder>{},
    this.initialRoute,
    this.onGenerateRoute,
    this.onUnknownRoute,
    /******部分代码省略*****/
  })
    /******部分代码省略*****/
    @override
    _MaterialAppState createState() => _MaterialAppState();
}
class _MaterialAppState extends State<MaterialApp> {
  /******部分代码省略*****/
  @override
  Widget build(BuildContext context) {
    Widget result = WidgetsApp(
      key: GlobalObjectKey(this),
      navigatorKey: widget.navigatorKey,
      navigatorObservers: _navigatorObservers,
      pageRouteBuilder: <T>(RouteSettings settings, WidgetBuilder builder) =>
      MaterialPageRoute<T>(settings: settings, builder: builder),
      home: widget.home,
      routes: widget.routes,
      initialRoute: widget.initialRoute,
      onGenerateRoute: widget.onGenerateRoute,
      onUnknownRoute: widget.onUnknownRoute,
      /******部分代码省略*****/
    );
  }
```

我们再看看 `WidgetsApp` 对应的 `State` 的 `_WidgetsAppState` 。在` _WidgetsAppState` 的 `Widget build(BuildContext context) ` 方法中我们找到了管理路由的 `Navigator` 的构造时机。

```dart
class WidgetsApp extends StatefulWidget {
  WidgetsApp({ // can't be const because the asserts use methods on Iterable :-(
    Key key,
    this.navigatorKey,
    this.onGenerateRoute,
    this.onUnknownRoute,
    this.navigatorObservers = const <NavigatorObserver>[],
    this.initialRoute,
    this.pageRouteBuilder,
    this.home,
    this.routes = const <String, WidgetBuilder>{},
    /******部分代码省略*****/
    );
    @override
    _WidgetsAppState createState() => _WidgetsAppState();
  }
class _WidgetsAppState extends State<WidgetsApp> implements WidgetsBindingObserver {
  @override
  Widget build(BuildContext context) {
    Widget navigator;
    if (_navigator != null) {
      navigator = Navigator(
        key: _navigator,
        // If window.defaultRouteName isn't '/', we should assume it was set
        // intentionally via `setInitialRoute`, and should override whatever
        // is in [widget.initialRoute].
        initialRoute: WidgetsBinding.instance.window.defaultRouteName != 		Navigator.defaultRouteName
            ? WidgetsBinding.instance.window.defaultRouteName
            : widget.initialRoute ?? WidgetsBinding.instance.window.defaultRouteName,
        onGenerateRoute: _onGenerateRoute,
        onUnknownRoute: _onUnknownRoute,
        observers: widget.navigatorObservers,
      );
    }
    Widget result;
    if (widget.builder != null) {
      result = Builder(
        builder: (BuildContext context) {
          return widget.builder(context, navigator);
        },
      );
    } else {
      assert(navigator != null);
      result = navigator;
    }
    /******部分代码省略*****/
    /**上面经过多次的操作之后，navigator变为result的某个子孙节点上的child**/
    Widget title;
    if (widget.onGenerateTitle != null) {
      title = Builder(
        // This Builder exists to provide a context below the Localizations widget.
        // The onGenerateTitle callback can refer to Localizations via its context
        // parameter.
        builder: (BuildContext context) {
          final String title = widget.onGenerateTitle(context);
          assert(title != null, 'onGenerateTitle must return a non-null String');
          return Title(
            title: title,
            color: widget.color,
            child: result,
          );
        },
      );
    } else {
      title = Title(
        title: widget.title,
        color: widget.color,
        child: result,
      );
    }
    /******部分代码省略*****/
    /**上面经过多次的操作之后，result变为title的某个子孙节点上的child**/
    return MediaQuery(
      data: MediaQueryData.fromWindow(WidgetsBinding.instance.window),
      child: Localizations(
        locale: appLocale,
        delegates: _localizationsDelegates.toList(),
        //将title作为child视图，也就是说navigator变为其中的某个子孙节点视图
        child: title,
      ),
    );
  }
}
```

在构建的 MediaQuery 就存在我们需要的 `Navigator` 。
![Navigator.png](https://upload-images.jianshu.io/upload_images/1319879-79075fdee2f218d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这张图是程序运行时候使用（[DevTools](https://flutter.dev/docs/testing/debugging)）进行的页面元素分析，也证明了 `Navigator` 是在页面的 `Widget` 元素路径上的。

# pushNamed方法解析

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
Route<T> _routeNamed<T>(String name, { @required Object arguments, bool allowNull = false }) {
  assert(!_debugLocked);
  assert(name != null);
  final RouteSettings settings = RouteSettings(
    name: name,
    isInitialRoute: _history.isEmpty,
    arguments: arguments,
  );
  Route<T> route = widget.onGenerateRoute(settings);
  if (route == null && !allowNull) {
    assert(() {
      if (widget.onUnknownRoute == null) {
        throw FlutterError(...);
      }
      return true;
    }());
    route = widget.onUnknownRoute(settings);
    assert(() {
      if (route == null) {
        throw FlutterError(...);
      }
      return true;
    }());
  }
  return route;
}
```

我们看到是调用了 `widget.onGenerateRoute(settings)` 生成路由， 这里的 `onGenerateRoute` 在 `Navigator` 在构造的时候传入的 `onGenerateRoute` 。

# onGenerateRoute

`Navigator` 在构造的时候如果我们细心就会发现 `onGenerateRoute` 现在改为了 `_onGenerateRoute` 。

也就是 `_WidgetsAppState` 的 `_onGenerateRoute` 方法实现：

```dart
Route<dynamic> _onGenerateRoute(RouteSettings settings) {
  final String name = settings.name;
  //从widget注册的路由中获取name对应的WidgetBuilder
  final WidgetBuilder pageContentBuilder = name == Navigator.defaultRouteName && widget.home != null
      ? (BuildContext context) => widget.home
      : widget.routes[name];
  //如果pageContentBuilder不为空，那么和RouteSettings一起执行widget.pageRouteBuilder构造一个route
  if (pageContentBuilder != null) {
    assert(widget.pageRouteBuilder != null,
      'The default onGenerateRoute handler for WidgetsApp must have a '
      'pageRouteBuilder set if the home or routes properties are set.');
    final Route<dynamic> route = widget.pageRouteBuilder<dynamic>(
      settings,
      pageContentBuilder,
    );
    assert(route != null,
      'The pageRouteBuilder for WidgetsApp must return a valid non-null Route.');
    return route;
  }
  //如果pageContentBuilder为空，那么执行widget.onGenerateRoute的方法
  if (widget.onGenerateRoute != null)
    return widget.onGenerateRoute(settings);
  return null;
}    
```

`widget.pageRouteBuilder` 的方法，我们在生成 `WidgetsApp`可以看到是：

```dart
pageRouteBuilder: <T>(RouteSettings settings, WidgetBuilder builder) =>
            MaterialPageRoute<T>(settings: settings, builder: builder)
```

所以最终我们通过在 `MaterialApp` 注册 `routes` 生成了一个 `MaterialPageRoute` 用来进行页面跳转。

最后如果 `routes` 为空的话，我们执行 `widget.onGenerateRoute` 。这个解释了在  [Flutter路由管理和页面参数的传递（获取&返回）](http://dandanlove.com/2019/07/20/flutter_router_fix_param/)  这篇文章末尾说的 `onGenerateRoute` 方式进行的参数传递，必须不能进行 `routers` 的注册。



文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：

<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>