---
title: Android内存泄漏检测工具使用手册
date: 2020-06-05 17:57:35
tags: [内存泄漏]
categories: Android
description: "性能优化除过我们平时自己设计和开发之外就得考虑使用工具进行检测。Android关于能够定位和剖析问题的内存工具有很多，但不是每个工具所有场景都能覆盖到。这篇文章主要介绍LeaKCanary、shark、Android Profile、MAT、Jhat、dumpsys meminfo、GC Log等。"
---

## 前言

性能优化除过我们平时自己设计和开发之外就得考虑使用工具进行检测。`Android` 关于能够定位和剖析问题的内存工具有很多，但不是每个工具所有场景都能覆盖到。

- `DDMS`
- `LeakCanary`
- `haha/shark`
- `Android Profile`
- `MAT`
- `Jhat`
- `dumpsys meminfo`
- `APT`
- `LeakInspector`
- `Chrome Devtool`
- `GC Log` 

现在对平时能发现问题，而且使用简单的一些工具的使用进行整理，并且对这个 `LeakCanaryTestActivity` 页面进行内存泄漏的分析。

```java
public class LeakCanaryTestActivity extends BaseActivity {
    private static Test test;
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        test = new Test(this);
    }
    private static class Test{
        public Test(Context context) {
            this.context = context;
        }
        private Context context;
        private int a;
        private int b;
    }
}
```

## LeakCanary

