---
title: 一次Observable的empty和never方法的rx源码笔记
date: 2019-01-11 09:35:00
tags: [Observable]
categories: RxJava
description: "我们在用 RxJava  的时候，如果需要在某个地方需要中断事件流，那么直接返回一个 Observable.empty()  ，与它有类似功能的有 Observable.never 。"
---


我们在用 `RxJava` 的时候，如果需要在某个地方需要中断事件流，那么直接返回一个 `Observable.empty()`  ，与它有类似功能的有 `Observable.never` 。

```java
Observable.just(1,2,3,4,5)
    .flatMap((Func1<Integer, Observable<Object>>) integer -> {
        if (integer > 3) {
            return Observable.empty();
            //return Observable.never();
        } else {
            return Observable.just(integer);
        }
    })
    .subscribe(new Subscriber<Object>() {
        @Override
        public void onCompleted() {
            System.out.println("onCompleted");
        }
        @Override
        public void onError(Throwable e) {
            System.out.println("onError");
        }
        @Override
        public void onNext(Object o) {
            System.out.println("onNext");
            System.out.println(Integer.valueOf(o.toString()));
        }
    });
```

`Observable.never` 的输出结果：

```
onNext
1
onNext
2
onNext
3
```

`Observable.empty` 的输出结果：

```
onNext
1
onNext
2
onNext
3
onCompleted
```

从结果可以看出来， `Observable.empty` 会执行 `订阅者` 的 `onCompleted` 方法， 而 `Observable.never` 方法则是立即终止整个流程。

源码分析（RxJava1.3.0）：

```java
public class Observable<T> {
    public static <T> Observable<T> never() {
        return NeverObservableHolder.instance();
    }
    /***部分代码省略***/
    public static <T> Observable<T> empty() {
        return EmptyObservableHolder.instance();
    }
}
```

```java
public enum EmptyObservableHolder implements OnSubscribe<Object> {
    INSTANCE;
    static final Observable<Object> EMPTY = Observable.unsafeCreate(INSTANCE);
    @SuppressWarnings("unchecked")
    public static <T> Observable<T> instance() {
        return (Observable<T>)EMPTY;
    }
    @Override
    public void call(Subscriber<? super Object> child) {
        child.onCompleted();
    }
}
```

```java
public enum NeverObservableHolder implements OnSubscribe<Object> {
    INSTANCE;
    static final Observable<Object> NEVER = Observable.unsafeCreate(INSTANCE);
    @SuppressWarnings("unchecked")
    public static <T> Observable<T> instance() {
        return (Observable<T>)NEVER;
    }
    @Override
    public void call(Subscriber<? super Object> child) {
        // deliberately no op
    }
}
```

 `Observable.empty()`  和 `Observable.never`  我们从源码实现就可以看出来两者的功能。


文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：

<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>
