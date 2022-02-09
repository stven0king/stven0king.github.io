---
title: 又一年对Android消息机制（Handler和Looper）的思考
date: 2017-6-25 22:39:00
tags: [Android消息机制,Handler,Looper]
categories: Android 
description: "Android消息机制对于每一个Android开发者来说都不陌生，在日常的开发中我们不可避免的要经常涉及这部分的内容。从开发角度来说，Handler是Android消息机制的上层接口，这使得在开发过程中只需要和Handler交互即可。Handler的使用过程很简单，通过它可以轻松的将一个任务切换Handler所在的线程中去执行。很多人认为Handler的作用是更新UI，这的确没错，但是更新UI仅仅是Handler的一个特殊的使用场景。具体来说是这样的；有时候需要再子线程中进行耗时的I/O操作，可能是读取文件或访问网络等。。。。。"
---


# 前言
Android消息机制对于每一个Android开发者来说都不陌生，在日常的开发中我们不可避免的要经常涉及这部分的内容。从开发角度来说，Handler是Android消息机制的上层接口，这使得在开发过程中只需要和Handler交互即可。Handler的使用过程很简单，通过它可以轻松的将一个任务切换Handler所在的线程中去执行。很多人认为Handler的作用是更新UI，这的确没错，但是更新UI仅仅是Handler的一个特殊的使用场景。具体来说是这样的；有时候需要再子线程中进行耗时的I/O操作，可能是读取文件或访问网络等。。。。。

本文是是作者在【温故而知新】时，发现自己之前做的笔记再次完善后的，希望对各位读者都有所帮助~！

<center>

