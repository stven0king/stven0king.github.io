---
title: Java线程池实现原理和源码分析
date: 2021-2-19 18:50:43
tags: [线程池]
categories: Java
description:  "本文章是从2019年11月下旬开始打开写的，一直拖到2020年的年尾才开始写，直到2021年年初才写完。"
---


# Java线程池实现原理和源码分析
![](https://img-blog.csdnimg.cn/img_convert/dc61b582844cf193af46b8f90f480960.png#pic_center)


# 前言

本文章是从2019年11月下旬开始打开写的，一直拖到2020年的年尾才开始写，直到2021年年初才写完。

时间太快也太慢~！

依稀记得2019年10月份的时候某东从创业公司离职打算面试找工作，他问我线程池你会么？然后给我他发了一篇我2017年写的笔记《Java并发编程之线程池必用知识点》，他说就这么点？我当时想线程池也差不多就这么多吧~！

2019年11月9号我和某东一起从大望路做815公交去燕郊。当时只是因为我正在学习一部分多线程相关的知识点，刚好公交车上没啥事情我俩就唠了唠。当时他问了我一些线程池的问题，我觉得在平时的工作线程池知道该怎么用就行顶多优化一下核心线程数量。主要讨论的还是多线程并发和锁的相关问题。

年底工作一般比较忙也就很少进行自我学习了，隔了一周想起了某东问我的问题“线程池中线程是怎么产生的，任务是怎么等待执行？”。

自己也不是很清楚这块的逻辑，临时把这个TODO项纪录了下来，想以后有时间了研究一下。结果这个以后跨越了2019年和2020年直接来到了2021年。

原谅我的啰里啰嗦，关键这个篇文章时间跨度太长了，给我的印象太深了。不得不说道说道，下面开始进去正题~！

`JDK1.8`的源码来分析Java线程池的核心设计与实现。

本文参考了[Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)这篇文章中的部分内容。

[Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)这篇文章写的非常好，除过本文内容之外这篇文章还讲述了的关于**线程池的背景**，**线程池在业务中的实践**和**动态化线程池**等，所以想了解线程池关于这些类容的可以阅读[Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)这篇文章。

如果读者为做服务端开发的同学那么强烈建议阅读[Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)。

# 外观

**外观**主要是我们平常使用线程池的时候所看到的一些点。

- 继承关系；
- 构造函数；
- 构造函数中的参数；
- 构造函数中的阻塞队列；
- 线程池的创建；
- 构造函数中的拒绝策略；

## 线程池继承关系

![ThreadPoolExecutor-uml.png](https://img-blog.csdnimg.cn/img_convert/a664410799a9e282b2235609a652ef21.png#pic_center =240x300)



`ThreadPoolExecutor`实现的顶层接口是`Executor`，在接口`Executor`中用户无需关注如何创建线程，如何调度线程来执行任务，用户只需提供`Runnable`对象，将任务的运行逻辑提交到执行器`Executor`中，由`Executor`框架完成线程的调配和任务的执行部分。

`ExecutorService`接口增加了一些能力：

1. 扩充执行任务的能力，补充可以为一个或一批异步任务生成`Future`的方法；
2. 提供了管控线程池的方法，比如停止线程池的运行。

`AbstractExecutorService`则是上层的抽象类，将执行任务的流程串联了起来，保证下层的实现只需关注一个执行任务的方法即可。

最下层的实现类`ThreadPoolExecutor`实现最复杂的运行部分:

1. 可以自动创建、管理和复用指定数量的一组线程，适用方只需提交任务即可

2. 线程安全，`ThreadPoolExecutor`内部有状态、核心线程数、非核心线程等属性，广泛使用了**CAS**和**AQS**锁机制避免并发带来的冲突问题

3. 提供了核心线程、缓冲阻塞队列、非核心线程、抛弃策略的概念，可以根据实际应用场景进行组合使用

4. 提供了`beforeExecute` 和`afterExecute()`可以支持对线程池的功能进行扩展

## 构造函数

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```
- **corePoolSize**：线程池的核心线程数，一般情况下不管有没有任务都会一直在线程池中一直存活，只有在 `ThreadPoolExecutor`中的方法`allowCoreThreadTimeOut(boolean value)`设置为`true`时，闲置的核心线程会存在超时机制，如果在指定时间没有新任务来时，核心线程也会被终止，而这个时间间隔由第3个属性 `keepAliveTime`指定。
- **maximumPoolSize**：线程池所能容纳的最大线程数，当活动的线程数达到这个值后，后续的新任务将会被阻塞。
- **keepAliveTime**：控制线程闲置时的超时时长，超过则终止该线程。一般情况下用于非核心线程，只有在 `ThreadPoolExecutor`中的方法`allowCoreThreadTimeOut(boolean value)`设置为`true`时，也作用于核心线程。
- **unit**：用于指定`keepAliveTime`参数的时间单位，`TimeUnit`是个`enum`枚举类型，常用的有：`TimeUnit.HOURS(小时)`、`TimeUnit.MINUTES(分钟)`、`TimeUnit.SECONDS(秒) `和 `TimeUnit.MILLISECONDS(毫秒)`等。
- **workQueue**：线程池的任务队列，通过线程池的`execute(Runnable command)`方法会将任务`Runnable`存储在队列中。
- **threadFactory**：线程工厂，它是一个接口，用来为线程池创建新线程的。
- **handler**：拒绝策略，所谓拒绝策略，是指将任务添加到线程池中时，线程池拒绝该任务所采取的相应策略。

## 成员变量

```java
/**
 * 任务阻塞队列 
 */
private final BlockingQueue<Runnable> workQueue; 
/**
 * 非公平的互斥锁（可重入锁）
 */
private final ReentrantLock mainLock = new ReentrantLock();
/**
 * 线程集合一个Worker对应一个线程，没有核心线程的说话，只有核心线程数
 */
private final HashSet<Worker> workers = new HashSet<Worker>();
/**
 * 配合mainLock通过Condition能够更加精细的控制多线程的休眠与唤醒
 */
private final Condition termination = mainLock.newCondition();
/**
 * 线程池中线程数量曾经达到过的最大值。
 */
private int largestPoolSize;  
/**
 * 已完成任务数量
 */
private long completedTaskCount;
/**
 * ThreadFactory对象，用于创建线程。
 */
private volatile ThreadFactory threadFactory;  
/**
 * 拒绝策略的处理句柄
 * 现在默认提供了CallerRunsPolicy、AbortPolicy、DiscardOldestPolicy、DiscardPolicy
 */
private volatile RejectedExecutionHandler handler;
/**
 * 线程池维护线程（超过核心线程数）所允许的空闲时间
 */
private volatile long keepAliveTime;
/**
 * 允许线程池中的核心线程超时进行销毁
 */
private volatile boolean allowCoreThreadTimeOut;  
/**
 * 线程池维护线程的最小数量，哪怕是空闲的  
 */
private volatile int corePoolSize;
/**
 * 线程池维护的最大线程数量，线程数超过这个数量之后新提交的任务就需要进入阻塞队列
 */
private volatile int maximumPoolSize;
```

## 创建线程池

`Executors`提供获取几种常用的线程池的方法：

- **缓存程线程池**

> `newCachedThreadPool`是一个可根据需要创建新线程的线程池，但是在以前构造的线程可用时将重用它们。对于执行很多短期异步任务的程序而言，这些线程池通常可提高程序性能。调用 `execute()` 将重用以前构造的线程（如果线程可用）。如果现有线程没有可用的，则创建一个新线程并添加到池中。终止并从缓存中移除那些已有 60 秒钟未被使用的线程。因此，长时间保持空闲的线程池不会使用任何资源。注意，可以使用 `ThreadPoolExecutor` 构造方法创建具有类似属性但细节不同（例如超时参数）的线程池。

```java
public static ExecutorService newCachedThreadPool() {
  return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                  60L, TimeUnit.SECONDS,
                  new SynchronousQueue<Runnable>());
}
```

- **单线程线程池**

> `newSingleThreadExecutor` 创建是一个单线程池，也就是该线程池只有一个线程在工作，所有的任务是串行执行的，如果这个唯一的线程因为异常结束，那么会有一个新的线程来替代它，此线程池保证所有任务的执行顺序按照任务的提交顺序执行。

```java
public static ExecutorService newSingleThreadExecutor() {
  return new FinalizableDelegatedExecutorService
    (new ThreadPoolExecutor(1, 1,
                0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<Runnable>()));
}
```

- **固定大小线程池**

> `newFixedThreadPool` 创建固定大小的线程池，每次提交一个任务就创建一个线程，直到线程达到线程池的最大大小，线程池的大小一旦达到最大值就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新线程。

```java
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
  return new ThreadPoolExecutor(nThreads, nThreads,
                  0L, TimeUnit.MILLISECONDS,
                  new LinkedBlockingQueue<Runnable>(),
                  threadFactory);
}
```

- **单线程线程池**

> `newScheduledThreadPool` 创建一个大小无限的线程池，此线程池支持定时以及周期性执行任务的需求。

```
public static ScheduledExecutorService newScheduledThreadPool(
    int corePoolSize, ThreadFactory threadFactory) {
  return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
}
public ScheduledThreadPoolExecutor(int corePoolSize,
                   ThreadFactory threadFactory) {
  super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
      new DelayedWorkQueue(), threadFactory);
}
```

我们可以看出来上面的方法一共使用了`DelayedWorkQueue`、`LinkedBlockingQueue` 和 `SynchronousQueue`。这个就是线程核心之一的阻塞队列。

## 任务阻塞队列

![BlockingQueue.png](https://img-blog.csdnimg.cn/img_convert/468012bde8d6b6b85e366776d6dc9894.png#pic_center)


**它一般分为直接提交队列、有界任务队列、无界任务队列、优先任务队列；**

### SynchronousQueue

1、**直接提交队列**：设置为`SynchronousQueue`队列，`SynchronousQueue`是一个特殊的`BlockingQueue`，它没有容量，每执行一个插入操作就会阻塞，需要再执行一个删除操作才会被唤醒，反之每一个删除操作也都要等待对应的插入操作。

使用`SynchronousQueue`队列，提交的任务不会被保存，总是会马上提交执行。如果用于执行任务的线程数量小于`maximumPoolSize`，则尝试创建新的进程，如果达到`maximumPoolSize`设置的最大值，则根据你设置的`handler`执行拒绝策略。因此这种方式你提交的任务不会被缓存起来，而是会被马上执行，在这种情况下，你需要对你程序的并发量有个准确的评估，才能设置合适的`maximumPoolSize`数量，否则很容易就会执行拒绝策略；

### ArrayBlockingQueue

2、**有界的任务队列**：有界的任务队列可以使用`ArrayBlockingQueue`实现，如下所示：

```
pool = new ThreadPoolExecutor(1, 2, 1000, TimeUnit.MILLISECONDS, new ArrayBlockingQueue<Runnable>(10),Executors.defaultThreadFactory(),new ThreadPoolExecutor.AbortPolicy());
```

使用`ArrayBlockingQueue`有界任务队列，若有新的任务需要执行时，线程池会创建新的线程，直到创建的线程数量达到`corePoolSize`时，则会将新的任务加入到等待队列中。若等待队列已满，即超过`ArrayBlockingQueue`初始化的容量，则继续创建线程，直到线程数量达到`maximumPoolSize`设置的最大线程数量，若大于`maximumPoolSize`，则执行拒绝策略。在这种情况下，线程数量的上限与有界任务队列的状态有直接关系，如果有界队列初始容量较大或者没有达到超负荷的状态，线程数将一直维持在`corePoolSiz`e以下，反之当任务队列已满时，则会以`maximumPoolSize`为最大线程数上限。

### LinkedBlockingQueue

3、**无界的任务队列**：无界任务队列可以使用`LinkedBlockingQueue`实现，如下所示：

```
pool = new ThreadPoolExecutor(1, 2, 1000, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>(),Executors.defaultThreadFactory(),new ThreadPoolExecutor.AbortPolicy());
```

使用无界任务队列，线程池的任务队列可以无限制的添加新的任务，而线程池创建的最大线程数量就是你`corePoolSize`设置的数量，也就是说在这种情况下`maximumPoolSize`这个参数是无效的，哪怕你的任务队列中缓存了很多未执行的任务，当线程池的线程数达到`corePoolSize`后，就不会再增加了；若后续有新的任务加入，则直接进入队列等待，当使用这种任务队列模式时，一定要注意你任务提交与处理之间的协调与控制，不然会出现队列中的任务由于无法及时处理导致一直增长，直到最后资源耗尽的问题。

### PriorityBlockingQueue

4**、优先任务队列：**优先任务队列通过`PriorityBlockingQueue`实现：

任务会按优先级重新排列执行，且线程池的线程数一直为`corePoolSize`，也就是只有一个。

`PriorityBlockingQueue`其实是一个特殊的无界队列，它其中无论添加了多少个任务，线程池创建的线程数也不会超过`corePoolSize`的数量，只不过其他队列一般是按照先进先出的规则处理任务，而`PriorityBlockingQueue`队列可以自定义规则根据任务的优先级顺序先后执行。

其实`LinkedBlockingQueue`也是可以设置界限的，它默认的界限是`Integer.MAX_VALUE`。同时也支持也支持构造的时候设置队列大小。

## 拒绝策略

```java
public interface RejectedExecutionHandler {
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```

当`Executor`已经关闭，即执行了`executorService.shutdown()`方法后，或者`Executor`将有限边界用于最大线程和工作队列容量，且已经饱和时。使用方法`execute()`提交的新任务将被拒绝.
在以上述情况下，`execute`方法将调用其`RejectedExecutionHandler`的 `RejectedExecutionHandler.rejectedExecution(java.lang.Runnable, java.util.concurrent.ThreadPoolExecutor) `方法。

### AbortPolicy 默认的拒绝策略

也称为终止策略，遭到拒绝将抛出运行时`RejectedExecutionException`。业务方能通过捕获异常及时得到对本次任务提交的结果反馈。

```java
public static class AbortPolicy implements RejectedExecutionHandler {
  public AbortPolicy() { }
  public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    throw new RejectedExecutionException("Task " + r.toString() +
                                         " rejected from " +
                                         e.toString());
  }
}
```

### CallerRunsPolicy

拥有自主反馈控制，让提交者执行提交任务，能够减缓新任务的提交速度。这种情况是需要让所有的任务都执行完毕。

```java
public static class CallerRunsPolicy implements RejectedExecutionHandler {
    public CallerRunsPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            r.run();
        }
    }
}
```

### DiscardPolicy

拒绝任务的处理程序，静默丢弃任务。使用此策略，我们可能无法感知系统的异常状态。慎用~！

```java
public static class DiscardPolicy implements RejectedExecutionHandler {
    public DiscardPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}