[LeakCanary 官网](https://square.github.io/leakcanary/)

`LeakCanary` 的原理很简单: 在 `Activity` 或 `Fragment` 被销毁后, 将他们的引用包装成一个 `WeakReference `, 然后将这个 `WeakReference` 关联到一个 `ReferenceQueue` 。查看`ReferenceQueue`中是否含有 `Activity` 或 `Fragment` 的引用。如果没有 **触发GC** 后再次查看。还是没有的话就说明回收成功, 否则可能发生了泄露. 这时候开始 `dump` 内存的信息,并分析泄露的引用链。

### 在Android中接入LeakCanary


```groovy
dependencies {
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.3'
}
```

在 `LeakCanary2.0` 之前我们接入的时候需要在 `Application.onCreate` 方法中显式调用 `LeakCanary.install(this);` 开启 `LeakCanary` 的内存监控。

`LeakCanary2.0` 开始通过自己注册的 `provider` 自己开启 `LeakCanary` 的内存监控。我们平时开发用的 `Instant Run` 运行过程中也使用的是这种静默方式进行启动。

```xml
<provider android:name="com.android.tools.ir.server.InstantRunContentProvider" 
    android:multiprocess="true" 
  android:authorities="com.tzx.androidcode.com.android.tools.ir.server.InstantRunContentProvider"/>
```

### LeakCanary内存泄漏分析


在进行 `debug` 或者 `UI自动化` 测试的时候，我们会在通知栏看到有关内存泄漏的提示。查看详情后我们能看到相关的内存泄漏具体位置，存在泄露的成员变量都用波浪线进行的标识。

![LeakCanary-user](https://img-blog.csdnimg.cn/20200605194330839.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)
### 内存泄漏上报到服务端

`LeakCanary` 升级到 `2.0` 的  `beta` 和 `final` 版本之后 [shark 官网](https://square.github.io/leakcanary/shark/) 文档提供的的内存泄漏上报方式对应的 `API` 已经过时，我们需要实现新的接口将 `LeakCanary` 捕获的内存泄漏进行上报。

```java
class LeakUploader : OnHeapAnalyzedListener {
  override fun onHeapAnalyzed(heapAnalysis: HeapAnalysis) {
    TODO("Upload heap analysis to server")
    //HeapAnalysis的toString和2.0之前的版本的LeakCanary.leakInfo获得的信息类似
    println(heapAnalysis)
  }
}
class MyApplication : Application() {
  override fun onCreate() {
    super.onCreate()
    LeakCanary.config = LeakCanary.config.copy(
        onHeapAnalyzedListener = LeakUploader()
    )
  }
}
```

## Shark

[shark 官网](https://square.github.io/leakcanary/shark/)

`Shark`是为 `LeakCanary 2` 提供支持的堆分析器，它是`Kotlin`独立堆分析库，可在低内存占用情况下高速运行（PS：`LeakCanary 2` 之前的堆分析库是 `haha` ，[haha Git地址](https://github.com/square/haha)）。

此处说的 `LeakCanary 2` 为 `beta` 和 `final` 版本，`alpha` 版依旧是用的 `haha` 只不过是用 `kotlin` 写的。

`Shark` 在为 `LeakCanary 2` 提供支持的同事也提供 **Shark CLI** 支持。

`Shark` 命令行界面（`CLI`）使您可以直接从计算机分析堆。它可以转储安装在已连接的 `Android` 设备上的应用程序的堆，对其进行分析，甚至剥离所有敏感数据（例如PII，密码或加密密钥）的堆转储，这在共享堆转储时非常有用。

### Shark分析当前应用的内存泄漏情况

```shell
shark-cli --device 设备id --process 包名 analyze
```

![shark-cli-analyze](https://img-blog.csdnimg.cn/20200605194413812.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

同时支持混淆后的内存泄漏分析，利用`mapping`文件进行可读性还原。

```shell
shark-cli -d 设备id -p 包名 -m 混淆文件 analyze
```

![shark-cli-analyze-mapping](https://img-blog.csdnimg.cn/20200605194453407.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

### Shark分析hprof文件

```shell
shark-cli -h 生成的hprof文件 analyze
```

![shark-cli-analyze-hprof](https://img-blog.csdnimg.cn/20200605194524639.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

## Android Profile

**Android Profiler**分为三大模块： **cpu**、**内存** 、**网络**。

[官网：使用 Memory Profiler 查看 Java 堆和内存分配](https://developer.android.com/studio/profile/memory-profiler?hl=zh-cn)

`Memory Profiler` 是`Android Profiler`中的一个组件，它可以帮助您识别内存泄漏和内存溢出，从而导致存根、冻结甚至应用程序崩溃。它显示了应用程序内存使用的实时图，让您捕获堆转储、强制垃圾收集和跟踪内存分配。

### 捕获堆转储进行分析

![profiler-docs](https://img-blog.csdnimg.cn/20200605194608555.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

在列表的顶部，您可以使用右下拉菜单在列表之间切换:

- `Arrange by class`： 根据类名分配。
- `Arrange by package`：根据包名分配。
- `Arrange by callstack`: 根据调用堆栈排序。

查看堆转储后的信息：

- 您的应用程序分配了哪些类型的对象，以及每个对象的数量;
- 每个对象使用多少内存;
- 每个对象的引用被保留在你的代码中;
- 调用堆栈，用于分配对象的位置（只有在记录分配时捕获堆转储）;

## MAT安装

打开 `Eclipse->help->Eclipse Marketplce`，搜索`Memory Analyze`进行安装，安装完成后重启 `Eclipse`。

![marketplace-memory-analyze](https://img-blog.csdnimg.cn/20200605194648980.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

## MAT使用

将`dump heap` 生成的 `hprof` 文件转化为**MAT**能处理的`hprof` 文件。

执行 `android.os.Debug.dumpHprofData(hprofPath)`  生成 `hprof` 文件，**执行之前记得进行GC**。

`hprof-conv` 位于 `sdk/platform-tools/hprof-conv` 。

```shell
hprof-conv memory-android.hprof memory-mat.hprof
```

### MAT处理导入hprof文件

![mat-overview](https://img-blog.csdnimg.cn/20200605194746623.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)


`Action` 有一下几个视图：

| 视图              | 含义                                           |
| ----------------- | ---------------------------------------------- |
| Histogram         | 列举内存中对象存在的个数和大小，以及对于的名称 |
| Dominator Tree    | 站在对象的角度查看他们的内存情况               |
| Top Consumers     | 该视图会显示可能的内存泄漏点                   |
| Duplicate Classes | 检测由多个类加载器加载的类                     |

### 寻找内存泄漏的类

根据内存中类的对象实例数量，判断该类对象是否被泄露。

![mat-histogram](https://img-blog.csdnimg.cn/20200605195109947.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

我们可以利用提供的多种检索方式进行目标类的检索，我这里用包名作为检索要素。

> **Shallow Size**

- 对象自身占用的内存大小，不包括它引用的对象。
- 针对非数组类型的对象，它的大小就是对象与它所有的成员变量大小的总和。当然这里面还会包括一些`java`语言特性的数据存储单元。
- 针对数组类型的对象，它的大小是数组元素对象的大小总和。

> **Retained Size**

`Retained Size`  = `当前对象大小` + `当前对象可直接或间接引用到的对象的大小总和`。(间接引用的含义：A->B->C, C就是间接引用。如果`B` 和 `C ` 没有被其他对象引用，那么 `RetainedSize-A = ShallowSize(A + B + C)` 它和 [Dominator](https://en.wikipedia.org/wiki/Dominator_(graph_theory)) 比较相似)
换句话说，`Retained Size`就是当前对象被`GC`后，从`Heap`上总共能释放掉的内存。
不过，释放的时候还要排除被`GC Roots`直接或间接引用的对象。他们暂时不会被被当做`Garbage`。

从上图可以看出 `MainActivity` 、`LeakCanaryTestActivity` 和 `LeakCanaryTestActivity$a` 都有一个实例没有被回收。

### 分析被泄露的类的引用关系

选择没有回收的类，进行 `list objects -> with incoming references` 操作得到被引用的对象。

![mat-histogram-list-object](https://img-blog.csdnimg.cn/20200605195136445.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

> `with outgoing references` : 该对象内部引用了那些其他对象；
>
> `with incoming references` : 该对象被谁进行了引用；

得到被引用的类之后，进行 `Path To GC Roots -> exclude all phantom/weak/soft etc. references`  操作，得到所有引用类型的引用。

![mat-histogram-list-gcroot](https://img-blog.csdnimg.cn/20200605195159729.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

> `StrongReference`(强引用)：通常我们编写的代码都是 `StrongReference`，于此对应的是强可达性，只有去掉强可达，对象才被回收。

> `SoftReference`(软引用)：只要有足够的内存，就一直保持对象，直到发现内存吃紧且没有`StrongReference`时才回收对象。一般可用来实现缓存,需要获取对象时，可以调用get方法。

> `WeakReference`(弱引用)：随时可能会被垃圾回收器回收，不一定要等到虚拟机内存不足时才强制回收。要获取对象时，同样可以调用`get`方法。

>`PhantomReference`(虚引用)：根本不会在内存中保持任何对象，你只能使用`PhantomReference`本身。一般用于在进入`finalize()`方法后进行特殊的清理过程。

### 找到最终的泄漏的地方

![mat-histogram-list-gcroot-result](https://img-blog.csdnimg.cn/20200605195226342.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

从这个图中我们可以可以得到：

1. `LeakCanaryTestActivity` 的一个实例被它的内部类 `LeakCanaryTestActivity$Test` 的成员变量 `context` 所持有；
2. `LeakCanaryTestActivity$Test` 的一个实例又被 `LeakCanaryTestActivity` 的成员变量 `test` 所持有。

### Merge对比分析

如果我们没有明确的目标类，我们可以将两个 `hprof文件（泄漏前、泄漏后）` 进行对比。

![mat-merge](https://img-blog.csdnimg.cn/20200605195250377.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

选择泄漏之前的 `hprof文件` 进行对比。

![mat-gcroot-merge](https://img-blog.csdnimg.cn/20200605195427161.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

对比会得到哪些实例对象数量的增加和减少。如上图所示对比结果为 `LeakCanaryTestActivity` 和 `LeakCanaryTestActivity$a` （此处的`a` 为混淆之后的 `Test`）两个类梳理分别增加1个。

我们继续向上面**MAT**分析步骤一样操作：

1. 进行 `list objects -> with incoming references` 操作;

2. 进行 `Path To GC Roots -> exclude all phantom/weak/soft etc. references`  操作;

![mat-merge-result](https://img-blog.csdnimg.cn/20200605195313781.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

最终得到的结果和之前分析的相同的。

## Jhat-Java自带的性能监测工具

[Java8 jhat Analyzes the Java heap docs](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jhat.html)

`JHat` 是 `Oracle` 推出的一款 `Hprof` 分析软件，它和 `MAT` 并称为 **Java 内存静态分析利器**。不同于 `MAT` 的单人界面式分析，`jHat` **使用多人界面式分析**。它被 **内置在 JDK 中**，在命令行中输入 `jhat` 命令可查看有没有相应的命令。

```shell
➜  Desktop jhat
ERROR: No arguments supplied
Usage:  jhat [-stack <bool>] [-refs <bool>] [-port <port>] [-baseline <file>] [-debug <int>] [-version] [-h|-help] <file>

	-J<flag>          Pass <flag> directly to the runtime system. For
			  example, -J-mx512m to use a maximum heap size of 512MB
	-stack false:     Turn off tracking object allocation call stack.
	-refs false:      Turn off tracking of references to objects
	-port <port>:     Set the port for the HTTP server.  Defaults to 7000
	-exclude <file>:  Specify a file that lists data members that should
			  be excluded from the reachableFrom query.
	-baseline <file>: Specify a baseline object dump.  Objects in
			  both heap dumps with the same ID and same class will
			  be marked as not being "new".
	-debug <int>:     Set debug level.
			    0:  No debug output
			    1:  Debug hprof file parsing
			    2:  Debug hprof file parsing, no server
	-version          Report version number
	-h|-help          Print this help and exit
	<file>            The file to read

For a dump file that contains multiple heap dumps,
you may specify which dump in the file
by appending "#<number>" to the file name, i.e. "foo.hprof#3".

All boolean options default to "true"
```

`Jhat` 使用的 `hprof` 文件和 `MAT` 一样都需要使用 `hprof-conv` 进行 `hprof` 转化。

使用 `Jhat` 分析完 `hprof` 文件后会给一个 `Server port` ，比如 `7000` 。那么我们可以访问 `http://localhost:7000/` 查看分析结果。

![jhat-main](https://img-blog.csdnimg.cn/20200605195500463.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

以包为单位展示所有的类，我们下拉到最底部可以看到有其他的查询方式。

![jhat-other-queries](https://img-blog.csdnimg.cn/20200605195530171.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

### Show heap histogram

我们可以看到对应的类的内存实例数量以及占用对应的内存大小。

http://localhost:7000/histo/

![jhat-histo](https://img-blog.csdnimg.cn/20200605195548301.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

### Execute Object Query Language (OQL) query

可以使用  `OQL ` 查询~!

`OQL` 查询语法与 `Visual VM` 的 `OQL` 类似~ 基本语法如下：

```sql
 select <JavaScript expression to select>
         [ from [instanceof] <class name> <identifier>
         [ where <JavaScript boolean expression to filter> ] ]
```

![jhat-oql-result](https://img-blog.csdnimg.cn/20200605195614229.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

我们点击某个类之后可以看到该类的详细信息：

![jhat-class-detail](https://img-blog.csdnimg.cn/20200605195634698.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

- `Exclude subclasses` 相当于**MAT** 的 `with outgoing references` : 该对象内部引用了那些其他对象；

- `Include subclasses` 相当于MAT 的 `with incoming references` : 该对象被谁进行了引用；

![jhat-class-instances](https://img-blog.csdnimg.cn/20200605195805208.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

先查看类的实例，然后再查看每个实例的相关引用情况。

![jhat-class-object](https://img-blog.csdnimg.cn/20200605195824649.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

# dumpsys meminfo

 `Android` 系统是基于 `Linux` 内核的操作系统，所以在 `Linux` 中查看内存使用情况的命令在 `Android` 手机上也能使用比如 `top` 命令。除此之外

- `procrank` ：获取所有进程的内存使用情况，排序是按照 `Pss` 大小，详细输出每个 `PID` 对应的 `Vss` 、 `Rss`      `Pss`  、`Uss`  、 `Swap`  、 `PSwap` 、 `USwap` 、 `ZSwap` 和 `cmdline`。但该命令使用需要 `root` 环境。

一般来说内存占用大小有如下规律：`VSS`  >=  `RSS`  >= `PSS`  >= `USS`

| 简称 | 全称                  | 含义                   | 等价                                                         |
| ---- | --------------------- | ---------------------- | ------------------------------------------------------------ |
| VSS  | Virtual Set Size      | 虚拟耗用内存           | （包含共享库占用的内存）是单个进程全部可访问的地址空间       |
| RSS  | Resident Set Size     | 实际使用物理内存       | (包含共享库占用的内存）是单个进程实际占用的内存大小，对于单个共享库， 尽管无论多少个进程使用，实际该共享库只会被装入内存一次。 |
| PSS  | Proportional Set Size | 实际使用的物理内存     | （比例分配共享库占用的内存）                                 |
| USS  | Unique Set Size       | 进程独自占用的物理内存 | （不包含共享库占用的内存）USS 是一个非常非常有用的数字， 因为它揭示了运行一个特定进程的真实的内存增量大小。如果进程被终止， USS 就是实际被返还给系统的内存大小。 |

`USS` 是针对某个进程开始有可疑内存泄露的情况，进行检测的最佳数字。怀疑某个程序有内存泄露可以查看这个值是否一直有增加。

- `cat /proc/meminfo` ：展示系统整体的内存情况，按照内存类型进行分类。
- `free` ：查看可用内存，缺省单位为`KB`。该命令比较简单、轻量，专注于查看剩余内存情况。数据来源于 `/proc/meminfo` 。

最后一个是本次叙述的重点 `dumpsys` 。

```shell
dumpsys [options]
               meminfo 显示内存信息
               cpuinfo 显示CPU信息
               account 显示accounts信息
               activity 显示所有的activities的信息
               window 显示键盘，窗口和它们的关系
               wifi 显示wifi信息
```

使用 `dumpysys meminfo` 查看内存信息,后面可以添加 `pid | packagename` 查看该应用程序的内存信息。

```shell
~/Desktop adb shell dumpsys meminfo com.tzx.androidcode
Applications Memory Usage (in Kilobytes):
Uptime: 131873995 Realtime: 240892295

** MEMINFO in pid 19924 [com.tzx.androidcode] **
                   Pss  Private  Private  SwapPss     Heap     Heap     Heap
                 Total    Dirty    Clean    Dirty     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------
  Native Heap    11062    11032        0       99    38912    21656    17255
  Dalvik Heap     4079     3984        0        0     5638     2819     2819
 Dalvik Other     1405     1404        0        1
        Stack       64       64        0        0
       Ashmem        2        0        0        0
      Gfx dev     2052     2052        0        0
    Other dev        8        0        8        0
     .so mmap      959       80       64        4
    .jar mmap     1062        0       24        0
    .apk mmap      109        0        0        0
    .ttf mmap       33        0        0        0
    .dex mmap     4513       36     4392        0
    .oat mmap      197        0        0        0
    .art mmap     6229     5924       16        1
   Other mmap      680      196       96        0
   EGL mtrack    19320    19320        0        0
    GL mtrack     6392     6392        0        0
      Unknown     1106     1092        0       15
        TOTAL    59392    51576     4600      120    44550    24475    20074

 App Summary
                       Pss(KB)
                        ------
           Java Heap:     9924
         Native Heap:    11032
                Code:     4596
               Stack:       64
            Graphics:    27764
       Private Other:     2796
              System:     3216

               TOTAL:    59392       TOTAL SWAP PSS:      120

 Objects
               Views:       82         ViewRootImpl:        2
         AppContexts:        8           Activities:        2
              Assets:       11        AssetManagers:        0
       Local Binders:       22        Proxy Binders:       41
       Parcel memory:       10         Parcel count:       24
    Death Recipients:        2      OpenSSL Sockets:        0
            WebViews:        0

 SQL
         MEMORY_USED:        0
  PAGECACHE_OVERFLOW:        0          MALLOC_SIZE:        0
```

`Android` 程序内存被分为2部分：`native` 和 `虚拟机` ，`虚拟机` 就是我们平常说的 `java堆`，我们创建的对象是在这里面分配的，而 `bitmap` 是直接在 `native` 上分配的，对于内存的限制是` native+dalvik` 不能超过最大限制。以上信息可以看到该应用程序占用的 `native` 和 `dalvik`，对于分析内存泄露，内存溢出都有极大的作用。

## 读取垃圾回收消息(GC Log)

[官网：读取垃圾回收消息](https://developer.android.com/studio/debug/am-logcat?hl=zh-cn#memory-logs)

### Dalvik 日志消息

在 `Dalvik`（而不是 `ART`）中，每个 `GC` 都会将以下信息输出到 `logcat` 中：

```java
D/dalvikvm(PID): GC_Reason Amount_freed, Heap_stats, External_memory_stats, Pause_time
```

示例：

```java
D/dalvikvm( 9050): GC_CONCURRENT freed 2049K, 65% free 3571K/9991K, external 4703K/5261K, paused 2ms+2ms
```

### ART 日志消息

与 `Dalvik` 不同，`ART` 不会为未明确请求的 `GC` 记录消息。只有在系统认为 `GC` 速度较慢时才会输出 `GC` 消息。更确切地说，仅在 `GC` 暂停时间超过 5 毫秒或 `GC` 持续时间超过 100 毫秒时。如果应用未处于可察觉到暂停的状态（例如应用在后台运行时，这种情况下，用户无法察觉 `GC` 暂停），则其所有 `GC` 都不会被视为速度较慢。系统一直会记录显式 `GC`。

`ART` 会在其垃圾回收日志消息中包含以下信息：

```java
I/art: GC_Reason GC_Name Objects_freed(Size_freed) AllocSpace Objects,
        Large_objects_freed(Large_object_size_freed) Heap_stats LOS objects, Pause_time(s)
```

示例：

```java
I/art : Explicit concurrent mark sweep GC freed 104710(7MB) AllocSpace objects,
        21(416KB) LOS objects, 33% free, 25MB/38MB, paused 1.230ms total 67.216ms
```





文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我的公共号：

<center>![振兴书城](http://upload-images.jianshu.io/upload_images/1319879-612c4c66d40ce855.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>