![handler.jpg](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzEzMTk4NzktYzk2YzAyMDk1OTUzNTI0Ni5qcGc?x-oss-process=image/format,png#pic_center)
</center>
思考上边这幅图，让我们带着自己的期待，开始从Handler的使用一步步探究到源代码的愉快之旅。

# Handler
## Handler的对象的创建
```java
private Handler mHandler = new Handler(){
	@Override
	public void handleMessage(Message msg) {
		super.handleMessage(msg);
	}
};
```

## 利用Handler发送消息
```java
Message message = Message.obtain();
message.setData(bundle);
message.setTarget(mHandler);
message.sendToTarget();
//或者mHandler.handleMessage(message);
```
Handler的使用基本上就是上边所展示的【创建实例、处理消息、发送消息】这些内容。为什么这么简单的使用就能实现线程间的通信呢。。。

## Handler创建
上面我们介绍了Handler的使用，并且带着疑问【Handler使用的每一行代码我们都知道他的具体作用以及异步线程中是怎么传递消息的么】？

我们先从Handler的构造函数开始
```java

public Handler() {
    this(null, false);
}
public Handler(Callback callback) {
    this(callback, false);
}
public Handler(Looper looper) {
    this(looper, null, false);
}
public Handler(Looper looper, Callback callback) {
    this(looper, callback, false);
}
public Handler(boolean async) {
    this(null, async);
}
public Handler(Callback callback, boolean async) {
    //检测handler是否是静态的，否则会造成内存泄露；
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }
    //获取当前线程的Looper否则抛异常
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
public interface Callback {
    public boolean handleMessage(Message msg);
}
/**
 * looper:处理消息的装置
 * callback：处理消息回调，为空则调用handler的handleMessage方法
 * async：是否开启消息同步设置
 */
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
我们可以通过构造函数可以看出来，`Handler` 的构造函数实际的作用就是实例化内部的 `Looper` 类型的成员变量 `mLooper` ，和 `Looper` 内部的 `MessageQueue` 类型的变量 `mQueue`。`Handler 通过构造函数也能设置发送**同步消息** 和 **异步消息** 。

## Handler发送消息
发送消息有两种一种是利用Message，一种是调用Handler的sendMessage方法。
```java
//Message.java
public void sendToTarget() {
	target.sendMessage(this);
}
```
但是很显然第一种方法内部还是调用Handler#sendMessage方法，接下来我们看看该方法的具体实现。
```java
//Handler.java
public final boolean sendMessage(Message msg){
	return sendMessageDelayed(msg, 0);
}
public final boolean sendMessageDelayed(Message msg, long delayMillis){
	if (delayMillis < 0) {
		delayMillis = 0;
	}
	return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
//在这个时候delay已经变成when，时间有延迟变为执行
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
	MessageQueue queue = mQueue;
	if (queue == null) {
		RuntimeException e = new RuntimeException(
				this + " sendMessageAtTime() called with no mQueue");
		Log.w("Looper", e.getMessage(), e);
		return false;
	}
	return enqueueMessage(queue, msg, uptimeMillis);
}
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    //将要发送的Message绑上该Handler的标签
	msg.target = this;
    //如果构造设置了async，那么会为所有的消息打上异步的标签
	if (mAsynchronous) {
		msg.setAsynchronous(true);
	}
	//将消息将入Handler构造函数中初始化的MessageQueue队列
	return queue.enqueueMessage(msg, uptimeMillis);
}
```

调用用 `Handler#sendMessage` 方法实际上是一步一步最终将消息与该 `Handler` 绑定，并放入 `Handler` 构造函数中初始化的 `MessageQueue` 队列中。消息队列有进就有出，`MessageQueue` 不断接受 `Handler` 发送过来的消息，那么 `MessageQueue` 是怎么处理这些消息的呢？因为`Handler` 是内的消息队列是引用的 `Looper` 中的成员变量，而消息的循环接受处理正式 `Looper` 的 `loop` 方法，接下来我们详细讲解并分析`Looper` 类在 `Android的消息机制` 中的所扮演的角色！

## 同步屏障
在上面的讲述过程中我们了解到 `Handler` 可以发送 **同步消息** 和 **异步消息** ，那么两种消息有什么不一样呢，又有什么样的作用呢？

我们首先看看**同步屏障** 是怎么产生的：

```java
//MessageQueue.java
/**
 * Posts a synchronization barrier to the Looper's message queue.
 * @hide
 */
//发送一个同步的屏障到Looper的消息队列中
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}
/**
 * when:消息的执行时间
 */
int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        //msg的target为空
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;
        Message prev = null;
        Message p = mMessages;
        //找到一个执行时间比msg更晚或相同同时执行的消息
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        //如果有那么插入到这个消息的前面，否则这个消息就是连表头部的消息
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

- 同步消息：Message的setAsynchronous(true)
- 异步消息：Message的setAsynchronous(false)
- 同步屏障：Message的target=null

任何工作都需要有始有终，看完**同步屏障**的产生之后，可能接下来我们会思考**同步屏障**怎么移除。
虽然不同屏障是以同步消息的格式添加到消息队列中的，但是它不会像普通的消息一样被消费。一般都是谁插入的屏障谁负责移除。

```java
/**
 * when:消息的执行时间
 */
public void removeSyncBarrier(int token) {
    // Remove a sync barrier token from the queue.
    // If the queue is no longer stalled by a barrier then wake it.
    synchronized (this) {
        Message prev = null;
        //链表的头结点
        Message p = mMessages;
        //从头结点开始遍历，找到target=null，而且token相等的Message
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        //指定的消息队列同步屏障令牌尚未发布或已被删除。
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                    + " barrier token has not been posted or has already been removed.");
        }
        final boolean needWake;
        //如果获取的屏障不是头结点，那么删除该结点，此时不需要唤醒
        if (prev != null) {
            prev.next = p.next;
            needWake = false;
        } else {
            //获取的屏障是头结点，而且下一个结点不是屏障那么就需要唤醒
            mMessages = p.next;
            needWake = mMessages == null || mMessages.target != null;
        }
        //回收这个消息
        p.recycleUnchecked();
        // If the loop is quitting then it is already awake.
        // We can assume mPtr != 0 when mQuitting is false.
        // 如果需要唤醒而且没有退出消息的循环，那么执行唤醒
        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
}
```

这里先埋一个点，**同步消息** 、 **异步消息** 和 **同步屏障** 有什么用后面 `Looper` 中讲解。

# Looper
## Lopper简介
Looper在Android的消息机制中扮演着消息循环的角色，具体来说就是它会不停地从MessageQueue中查看是否有新消息，如果有新消息就会立即处理，否则就一直阻塞在哪里。
## Looper的构造函数
```java
private Looper(boolean quitAllowed) {
	mQueue = new MessageQueue(quitAllowed);
	mThread = Thread.currentThread();
}
```
通过Looper的构造函数我们可以看出，Looper每个实例中都会有一个MessageQueue队列，用来循环读取消息。


## Looper的主要成员变量
```java
// sThreadLocal.get() will return null unless you've called prepare().
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
private static Looper sMainLooper;  // guarded by Looper.class