```

### DiscardOldestPolicy 

丢弃队列中最前面的任务，然后重新提交被拒绝的任务。是否要使用此策略需要看业务是否需要新老的替换，慎用~！

```java
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
    public DiscardOldestPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            e.getQueue().poll();
            e.execute(r);
        }
    }
}
```

# 内核

前面讲了**线程池的外观**，接下来讲述它的**内核**。

线程池在内部实际上构建了一个**生产者消费者模型**，将`线程`和`任务`两者解耦，并不直接关联，从而良好的缓冲任务，复用线程。

线程池的运行主要分成两部分：任务管理、线程管理。

任务管理部分充当生产者的角色，当任务提交后，线程池会判断该任务后续的流转：

1. 直接申请线程执行该任务；
2. 缓冲到队列中等待线程执行；
3. 拒绝该任务。

线程管理部分是消费者，它们被统一维护在线程池内，根据任务请求进行线程的分配，当线程执行完任务后则会继续获取新的任务去执行，最终当线程获取不到任务的时候，线程就会被回收。

接下来，我们会按照以下三个部分去详细讲解线程池运行机制：

1. 线程池如何维护自身状态。
2. 线程池如何管理任务。
3. 线程池如何管理线程。

## 线程池的生命周期

线程池运行的状态，并不是用户显式设置的，而是伴随着线程池的运行，由内部来维护。

线程池内部使用一个变量维护两个值：运行状态(`runState`)和线程数量 (`workerCount`)。

在具体实现中，线程池将运行状态(`runState`)、线程数量 (`workerCount`)两个关键参数的维护放在了一起:

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

`ctl`这个`AtomicInteger`类型，是对线程池的运行状态和线程池中有效线程的数量进行控制的一个字段.

它同时包含两部分的信息：线程池的运行状态 (`runState`) 和线程池内有效线程的数量 (`workerCount`)，高**3位**保存`runState`，低**29**位保存`workerCount`，两个变量之间互不干扰。

> 用一个变量去存储两个值，可避免在做相关决策时，出现不一致的情况，不必为了维护两者的一致，而占用锁资源。通过阅读线程池源代码也可以发现，经常出现要同时判断线程池运行状态和线程数量的情况。线程池也提供了若干方法去供用户获得线程池当前的运行状态、线程个数。这里都使用的是位运算的方式，相比于基本运算，速度也会快很多(PS:这种用法在许多源代码中都可以看到)。

关于内部封装的获取生命周期状态、获取线程池线程数量的计算方法如以下代码所示：

```java
private static final int COUNT_BITS = Integer.SIZE - 3;//32-3
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;//低29位都为1，高位都为0

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;//111
private static final int SHUTDOWN   =  0 << COUNT_BITS;//000
private static final int STOP       =  1 << COUNT_BITS;//001
private static final int TIDYING    =  2 << COUNT_BITS;//010
private static final int TERMINATED =  3 << COUNT_BITS;//011

