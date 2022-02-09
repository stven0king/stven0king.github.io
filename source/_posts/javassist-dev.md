---
title: 修改AAR和Jar中class文件
date: 2021-01-22 15:10:05
tags: [AAR,jar,class,javassist]
categories: Android
description: "最近帮助同事解决了一个比较棘手的问题，一路采坑的过程比较有意思。在此记录下来。（PS：主要原因是项目比较大，我们只有整个Android项目部分业务侧代码的开发权限。所以解决问题的一些解决问题的常规手段无法使用。） "
---

![hi，2021](https://img-blog.csdnimg.cn/img_convert/cb84db57e4f6bbcf2c70c0bf9740d8ed.png#pic_center)


## 前言
最近帮助同事解决了一个比较棘手的问题，一路采坑的过程比较有意思。在此记录下来。（PS：主要原因是项目比较大，我们只有整个`Android`项目部分业务侧代码的开发权限。所以解决问题的一些解决问题的常规手段无法使用。）

## 问题

需求：`Web`页面中`H5`和`native`交互，保存`base64图片`。

问题：使用现有的已经封装好的`Hybrid`协议，在最后集成测试发现部分手机无法保存成功。

- 调试发现`H5`中使用原有的协议格式调用新协议，无法触发业务侧`native`注册的新协议的日志和断点。

1. 怀疑原有的协议格式问题，当H5使用原有协议格式调用线上已经存在的新协议发现`native`可以端调用成功，此问题排除；
2. 怀疑新协议中的参数问题，H5去掉新协议中的参数发现可以调用到`native`新协议实现。猜测可能是协议参数导致出的问题；
    1. 通过断点找到触发协议调用的地方，也就是`H5`和`native`数据通信的地方。发现目前`Hybrid`协议使用的是`native`端复写`onJsPrompt`方法，拦截`JavaScript`的`prompt()`方法。
    2. 将新协议参数改回来，再次调用。断点在`native`端复写`onJsPrompt`方法中发现传输的数据被截断，数据解析失败无法进行下一个转发到业务测。

### 问题点

项目中原有的`Hybrid协议`使用的`复写prompt方法`作为通信的桥梁，现在遇到了**发大洪水（较大的数据）**出现了阻碍问题。

## 解决方案选择

1. 让`H5`将`base64`格式的图片改为`http`格式图片；
  
   > 图片本就是`H5`绘制出来的，再上传之后客户端再下载交互体验太差；
2. 我们业务侧实现自已一套的`Hybrid`协议；
3. 让项目的基础架构部修改现有的`Hybrid`协议；

   > 晚上发现的bug，明天就需要封测。24小时之内想要完成跨部门的基础架构的改动，很难实现。

最后我们选择的是第二种方案，自己实现一套`Hybrid`协议。

## 解决方案实现

1. 拿到`WebView` 调用`addJavascriptInterface`方法给`H5`环境下添加`JS`对象。
2. 开发`JS`工具让其能按照老协议格式，调用到新的`JS`通信方法。
3. 将拿到的数据解析抛给原来在`onJsPrompt`方法中处理数据的包装类对象。

如果所有的项目代码我们都能改动，那么这个解决方案也就没有难点。难点在于我们现在只有这个`H5`页面的最外层的一个壳`Activity`，而且封装的`WebView`没有为对外暴露我们想要的方法。所以方案的第三步执行产生了问题。

针对这个问题我们有两个解决方案：

![Hybrid-Base64-project.jpg](https://img-blog.csdnimg.cn/img_convert/60faeadb5a7c3cbbe60255c0198cc86d.png#pic_center)


1. 本次通过注入`JS`对象的`Hybrid`通信协议和项目原有的`Hybrid`协议做两套逻辑；
2. 通过多次`hook`黑科技调用到原有其他类中的`dispatch`方法；

如果仅仅是这样也就没有本篇文章了。

好了，啰嗦了这么久了终于开始进入正题了。不知道还有几个同学打算继续看下去。

我们可以拿到项目中所有的`AAR`文件，想着是否能通过修改源代码使之提供我们想要的`API`，然后通过升级 `AAR`版本解决问题。好了本文的重点已经出来了**修改AAR中class文件**。



## 修改AAR中class文件

### 方案一

> 先把`AAR`中的想要修改的`class`删除，重新打包为新的`AAR`。项目依赖新版本`AAR`，然后在项目对应的包下创建一个相同的类。
> 1. 将原有的`class`文件内容反编译之后拷贝到新建的类中，直接运行。
> 2. 将原有的`class`文件内容反编译之后拷贝到新建的类中。最后重新编译生成的`class`再添加到`AAR`中重新打包生成新的`AAR`。

如果类被混淆过的，那么这个方案基本废掉了。因为反编译出的`class`的内容里面存在大量包和类名相同的情况，这个再编译期间无法确认本次调用时使用的类还是包。

举个例子，混淆之后经常看到下面的结构。

```java
com.xx.a
com.xx.a.a
```

在写下面代码的时候会提示类`a`下没有类`a`，而不是去包`a`下找类`a`。

```java
a ma = new com.xx.a.a();
```

### 方案二

![aop-asm.JPEG](https://img-blog.csdnimg.cn/img_convert/8c9a841fee6b7bb598bdac9ec03a5377.png#pic_center)


根据上图就可以看出这个方案是进行切面编程。我们仅仅需要在某个类中添加一两个方法，去解决访问受限的问题。考虑到当前这个问题的难易程度我们选择`Javassist`。因为` Javassist`源代码级`API`比`ASM`中实际的字节码操作更容易使用，无需深入了解`JVM`规范也能使用。

`Javassist` [官方文档](http://www.javassist.org/html/)

`jar`包下载地址 [Github:javassist](https://github.com/jboss-javassist/javassist)

```java
//需要添加的方法
//public void executeJSCmd(String var1) {
//    if (this.mActionDispatcher != null) {
//        Message var2 = this.mActionHandler.obtainMessage(0);
//        var2.obj = var1;
//        this.mActionHandler.sendMessage(var2);
//    }
//}
//需要操作的class的类名：com.xxx.android.web.webview.BaseWebChromeClient
public class Test {
    public static void main(String[] args) throws NotFoundException, CannotCompileException, IOException {
        ClassPool classPool = ClassPool.getDefault();
        // 必须将class文件放在这个工程编译后的class文件中，路径也对应起来
        CtClass ctClass = classPool.get("com.xxx.android.web.webview.BaseWebChromeClient");
        CtMethod newmethod = CtNewMethod.make("public void executeJSCmd(String message) { if (this.mActionDispatcher != null) { android.os.Message msg = this.mActionHandler.obtainMessage(0); msg.obj = message; this.mActionHandler.sendMessage(msg); } }",ctClass);
        ctClass.addMethod(newmethod);
        ctClass.writeFile();
    }
}
```

需要注意的是，比如我们添加的方法中涉及到了其他的类需要写全路径`android.os.Message`，而且这个类相关的`jar`包也必须添加到运行环境中（也可以将这个类的class文件放着这个工程编译后的class文件目录中），否则执行时候会报一下的错误。

```java
Exception in thread "main" javassist.CannotCompileException: [source error] no such class: android.os.Message
	at javassist.CtNewMethod.make(CtNewMethod.java:78)
	at javassist.CtNewMethod.make(CtNewMethod.java:44)
	at com.test.pattern.Test.main(Test.java:13)
```

### 注意点

替换或者删除`jar`中的`class`的时候最好不要解压然后再使用命名打包，我自己在`Max`电脑上使用命令打`jar`包的时候会有一个`.DS_Store`文件。我使用的`BetterZip`**压缩&解压**工具，在不解压的情况下进行`jar`包中的`class`的**添加和删除**操作非常方便。

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！