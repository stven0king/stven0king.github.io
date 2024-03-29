---
title: 从硬件角度去理解协程
date: 2022-02-07 16:19:53
tags: [协程]
categories: Kotlin
description:  "Android 开发者来说 Kotlin 语言已经是很熟悉的了，但 Kotlin 中的 协程 不了解的同学可能还有很多。阅读网络上大多数文章得到的关于 协程 几个关键词：像是线程、不是线程、用户态、协作式。感觉很懵逼，我就问一个 协程 而已为什么出现这么多奇奇怪该的名词。。
"
---

![son](https://img-blog.csdnimg.cn/1d234942dce746f6a8127e49c6bddb66.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Z2Z6buY5Yqg6L29,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

2022年，虎年虎虎生威~！

## 前言

`Android` 开发者来说 `Kotlin` 语言已经是很熟悉的了，但 `Kotlin` 中的 `协程` 不了解的同学可能还有很多。

阅读网络上大多数文章得到的关于 `协程` 几个关键词：

- 像是线程；
- 不是线程；
- 用户态；
- 协作式；

感觉很懵逼，我就问一个 `协程` 而已为什么出现这么多奇奇怪该的名词。

## 协程简介

[维基百科：协程](https://zh.wikipedia.org/wiki/%E5%8D%8F%E7%A8%8B)

`协程（英語：coroutine）`是计算机程序的一类组件，推广了协作式多任务的子例程，允许执行被挂起与被恢复。 相对子例程而言，协程更为一般和灵活，但在实践中使用没有子例程那样广泛。 协程更适合于用来实现彼此熟悉的程序组件，如协作式多任务、异常处理、事件循环、迭代器、无限列表和管道。

## 电脑物理硬件


说 `协程` 之前我们先聊一下计算机硬件相关的知识。


### 物理 cpu 数

指主板上实际插入的 `cpu` 硬件个数（`socket`）。（但是这一概念经常被泛泛的说成是 `cpu` 数，这很容易导致与 `core` 数，`processor` 数等概念混淆，所以此处强调是物理 `cpu` 数）。

由于在主板上引入多个 `cpu` 插槽需要更复杂的硬件支持（连接不同插槽的 `cpu` 到内存和其他资源），通常只会在服务器上才这样做。在家用电脑中，一般主板上只会有一个 `cpu` 插槽。

### 核数

一开始，每个物理 `cpu` 上只有一个核心 `a single core` ，对操作系统而言，也就是同一时刻只能运行一个进程/线程。 为了提高性能，`cpu` 厂商开始在单个物理 `cpu` 上增加核心（实实在在的硬件存在），也就出现了双核心 `cpu`（`dual-core cpu`）以及多核心 `cpu`（`multiple cores`），这样一个双核心 `cpu` 就是同一时刻能够运行两个进程/线程的。

### 超线程技术

> 同时多线程技术（simultaneous multithreading）

> 超线程技术（hyper–threading/HT）

本质一样，是为了提高单个 `core` 同一时刻能够执行的多线程数的技术（充分利用单个 `core` 的计算能力，尽量让其“一刻也不得闲”）。

`simultaneous multithreading` 缩写是 `SMT`，`AMD` 和其他 `cpu` 厂商的称呼。 `hyper–threading` 是 `Intel` 的称呼，可以认为 `hyper–threading` 是 `SMT` 的一种具体技术实现。

![AMD-R7](https://img-blog.csdnimg.cn/8071c3c733d5448f9084520e029ed5d3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Z2Z6buY5Yqg6L29,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)


![Intel-i7](https://img-blog.csdnimg.cn/c862f543fac74f67b8fb7a4bb0522204.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA6Z2Z6buY5Yqg6L29,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)


所以可以这样说：某款采用 `SMT` 技术的 `4核心`  `AMD cpu` 提供了 `8线程` 同时执行的能力；某款采用 `HT` 技术的 `2 核心` `Intel cpu` 提供了 `4 线程` 同时执行的能力。

> 总的逻辑 `cpu` 数 = 物理 `cpu` 数 * 每颗物理 `cpu` 的核心数 * 每个核心的超线程数

## 线程和协程

讲 `协程` 的时候绝对不能不提 `线程` 。

`线程` 是操作系统能够进行运算的最小单位。

在之前一般情况下 `CPU` 的每个核心同一时间只能执行一个线程，除了现在比较新的 `CPU` 拥有上面说的使用 `SMT` 或者 `HT` 技术。

但 `CPU` 的核心数和 `线程` 的个数没有必然关系。举个很简单的例子，我一段代码可以一直创建**100个线程**。

`CPU` 根本不理解自己执行的指令属于哪个 `线程`，`CPU` 也不需要理解这些，它只需只需当前操作系统给它分配的指令就行。

在单核 `CPU` 时代所有的多线程其实都是多任务，多个任务交替使用 `CPU资源` 。

有了多核之后，运行在两个线程的任务才实现正真的并行，但电脑的实际核数永远也达不到我们运算需要的任务数量。所以多个任务交替使用 `CPU资源` 这种情况一直存在，但我们知道 `CPP` 切换执行线程的上下文都是需要消耗资源的，任务数量越多不一定执行效率更高。对于计算密集型的程序有的建议是设置线程的最佳数量为 `CPU` 可执行线程数的 `1.5倍` 或者 `1倍+1` 。

在这个时候我们想到能不能在异步任务之间切换的时候不切换 `CPU` 的上下文状态，这样可以减少很多资源的浪费。或者在 `CPU` 长时间执行 `I/O操作` 的时候让其他例程先执行，提供资源的利用率。

`协程` 就在这个时候产生了，协作式执行多任务的子例程。

这时候我们已经对 `协程` 有了初步的了解了，回头想想文章开头4个描述 `协程` 的说明。

- 像是线程：在部分程序执行的过程中，协程的并发执行就是利用的多线程技术（例如：没有进行改版的 `Java程序` ）。所以说它像是线程；
- 不是线程：并发任务的调度不是都通过操作系统级别线程切换执行，而是程序本身支持单个线程的多个并发任务。所以也可以说它不是线程，可以叫它们纤程 `Fiber` ，或者绿色线程 `GreenThread` 。正如一个进程可以拥有多个线程一样，一个线程也可以拥有多个协程。
- 用户态：不是系统级别的线程而且能自主执行异步任务，这种由程序员自己写程序来管理的轻量级线程叫做**用户空间线程**，具有对内核来说不可见的特性。
- 协作式：要求每个运行中的程序，定位放弃自己的执行权利，让多个任务一起交替执行。[维基百科：协作式多任务](https://zh.wikipedia.org/wiki/%E5%8D%8F%E4%BD%9C%E5%BC%8F%E5%A4%9A%E4%BB%BB%E5%8A%A1)；

## Android中的协程

上面说的 `协程` 减少上下文切换，提供效率，那么 `Android` 的 `kotlin` 支持协程么?

`kotlin` 官方文档说：本质上，协程是轻量级的线程。

但就目前 `Kotlin-JVM` 而言来说 `协程` 它就是线程。其本质上还是一套基于原生 `Java Thread API` 的封装。

可能后续 `Kotlin`  的版本会有正真的协程相关的机制来代替线程。

这个时候可能我们可能就有一些疑问，既然 `协程` 在 `Android` 平台上依旧是 `线程` 并没有提示运行效率，`Java` 中的  `Executor`  和  `Android` 中的 `AsyncTask` 都能提供并发任务，那么 `kotlin` 的 `协程` 它有什么用？后面会有一篇文章单独讲解~！

参考资料：

[一文读懂什么是进程、线程、协程](https://www.cnblogs.com/Survivalist/p/11527949.html)

[Kotlin 协程真的比 Java 线程更高效吗?](https://segmentfault.com/a/1190000021548651###)

[扔物线：Kotlin的协程用例瞥一眼](https://rengwuxian.com/kotlin-coroutines-1/)


文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！