---
title: Flutter混合开发：启动黑屏处理
date: 2019-07-20 10:07:00
tags: [flutter问题]
categories: Flutter
description: "我们讲到在 Flutter混合开发 中主要有、有addView（页面局部Flutter） 和 setContentView（整个页面Flutter）两种方式。这两种方式在启动页面的时候都会遇到 FlutterView出现黑屏的情况。。
"
---


上一篇 [Flutter混合开发：Android接入Flutter](http://dandanlove.com/2019/07/20/flutter_with_android_dev/) 我们讲到在 `Flutter混合开发` 中主要有、有 `addView` （页面局部Flutter） 和 `setContentView` （整个页面Flutter）两种方式。这两种方式在启动页面的时候都会遇到 `FlutterView` 出现黑屏的情况。

## 解决思路

延迟 `FlutterView` 的加载时间。

## setContentView

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        View flutterView = Flutter.createView(MainActivity.this, getLifecycle(), "root1");
        FrameLayout.LayoutParams layout = new FrameLayout.LayoutParams(FrameLayout.LayoutParams.MATCH_PARENT, FrameLayout.LayoutParams.MATCH_PARENT);
        setContentView(flutterView, layout);
    }
}
```

这中方式目前没有找到一种很好的方式推迟 `FlutterView` 的加载时间。

## addView


```java
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

这种情况我们可以使用多种方式进行 `FlutterView` 加载的延迟。

### 检测FlutterView的第一帧

```java
public class MainActivity extends AppCompatActivity {
  @Override
  protected void onCreate(Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      setContentView(R.layout.activity_main);
      FrameLayout frameLayout = findViewById(R.id.flutter_root);
      frameLayout.setVisibility(View.INVISIBLE);
      FlutterView flutterView = Flutter.createView(MainActivity.this, getLifecycle(), "root");
      FrameLayout.LayoutParams layout = new FrameLayout.LayoutParams(FrameLayout.LayoutParams.MATCH_PARENT, FrameLayout.LayoutParams.MATCH_PARENT);

      FlutterView.FirstFrameListener listeners = () -> frameLayout.setVisibility(View.VISIBLE);
      flutterView.addFirstFrameListener(listeners);
      frameLayout.addView(flutterView, layout);
  }
}
```

### 在View的post方法中延迟执行FlutterView的添加

```java
public class MainActivity extends AppCompatActivity {
  @Override
  protected void onCreate(Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      setContentView(R.layout.activity_main);
      FrameLayout frameLayout = findViewById(R.id.flutter_root);
      FlutterView flutterView = Flutter.createView(MainActivity.this, getLifecycle(), "root");
      FrameLayout.LayoutParams layout = new FrameLayout.LayoutParams(FrameLayout.LayoutParams.MATCH_PARENT, FrameLayout.LayoutParams.MATCH_PARENT);
      getWindow().getDecorView().post(() -> frameLayout.addView(flutterView, layout));
  }
}
```



文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！