final MessageQueue mQueue;
final Thread mThread;
```

> * sThreadLocal是一个ThreadLocal类型的变量，使用时可以得到当前线程的Looper实例。
> * sMainLooper表示主线程的Looper实例。
> * mQueue是Looper实例中的消息队列。
> * mThread线程类型的对象。

`ThreadLocal` 相关知识可以阅读 [Android与Java中的ThreadLocal](https://dandanlove.blog.csdn.net/article/details/50791483) 。

通过Looper的成员变量我们可以看出来，每个线程只能有一个Loope实例。Looper是通过mQueue不断循环接受消息、分发消息的。

## Looper的获取
为什么这里没有写Looper的创建而是Looper的获取呢？Looper的构造方法能为我们创建可用Looper实例么？仔细看上面的Looper的构造函数我们可以发现，构造函数中没有将Looper实例放入sThreadLocal中。我们看看下边 ```prepare() ```和 ```myLooper() ```方法的实现。
```java
//Looper.java
public static void prepare() {
	prepare(true);
}
private static void prepare(boolean quitAllowed) {
	if (sThreadLocal.get() != null) {
		throw new RuntimeException("Only one Looper may be created per thread");
	}
     //Looper类会创建一个新的Looper对象,并放入全局的sThreadLocal中。
	sThreadLocal.set(new Looper(quitAllowed));
}
public static Looper myLooper() {
	return sThreadLocal.get();
}
public static void prepareMainLooper() {
	prepare(false);
	synchronized (Looper.class) {
		if (sMainLooper != null) {
			throw new IllegalStateException("The main Looper has already been prepared.");
		}
		sMainLooper = myLooper();
	}
}
public static Looper getMainLooper() {
	synchronized (Looper.class) {
		return sMainLooper;
	}
}
```

在启动Activity的时候sMainLooper已经开始工作了（具体请看下边ActivityThread的main方法），所以我们要获取主线程的Looper实例，只需要调用```Looper.getMainLooper ```。但在其他异步线程中我们要创建Looper通常我们通过以下方式获取Looper对象。

```java
Looper.prepare();
Looper looper = Looper.myLooper();
```

接下来我们看看在Framwork层中Looper是怎么使用的：ActivityThread#main

```java
public static void main(String[] args) {
	Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
	SamplingProfilerIntegration.start();

	// CloseGuard defaults to true and can be quite spammy.  We
	// disable it here, but selectively enable it later (via
	// StrictMode) on debug builds, but using DropBox, not logs.
	CloseGuard.setEnabled(false);

	Environment.initForCurrentUser();

	// Set the reporter for event logging in libcore
	EventLogger.setReporter(new EventLoggingReporter());

	// Make sure TrustedCertificateStore looks in the right place for CA certificates
	final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
	TrustedCertificateStore.setDefaultUserDirectory(configDir);

	Process.setArgV0("<pre-initialized>");
    //创建主线程的Looper对象
	Looper.prepareMainLooper();

	ActivityThread thread = new ActivityThread();
	thread.attach(false);

	if (sMainThreadHandler == null) {
		sMainThreadHandler = thread.getHandler();
	}

	if (false) {
		Looper.myLooper().setMessageLogging(new
				LogPrinter(Log.DEBUG, "ActivityThread"));
	}

	// End of event ActivityThreadMain.
	Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
	//开启循坏队列监听并接受&处理消息
	Looper.loop();

	throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

我们可以看见main方法执行的时候主线程中的Looper就已经创建并循环等待处理消息了。

## Looper的loop
上面我们讲述了Looper对象实例的获取，那么获取完后我们就需要开启循坏队列监听并接受&处理消息，即执行```Looper.loop() ```方法。

```java
/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */
public static void loop() {
    //获取当前线程的Looper对象实例
    final Looper me = myLooper();
    //调用loop()方法之前必须在当前线程通过Looper.prepare()方法创建Looper对象实例。
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    //获取Looper实例的中的消息队列
    final MessageQueue queue = me.mQueue;
    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();
    /***部分代码省略***/
    //开启循环队列-不断
    for (;;) {
        //从队列中取出消息，有可能产生阻塞
        Message msg = queue.next(); // might block
        //如果获取不到消息那么退出
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        /***部分代码省略***/
        //将消息分发给注册它的handler
        try {
            msg.target.dispatchMessage(msg);
            dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
        /***部分代码省略***/
        msg.recycleUnchecked();
    }
}
```

到这里我们终于找到消息发送后的处理逻辑，实际上是通过每条消息传递给该消息所绑定的Handler的dispatchMessgage方法进行处理。

## Handler的消息处理
```java
//Handler.java
public void dispatchMessage(Message msg) {
	if (msg.callback != null) {
		handleCallback(msg);
	} else {
		if (mCallback != null) {
			if (mCallback.handleMessage(msg)) {
				return;
			}
		}
		handleMessage(msg);
	}
}
```
这个就是为什么我们在创建Handler对象的时候都需要复写它的handleMessage方法。

## MessageQueue.next
从 `MessageQueue ` 中获取消息，这个方法可能产生阻塞；

```java
//MessageQueue.java
@SuppressWarnings("unused")
private long mPtr; // used by native code
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    //如果消息循环已经退出并被处理，请返回此处。 
    //如果应用程序退出后不尝试重新启动循环程序，则可能会发生这种情况。
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }
    //挂起空闲处理程序计数，一次next方法只设置一次；
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    //不断循环直到获取到Message为止；
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        //唤醒消息队列；
        //nextPollTimeoutMillis=-1,表示等待；
        //nextPollTimeoutMillis=0,表示唤醒；
        ////nextPollTimeoutMillis>0,表示nextPollTimeoutMillis时间之后唤醒；
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            //如果遇到同步屏障，那么找到离头部最近的异步消息；
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                //如果消息不为空，而且在当前时间之后执行那么设置下次唤醒时间为Math.min(msg.when - now, Integer.MAX_VALUE)；
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    //如果消息现在执行，那么消息队列设置不需要阻塞，并且设置改消息为已使用并在链表中删除；
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                //获取不到消息那么nextPollTimeoutMillis = -1，将会进行空闲等待；
                nextPollTimeoutMillis = -1;
            }
            //已经处理当前获取的消息是否进行退出，调用了MessageQueue的quit方法；
            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            //如果是第一次空闲，则获取发现线程何时阻塞的回调数量；
            //挂起空闲处理程序仅在队列为空或将来要处理队列中的第一条消息（可能是同步屏障）时才运行；
            //mIdleHandlers是添加了等待空闲的程序回调接口的列表；
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            //没有挂起空闲处理程序；
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }
            //等待空闲的程序回调接口的数组如果为空，那么创建新的数组；
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }
        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        //运行等待空闲时间的回调；
        //我们只会在第一次迭代时到达此代码块,因为后续的pendingIdleHandlerCount会置为0；
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            //释放mPendingIdleHandlers的引用对象；
            mPendingIdleHandlers[i] = null; // release the reference to the handler
            boolean keep = false;
            try {
                //进行接口回调，返回是否该回调要继续保持；
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            //如果不需要保持那么删除这个等待空闲程序；
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }
        //将空闲处理程序计数重置为0，这样我们本地next方法就不再运行它们；
        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;
        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        //在调用空闲处理程序时，可能已经传递了一条新消息；
        //因此，请返回并再次查找未决的消息，而无需等待；
        nextPollTimeoutMillis = 0;
    }
}
```

1. `nextPollTimeoutMillis=0` ，唤醒消息队列；
2. 获取当前链表中的消息头 `mMessages` ;
3. 如果头消息不为空，而且消息的 `target` 为空；
	1. 那么这个是`屏障消息`，遇到消息屏障会从链表的顶端往下找一个`异步消息` ；
		1. 如果找到异步消息那么这个异步消息将成为这次要处理的消息；
		2. 否则消息队列会进行等待，直到有异步消息到来。
4. 如果消息不为空，判断当前消息的执行时间是否比现在晚；
	1. 如果改消息的执行时间比现在晚，那么设置下次唤醒时间为当前消息的执行时间；
	2. 否则标记消息为已使用，并从链表当中删除该消息并进行返回，结束循环；
5. 如果消息为空那么 `nextPollTimeoutMillis=-1` ，那么消息队列进行等待；

## 同步屏障的使用
在 `MessageQueue.next` 这个小结当中我们看到了 `屏障消息` 的出现，他的作用是：**忽略所有的同步消息，返回异步消息。再换句话说，同步屏障为Handler消息机制增加了一种简单的优先级机制，异步消息的优先级要高于同步消息。**

同时我们也看到了 `MessageQueue.next` 的源代码中是不会删除同步 `屏障消息` 的，所以 **同步屏障** 出现后在不删除的情况下会一直保留。这个也解释为什么需要有删除**同步屏障**的消息方法。

之前在写 [ViewRootImpl的独白，我不是一个View(布局篇)](https://dandanlove.blog.csdn.net/article/details/78775166) 这篇文章的时候讲述过 `View绘制` 相关的知识点中就有**同步屏障** 的使用。

```java
//ViewRootImpl.java
//请求View开始进行布局
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        //检查是否是主线程让
        checkThread();
        mLayoutRequested = true;
        //进行绘制的任务队列
        scheduleTraversals();
    }
}
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //往handler的消息队列中插入同步屏障，并保存token
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        //添加异步消息任务mTraversalRunnable
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        //移除同步屏障
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        if (mProfile) {
            Debug.startMethodTracing("ViewAncestor");
        }
        //开始进行绘制
        performTraversals();
        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```

这里使用**同步屏障**可以是绘制的任务提高优先级，同时也在绘制的时候取消的**同步屏障**这样也就不影响其他消息的执行。

# 总结
看完整篇文章我们就能理解Android中的消息机制了吧？可能有些同学还是有些小疑惑，我貌似看到了并理解了Handler对消息的处理【Handler发送消息并添加到队列中，Looper循环将队列里的消息发给Handler处理】，但是好像对Handler是怎么实现多线程异步通信还有些不清楚？

## 多线程异步通信，Handler它到底是怎么实现的？

首先我们需要明确三点：
> * 线程之间内存是共享的；
> * 每个线程最多只能有一个Looper对象实例；
> * Handler创建的时候必须绑定Looper对象（默认为当前线程的Looper实例对象）；

确定以上三点，我们就容易理解多了。多个线程编程中，当多个线程只拥有一个Handler实例时，无论消息在哪个线程里发送出来，消息都会在这个Handler的handleMessage方法中做处理（举例：若在主线程中创建一个Handler对象并且该Handler对象绑定的是主线程的Looper对象实例，那么我们在异步线程中使用该handler发送的消息，最终都会将在主线程中进行处理。eg：网络请求，文件读写。。）。

现在我们终于可以清空我们所有的疑问愉快的结束本文章的阅读啦，若有其他需要交流的可以留言哦~！~！