// Packing and unpacking ctl
//计算当前运行状态，取高三位
private static int runStateOf(int c)     { return c & ~CAPACITY; }
//计算当前线程数量，取低29位
private static int workerCountOf(int c)  { return c & CAPACITY; }
//通过状态和线程数生成ctl
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

`ThreadPoolExecutor`的运行状态有5种，分别为：

|  运行状态  | 状态描述                                                     |
| :--------: | ------------------------------------------------------------ |
|  RUNNING   | 能接受新提交的任务，并且也能处理阻塞队列中的任务             |
|  SHUTDOWN  | 不能接受新提交的任务，但却可以继续处理阻塞队列中的任务       |
|    STOP    | 不能接受新任务，也不能处理队列中的任务同时会中断正在处理的任务线程 |
|  TIDYING   | 所有的任务都已经终止，workCount（有效线程数）为0             |
| TERMINATED | 在terminated方法执行完之后进入该状态                         |

![线程池声明周期.jpg](https://img-blog.csdnimg.cn/img_convert/04cb57348838a44aa491d0d300a720a4.png#pic_center)


## 任务调度机制

任务调度是线程池的主要入口，当用户提交了一个任务，接下来这个任务将如何执行都是由这个阶段决定的。了解这部分就相当于了解了线程池的核心运行机制。

首先，所有任务的调度都是由`execute`方法完成的，这部分完成的工作是：检查现在线程池的**运行状态**、**运行线程数**、**运行策略**，决定接下来执行的流程，是直接**申请线程执行**，或是**缓冲到队列中执行**，亦或是**直接拒绝该任务**。其执行过程如下：

1. 首先检测线程池运行状态，如果不是`RUNNING`，则直接拒绝，线程池要保证在`RUNNING`的状态下执行任务。
2. 如果`workerCount < corePoolSize`，则创建并启动一个线程来执行新提交的任务。
3. 如果`workerCount >= corePoolSize`，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中。
4. 如果`workerCount >= corePoolSize && workerCount < maximumPoolSize`，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务。
5. 如果`workerCount >= maximumPoolSize`，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。

![任务调度流程图.png](https://img-blog.csdnimg.cn/img_convert/7ec699190187108a2d80121090d2f4ef.png#pic_center)


接下来进入源代码分析时间~！

## 提交任务

```java
//AbstractExecutorService.java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
//ThreadPoolExecutor.java
public void execute(Runnable command) {
  if (command == null)
    throw new NullPointerException();
  int c = ctl.get();//获取ctl
  //检查当前核心线程数，是否小于核心线程数的大小限制
  if (workerCountOf(c) < corePoolSize) {
    //没有达到核心线程数的大小限制，那么添家核心线程执行该任务
    if (addWorker(command, true))
      return;
    //如果添加失败，刷新ctl值
    c = ctl.get();
  }
  //再次检查线程池的运行状态，将任务添加到等待队列中
  if (isRunning(c) && workQueue.offer(command)) {
    int recheck = ctl.get();//刷新ctl值
    //如果当前线程池的装不是运行状态，那么移除刚才添加的任务
    if (! isRunning(recheck) && remove(command))
      reject(command);//移除成功后，使用拒绝策略处理该任务；
    else if (workerCountOf(recheck) == 0)//当前工作线程数为0
      //线程池正在运行，或者移除任务失败。
      //添加一个非核心线程，并不指定该线程的运行任务。
      //等线程创建完成之后，会从等待队列中获取任务执行。
      addWorker(null, false);
  } 
  //逻辑到这里说明线程池已经不是RUNNING状态，或者等待队列已满，需要创建一个新的非核心线程执行该任务；
  //如果创建失败，那么非核心线程已满，使用拒绝策略处理该任务；
  else if (!addWorker(command, false))
    reject(command);
}
```

## 添加工作线程和执行任务

```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
  Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;//初始化的任务，可以为null
    this.thread = getThreadFactory().newThread(this);//Worker持有的线程
  }
  /**部分代码省略*/
	public void run() {
  	runWorker(this);
	}
}
```

**添加工作线程和执行任务**：总体就是创建`Worker`，并且为它找到匹配的`Runnable`。

![addwork.png](https://img-blog.csdnimg.cn/img_convert/83f919cb70bf439b9f113f967f181b85.png#pic_center)


### 添加工作线程

增加线程是通过线程池中的`addWorker`方法，该方法的功能就是增加一个线程，该方法不考虑线程池是在哪个阶段增加的该线程，这个分配线程的策略是在上个步骤完成的，该步骤仅仅完成增加线程，并使它运行，最后返回是否成功这个结果。

`addWorker`方法有两个参数：`firstTask`、`core`。

`firstTask`参数用于指定新增的线程执行的第一个任务，该参数可以为空；

`core`参数为`true`表示在新增线程时会判断当前活动线程数是否少于`corePoolSize`，`false`表示新增线程前需要判断当前活动线程数是否少于`maximumPoolSize`。

![addwork2.png](https://img-blog.csdnimg.cn/img_convert/15bfac21e604754038a95ead5fd19e76.png#pic_center)


```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry://break和continue的跳出标签
    for (;;) {
        int c = ctl.get();//获取ctl的值
        int rs = runStateOf(c);//获取当前线程池的状态；
        /**
         * 1、如果当前的线程池状态不是RUNNING
         * 2、当前线程池是RUNNING而且没有添加新任务，而且等待队列不为空。这种情况下是需要创建执行线程的。
         * 所以满足1，但不满足2就创建执行线程失败，返回false。
         */
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
                firstTask == null &&
                ! workQueue.isEmpty()))
            return false;
        /**进入内层循环 */
        for (;;) {
            int wc = workerCountOf(c);//获取当前执行线程的数量
            /**
             * 1、工作线程数量大于或等于计数器的最大阈值，那么创建执行线程失败，返回false。
             * 2、如果当前创建的核心线程，那么工作线程数大于corePoolSize的话，创建执行线程失败，返回false。
             * 3、如果当前创建的是非核心线程，那么工作线程数大于maximumPoolSize的话，创建执行线程失败，返回false。
             */
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            //用CAS操作让线程数加1，如果成功跳出整个循环
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)//线程状态前后不一样，重新执行外循环
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
            //如果CAS操作由于工作线程数的增加失败，那么重新进行内循环
        }
    }
    /**就现在，线程数已经增加了。但是真正的线程对象还没有创建出来。*/
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();//加锁
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());
                /**
                 * 再次检查线程池的运行状态
                 * 1、如果是RUNNING状态，那么可以创建；
                 * 2、如果是SHUTDOWN状态，但没有执行线程，可以创建（创建后执行等待队列中的任务）
                 */
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    //检测该线程是否已经开启
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);//将改工作线程添加到集合中
                    int s = workers.size();
                    if (s > largestPoolSize)//更新线程池的运行时的最大线程数
                        largestPoolSize = s;
                    workerAdded = true;//标识工作线程添加成功
                }
            } finally {//释放锁
                mainLock.unlock();
            }
            if (workerAdded) {//如果工作线程添加成功，那么开启线程
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        //如果工作线程添加失败，那么进行失败处理
        //将已经增加的线程数减少，将添加到集合的工作线程删除
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

### 执行任务

在**添加工作线程**部分我们看到了，添加成功之后会开启线程执行任务。

![runwork.png](https://img-blog.csdnimg.cn/img_convert/c983921d315a5083bf356e2f8a67a09f.png#pic_center)


```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    //解锁，允许中断
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        //如果当前的工作线程已经有执行任务，或者可以从等待队列中获取到执行任务
        //getTask获取任务时候会进行阻塞
        while (task != null || (task = getTask()) != null) {
            w.lock();//开始执行，上锁
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            //判断线程是否需要中断
            //如果线程池状态是否为STOP\TIDYING\TERMINATED,同时当前线程没有被中断那么将当前线程进行中断
            if ((runStateAtLeast(ctl.get(), STOP) ||
                    (Thread.interrupted() &&
                    runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;//异常处理
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;//工作线程的当前任务置空
                w.completedTasks++;//当前工作线程执行完成的线程数+1
                w.unlock();//执行完成解锁
            }
        }
        completedAbruptly = false;//完成了所有任务，正常退出
    } finally {//执行工作线程的退出操作
        processWorkerExit(w, completedAbruptly);
    }
}
```

## 工作线程获取任务

![getTask.jpg](https://img-blog.csdnimg.cn/img_convert/15ab456774deee5a09f037c61f6da916.png#pic_center)


```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?
    for (;;) {
        int c = ctl.get();//获取ctl的值
        int rs = runStateOf(c);//获取线程池状态

        // Check if queue empty only if necessary.
        /**
         * 1、rs为STOP\TIDYING\TERMINATED，标识无法继续执行任务
         * 2、等待队列中没有任务可以被执行
         * 工作线程数量减一
         */
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
        int wc = workerCountOf(c);//获取工作线程数量
        // Are workers subject to culling?
        //如果允许核心线程超时，或者当前工作线程数量大于核心线程数量。标识需要进行超时检测
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        /**
         * 1、如果当前工作线程数是否大于线程池可允许的最大工作线程数（maximumPoolSize可以动态设置）
         * ，或者当前需要进行超时控制并且上次从等待队列中获取执行任务发生了超时。
         * 2、如果当前不是唯一的线程，并且等待队列中没有需要执行的任务。
         * 这两种情况下一起存在就表示，工作线程发生了超时需要回收，所以对线程数进行-1；
         */
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))//线程数量减少成功，否则重新执行本次循环
                return null;
            continue;
        }
        try {
            //如果设置有超时，那么设定超时时间。否则进行无限的阻塞等待执行任务
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;//获取超时,设置标记
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

## 工作线程的退出

线程池中线程的销毁依赖**JVM自动**的回收，线程池做的工作是根据当前线程池的状态维护一定数量的线程引用，防止这部分线程被JVM回收，当线程池决定哪些线程需要回收时，只需要将其引用消除即可。`Worker`被创建出来后，就会不断地进行轮询，然后获取任务去执行，核心线程可以无限等待获取任务，非核心线程要限时获取任务。当`Worker`无法获取到任务，也就是获取的任务为空时，循环会结束，`Worker`会主动消除自身在线程池内的引用。

![processWorkerExit.png](https://img-blog.csdnimg.cn/img_convert/2f2a67e167362a69c5ae28e20fa6af88.png#pic_center)


```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    //completedAbruptly为true，标识该工作线程执行出现了异常，将工作线程数减一
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();
    //否则标识该工作线程为正常结束，这种情况下getTask方法中已经对工作线程进行了减一
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();//加锁
    try {
        completedTaskCount += w.completedTasks;//更新线程池的，线程执行完成数量
        workers.remove(w);//工作线程容器移除该工作线程
    } finally {
        mainLock.unlock();//解锁
    }
    //尝试结束线程池
    tryTerminate();
    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {//如果当前线程池的运行状态是RUNNING\SHUTDOWN
        if (!completedAbruptly) {//如果该工作线程为正常结束
            /**
             * 判断当前需要的最少的核心线程数(如果允许核心线程超时，那么最小的核心线程数为0，否则为corePoolSize)
             */
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            //如果允许核心线程超时，而且等待队列不为空，那么工作线程的最小值为1，否则为0。
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            //当前工作线程数，是否满足最先的核心线程数
            if (workerCountOf(c) >= min)
                //如果满足那么直接return
                return; // replacement not needed
        }
        //如果是异常结束，或者当前线程数不满足最小的核心线程数，那么添加一个非核心线程
        //核心线程和非核心线程没有什么不同，只是在创建的时候判断逻辑不同
        addWorker(null, false);
    }
}
```

# 特需

## 线程池的监控

通过线程池提供的参数进行监控。线程池里有一些属性在监控线程池的时候可以使用

- `getTaskCount`：线程池已经执行的和未执行的任务总数；
- `getCompletedTaskCount`：线程池已完成的任务数量，该值小于等于`taskCount`；
- `getLargestPoolSize`：线程池曾经创建过的最大线程数量。通过这个数据可以知道线程池是否满过，也就是达到了`maximumPoolSize`；
- `getPoolSize`：线程池当前的线程数量；
- `getActiveCount`：当前线程池中正在执行任务的线程数量。

## 动态调整线程池的大小

`JDK`允许线程池使用方通过`ThreadPoolExecutor`的实例来动态设置线程池的核心策略，以`setCorePoolSize`为方法例;

> 在运行期线程池使用方调用此方法设置`corePoolSize`之后，线程池会直接覆盖原来的`corePoolSize`值，并且基于当前值和原始值的比较结果采取不同的处理策略。
>
> 对于当前值小于当前工作线程数的情况，说明有多余的`worker`线程，此时会向当前`idle`的`worker`线程发起中断请求以实现回收，多余的`worker`在下次`idel`的时候也会被回收；对于当前值大于原始值且当前队列中有待执行任务，则线程池会创建新的`worker`线程来执行队列任务（PS:`idel`状态为`worker`线程释放锁之后的状态，因为它在运行期间都是上锁的）。

![setCorePoolSize.png](https://img-blog.csdnimg.cn/img_convert/a37f0ff31615cf75d0a67061b26429d7.png#pic_center)


```java
public void setCorePoolSize(int corePoolSize) {
    if (corePoolSize < 0)
        throw new IllegalArgumentException();
    //计算增量
    int delta = corePoolSize - this.corePoolSize;
    //覆盖原有的corePoolSize
    this.corePoolSize = corePoolSize;
    //如果当前的工作线程数量大于线程池的最大可运行核心线程数量，那么进行中断工作线程处理
    if (workerCountOf(ctl.get()) > corePoolSize)
        interruptIdleWorkers();
    else if (delta > 0) {//如果增量大于0
        // We don't really know how many new threads are "needed".
        // As a heuristic, prestart enough new workers (up to new
        // core size) to handle the current number of tasks in
        // queue, but stop if queue becomes empty while doing so.
        //等待队列非空，获取等待任务和增量的最小值
        int k = Math.min(delta, workQueue.size());
        //循环创建核心工作线程执行等待队列中的任务
        while (k-- > 0 && addWorker(null, true)) {
            if (workQueue.isEmpty())
                break;
        }
    }
}
private void interruptIdleWorkers() {
    interruptIdleWorkers(false);
}
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();//加锁
    try {
        //遍历工作线程的集合
        for (Worker w : workers) {
            Thread t = w.thread;
            //如果当前线程没有被中断，而且能获取到锁，那么尝试进行中断，最后释放锁
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            //是否仅仅中断一个工作线程
            if (onlyOne)
                break;
        }
    } finally {//释放锁
        mainLock.unlock();
    }
}
```

## 优雅的关闭线程池

从《线程池声明周期》图上还可以看到，当我们执行 `ThreadPoolExecutor#shutdown` 方法将会使线程池状态从 **RUNNING** 转变为 **SHUTDOWN**。而调用 `ThreadPoolExecutor#shutdownNow` 之后线程池状态将会从 **RUNNING** 转变为 **STOP**。

### shutdown

> 停止接收新任务，原来的任务继续执行

1. 停止接收新的submit的任务；
2. 已经提交的任务（包括正在跑的和队列中等待的）,会继续执行完成；
3. 等到第2步完成后，才真正停止；

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();// 检查权限
        advanceRunState(SHUTDOWN);// 设置线程池状态
        interruptIdleWorkers();// 中断空闲线程
        // 钩子函数，主要用于清理一些资源
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
```

`shutdown` 方法首先加锁，其次先检查系统安装状态。接着就会将线程池状态变为 **SHUTDOWN**，在这之后线程池不再接受提交的新任务。此时如果还继续往线程池提交任务，将会使用线程池拒绝策略响应，默认情况下将会使用 `ThreadPoolExecutor.AbortPolicy`，抛出 `RejectedExecutionException` 异常。

`interruptIdleWorkers` 方法在[动态调整线程池大小]()部分有源码讲述，它只会中断空闲的线程，不会中断正在执行任务的的线程。空闲的线程将会阻塞在线程池的阻塞队列上。

### shutdownNow

> 停止接收新任务，原来的任务停止执行

1. 跟 `shutdown() `一样，先停止接收新`submit`的任务；
2. 忽略队列里等待的任务；
3. 尝试将正在执行的任务`interrupt`中断；
4. 返回未执行的任务列表；

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();// 检查状态
        advanceRunState(STOP);// 将线程池状态变为 STOP
        interruptWorkers();// 中断所有线程，包括工作线程以及空闲线程
        tasks = drainQueue();// 丢弃工作队列中存量任务
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers)
            //如果工作线程已经开始，那么调用interrupt进行中断
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
}
private List<Runnable> drainQueue() {
    BlockingQueue<Runnable> q = workQueue;
    ArrayList<Runnable> taskList = new ArrayList<Runnable>();
    //从此队列中删除所有可用的元素，并将它们添加到给定的集合中。
    q.drainTo(taskList);
    //如果队列是DelayQueue或其他类型的队列，而poll或drainTo可能无法删除某些元素，则会将它们逐个删除。
    if (!q.isEmpty()) {
        for (Runnable r : q.toArray(new Runnable[0])) {
            if (q.remove(r))
                taskList.add(r);
        }
    }
    return taskList;
}
```

`shutdownNow`试图终止线程的方法是通过调用 `Thread.interrupt()` 方法来实现的，这种方法的作用有限，如果线程中没有`sleep` 、`wait`、`Condition`、定时锁等应用, `interrupt()` 方法是无法中断当前的线程的。所以，`shutdownNow() `并不代表线程池就一定立即就能退出，它也可能必须要等待所有正在执行的任务都执行完成了才能退出。但是大多数时候是能立即退出的。

> 线程中断机制： `thread#interrupt` 只是设置一个中断标志，不会立即中断正常的线程。如果想让中断立即生效，必须在线程 内调用 `Thread.interrupted()` 判断线程的中断状态。 对于阻塞的线程，调用中断时，线程将会立刻退出阻塞状态并抛出 `InterruptedException` 异常。所以对于阻塞线程需要正确处理 `InterruptedException` 异常。

### awaitTermination

线程池 `shutdown` 与 `shutdownNow` 方法都不会主动等待执行任务的结束，如果需要等到线程池任务执行结束，需要调用 `awaitTermination` 主动等待任务调用结束。

- 等所有已提交的任务（包括正在跑的和队列中等待的）执行完；
- 等超时时间到了；
- 线程被中断，抛出`InterruptedException`;

如果线程池任务执行结束，`awaitTermination` 方法将会返回 `true`，否则当等待时间超过指定时间后将会返回 `false`。

```java
// 关闭线程池的钩子函数
private static void shutdown(ExecutorService executorService) {
    // 第一步：使新任务无法提交
    executorService.shutdown();
    try {
        // 第二步：等待未完成任务结束
        if(!executorService.awaitTermination(60, TimeUnit.SECONDS)) {
             // 第三步：取消当前执行的任务
            executorService.shutdownNow();
            // 第四步：等待任务取消的响应
            if(!executorService.awaitTermination(60, TimeUnit.SECONDS)) {
                System.err.println("Thread pool did not terminate");
            }
        }
    } catch(InterruptedException ie) {
        // 第五步：出现异常后，重新取消当前执行的任务
        executorService.shutdownNow();
        Thread.currentThread().interrupt(); // 设置本线程中断状态
    }
}
```

# 其他

感觉内容好多，写不完啊~！

说的线程池就不得不说多线程并发操作，`同步`，`异步`，`CSA`，`AQS`，`公平锁和非公平锁`，`可重入锁和非可重入锁`等各种并发控制需要的知识点。

平常工作中使用比较少，自己有没有系统的知识体系结构。导致好多学过之后忘掉，然后又学习又忘记。

希望我以后有机会能逐步进行学习和分享。

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：
![振兴书城](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzEzMTk4NzktNjEyYzRjNjZkNDBjZTg1NS5qcGc?x-oss-process=image/format,png#pic_center)