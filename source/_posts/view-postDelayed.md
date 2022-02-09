---
title: View的postDelayed方法深度思考
date: 2019-10-25 16:42:53
tags: [postDelayed,pollOnce,Handler,Looper]
categories: Android
description: "突然某天好友老瑞问我 “View的postdelayed方法，延迟时间如果设置10分钟或者更长的时间有什么问题吗？“ 。当时听到这个问题时候我只能联想到 `Handle.postDelay`  ，与此同时让我回想起了之前的一些疑问？ "
---

# 前言

<center>
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191025162156513.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)
</center>

突然某天好友老瑞问我 `“View的postdelayed方法，延迟时间如果设置10分钟或者更长的时间有什么问题吗？“` 。当时听到这个问题时候我只能联想到 `Handle.postDelay`  ，与此同时让我回想起了之前的一些疑问？

> - View的postDelayed方法，延迟时间如果设置10分钟或者更长的时间有什么问题吗？
> - View的postDelayed方法，延迟时间如果设置为负数有没有问题？
> - View的postDelayed方法，是先delay还是先post？
> - View的postDelayed方法，有没有可能造成内存泄露？
> - Handle有没有可能造成内存泄露？
> - Looper的无线循环会使线程卡死么？

如果上面的问题大家知道答案，那么文章大家可以快速阅读。如果不是，那么我们脑海中可以带着问题跟着我的思路去一步步学习和理解相关的知识点，最后回过头来自己再回答这些问题。

网上搜索资料找到一篇 [[Handler.postDelayed()精确延迟指定时间的原理](http://www.dss886.com/2016/08/17/01/)] 文章，自己感觉从中学到很很多知识。本篇文章是我结合 `Android源码` 在此基础之上进行了思考和分析，文章内容也包含这篇资料所讲的内容。

关于 `Handler` 和 `Looper` 的大部分知识在以前的 [又一年对Android消息机制（Handler&Looper）的思考](https://dandanlove.blog.csdn.net/article/details/73730417)  文章讲的比较详细，这里讲的比较省略。

文章代码基于 `Android5.1`  进行分析。



# postDelayed源码分析

## View.postDelayed

> 导致将Runnable添加到消息队列中，并在经过指定的时间后运行。runnable将在用户界面线程上运行。

```java
//View.java
/**
 * <p>Causes the Runnable to be added to the message queue, to be run
 * after the specified amount of time elapses.
 * The runnable will be run on the user interface thread.</p>
 * *********
 */
public boolean postDelayed(Runnable action, long delayMillis) {
    final AttachInfo attachInfo = mAttachInfo;
    //如果mAttachInfo不为空，调用mAttachInfo的mHandler的post方法
    //即ViewRootHandler的postDelayed方法，ViewRootHandler继承Handler
    if (attachInfo != null) {
        return attachInfo.mHandler.postDelayed(action, delayMillis);
    }
    //假设会稍后成功，为什么假设？可以看方法前面的注释，后面也会讲。
  	//Assume that post will succeed later
    ViewRootImpl.getRunQueue().postDelayed(action, delayMillis);
    return true;
}
```

其中 `mAttachInfo` 是在 `dispatchAttachedToWindow` 方法中进行赋值的；


```java
//View.java
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    //System.out.println("Attached! " + this);
    mAttachInfo = info;
    /***部分代码省略***/
    onAttachedToWindow();
    /***部分代码省略***/
}
```

`View` 的 `dispatchAttachedToWindow` 方法是在 `ViewRootImpl.performTraversals`  第一次执行的时候调用的。相关知识可以看 [ViewRootImpl的独白，我不是一个View(布局篇)](https://dandanlove.blog.csdn.net/article/details/78775166) 这篇文章。 

上面通过判断 `mAttachInfo` 是否为空分为两种情况：

- 间接调用 `ViewRootHandler` 的 `postDelayed` 方法；
- 调用 `ViewRootImpl.RunQueue`  的 `postDelayed` 方法；

### ViewRootImpl.getRunQueue

> 当没有附加处理程序时，运行队列用于将未完成的工作从View中排队。 该工作在线程的下一次对performTraversals的调用期间执行。

```java
//ViewRootImpl.java
/**
 * The run queue is used to enqueue pending work from Views when no Handler is
 * attached.  The work is executed during the next call to performTraversals on
 * the thread.
 * @hide
 */
static final class RunQueue {
    private final ArrayList<HandlerAction> mActions = new ArrayList<HandlerAction>();
    void post(Runnable action) {
        postDelayed(action, 0);
    }
    //往队列中添加HandlerAction对象
    void postDelayed(Runnable action, long delayMillis) {
        HandlerAction handlerAction = new HandlerAction();
        handlerAction.action = action;
        handlerAction.delay = delayMillis;
        synchronized (mActions) {
            mActions.add(handlerAction);
        }
    }

    void removeCallbacks(Runnable action) {
        final HandlerAction handlerAction = new HandlerAction();
        handlerAction.action = action;
        //这里不考虑delay，所以HandlerAction的equals不考虑delay是否相等
        synchronized (mActions) {
            final ArrayList<HandlerAction> actions = mActions;
            //循环删除队列中的HandlerAction，如果该对象的Runnable与action相等
            while (actions.remove(handlerAction)) {
                // Keep going
            }
        }
    }
    //执行RunQueue队列中的HandlerAction，并清空mActions
    //执行调用的是handler.postDelayed，这里的handler也是mAttachInfo.mHandler
    void executeActions(Handler handler) {
        synchronized (mActions) {
            final ArrayList<HandlerAction> actions = mActions;
            final int count = actions.size();
            for (int i = 0; i < count; i++) {
                final HandlerAction handlerAction = actions.get(i);
                handler.postDelayed(handlerAction.action, handlerAction.delay);
            }
            actions.clear();
        }
    }
    //对Runnable和delay进行包装
    private static class HandlerAction {
        Runnable action;
        long delay;
        //判断两个对象是否相等，action相等则两个对象相等，不用考虑delay
        //为什么不用考虑，上面有讲
        @Override
        public boolean equals(Object o) {
            if (this == o) return true;
            if (o == null || getClass() != o.getClass()) return false;
            HandlerAction that = (HandlerAction) o;
            return !(action != null ? !action.equals(that.action) : that.action != null);

        }
        //这里不太清楚为什么这样重写hashCode方法
        @Override
        public int hashCode() {
            int result = action != null ? action.hashCode() : 0;
            result = 31 * result + (int) (delay ^ (delay >>> 32));
            return result;
        }
    }
}
```

-  `RunQueue.executeActions`   是在 `ViewRootImpl.performTraversal` 当中进行调用；

-  `RunQueue.executeActions`   是在执行完 `host.dispatchAttachedToWindow(mAttachInfo, 0);` 之后调用；

-  `RunQueue.executeActions`   是每次执行 `ViewRootImpl.performTraversal` 都会进行调用；
-  `RunQueue.executeActions`   的参数是 `mAttachInfo` 中的 `Handler` 也就是 `ViewRootHandler` ;

**这里我们得到的结论是 `ViewRootImpl.getRunQueue` 的 `postDelayed` 方法最终也是间接调用 `ViewRootHandler` 的 `postDelayed` 方法。**

#### RunQueue的小问题

```java
//ViewRootImpl.java 
static final ThreadLocal<RunQueue> sRunQueues = new ThreadLocal<RunQueue>();
static RunQueue getRunQueue() {
    //获取当前线程的RunQueue
    //其他线程：你不管我了么？
    RunQueue rq = sRunQueues.get();
    if (rq != null) {
        return rq;
    }
    rq = new RunQueue();
    sRunQueues.set(rq);
    return rq;
}
```

这个是 `ViewRootImpl` 中的 `geRunQueue` 实现，并且 `sRunQueues` 使用的是 `ThreadLocal` 。

`ThreadLocal` 相关知识可以阅读 [Android与Java中的ThreadLocal](https://dandanlove.blog.csdn.net/article/details/50791483) 。

也就是说这里**只能执行主线程**中  `postDelayed` 中的 `Runnable` 。

我用 `Android4.1.2` 设备在 `new Thread`  使用 `View.postDelayed` 的 `Runnable` 是不执行的， 但相同代码在 `Android8.0` 上是没有任何问题的。

```java
//android-28中的View.java
public boolean postDelayed(Runnable action, long delayMillis) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        return attachInfo.mHandler.postDelayed(action, delayMillis);
    }
		// 我们看到这里的注解也不一样了~！
    // Postpone the runnable until we know on which thread it needs to run.
    // Assume that the runnable will be successfully placed after attach.
    getRunQueue().postDelayed(action, delayMillis);
    return true;
}
private HandlerActionQueue mRunQueue;
private HandlerActionQueue getRunQueue() {
    if (mRunQueue == null) {
        mRunQueue = new HandlerActionQueue();
    }
    return mRunQueue;
}
```

其中 `android-28` 中不仅修改了之前的小问题， `HandlerActionQueue` 同样也做了些许优化。

### View.postDelayed小结

`postDelayed` 方法调用的时候如果当前的 `View` 没有依附在 `Window` 上的时候先将 `Runnable` 缓存在 `RunQueue` 队列中等到 `View.dispatchAttachedToWindow` 调用之后再被 `ViewRootHandler` 一次 `postDelayed` 这个过程中相同的 `Runnable` 只会被 `postDelay` 一次。

在当前的 `View` 依附在 `Window` 上的时候直接调用 `ViewRootHandler` 的 `postDelayed` 方法。

**`View.postDelayed` 最终都调用 `ViewRootHandler.postDelayed` 。**

## ViewRootHandler.postDelayed

```java
//ViewRootImpl.java
final class ViewRootHandler extends Handler {
    @Override
    public String getMessageName(Message message) {
        /***部分代码省略***/
        return super.getMessageName(message);
    }
    @Override
    public void handleMessage(Message msg) {
        /***部分代码省略***/
    }
}
```

`ViewRootImpl` 继承 `Handler` ，所以下边我们只需要分析 `Handler.postDelayed` 即可。 

[又一年对Android消息机制（Handler&Looper）的思考](https://dandanlove.blog.csdn.net/article/details/73730417) 这篇文章讲了一些 `Handler` 和 `Looper` 的基础知识。

<center>
![handler.jpg](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzEzMTk4NzktYzk2YzAyMDk1OTUzNTI0Ni5qcGc?x-oss-process=image/format,png#pic_center)
</center>

```java
//Handler.java
//构造Message消息
public final boolean postDelayed(Runnable r, long delayMillis) {
    return sendMessageDelayed(getPostMessage(r), delayMillis);
}
//获取缓存中的Message，绑定callback
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
//对Delay时间做判断，将delaytime改变为updatetime（延迟时间变为执行时间）
public final boolean sendMessageDelayed(Message msg, long delayMillis){
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
//对Queue队列做判空校验
//这里有一个思考，如果uptimeMillis小于当前时间或者小于0呢？
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
//给当前的Message绑定当前的Handler，如果handler构造为async那么将message设置为异步消息
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

上面是 `postDelay` 将一个 `Runnable` 添加到消息队列的方法调用路径。

将 `Runnable` 绑定到了一个 `Message` 上，这个 `Message` 也绑定了当前的 `Handler` 。

### MessageQueue.enqueueMessage

将`Message` 添加到 `Looper` 的消息队列：

```java
//MessageQueue.java
//标识是否next方法被poolOnce阻塞进行等待
// Indicates whether next() is blocked waiting in pollOnce() with a non-zero timeout.
private boolean mBlocked;
/**
 * msg:消息体
 * when：消息的执行时间=SystemClock.uptimeMillis() + delayMillis
 */
boolean enqueueMessage(Message msg, long when) {
    //校验message对应的handler是否被回收；
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    //校验消息是否有效；
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    //加锁机进行同步，以下代码并发执行会有逻辑错误；
    synchronized (this) {
        //当前消息队列是否执行了quit方法进行退出；
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w("MessageQueue", e.getMessage(), e);
            msg.recycle();
            return false;
        }
        //标记当前的message已经被使用过；
        msg.markInUse();
        msg.when = when;
        //将当前连表的头消息赋值给p；
        Message p = mMessages;
        //是否需要唤醒looper；
        boolean needWake;
        //如果头消息为空，或者新消息执行时间为0，或者头消息的执行时间比新消息的还晚；
        //那么新消息作为连表的消息头，如果被阻塞了也需要唤醒事件队列；
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            //将新的消息插入到队列中；
            //通常我们不必唤醒事件队列，除非队列的开头有同步障碍并且消息是队列中最早的异步消息
            //如果当前是阻塞状态奇，且队列开头是同步屏障的话，那么当改消息为异步消息的时候时候可能需要需要唤醒消息队列的
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            //找打一个比当前消息执行时间更晚的消息，插入到它的前面
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                //如果可能需要唤醒，但是在队列中找到其他异步消息。
                //那么不需要进行唤醒，因为当前的异步消息不会即可被执行
                //要执行也是它前面的异步消息p被执行
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }
        //如果需要唤醒
        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

上面这个过程讲述了将一个 `Message` 根据他的**执行时间从小到大**插入到链表中。除过 **同步屏障** 之外应该都不难理解，[又一年对Android消息机制（Handler&Looper）的思考](https://dandanlove.blog.csdn.net/article/details/73730417)  这一篇文章中我们讲了**同步屏障的产生、使用和移除** `MessageQueue.next` 方法。

### Handler.postDelayed小结

看到这里的话，文章开始提的问题几乎都能够找到答案了吧~！

>- View的postDelay方法，延迟时间如果设置10分钟或者更长的时间有什么问题吗？

**Delay时间是没有任何限制的，这个时间过程长只是使 post的Runnable的执行时间变长 。当前在这个过程中对应的Handler和Runnable是没有办法进行回收的，因为他们一直存储在消息队列中。**

>- View的postDelay方法，延迟时间如果设置为负数有没有问题？

**Delay的时间为负数也是没有问题，因为sendMessageDelayed方法会对Delay的时间做校验小于零的时候赋值为零。**

>- View的postDelay方法，是先delay还是先post？

**需要执行的Runnable是先被post到消息队列中的，然后延迟delay时间之后执行。**

>- View的postDelay方法，有没有可能造成内存泄露？

**只要post之后进入消息队列中的Message一直在链表中，那么相关对象的引用都不会被释放，所以这里会造成内存泄露**

>- Handle有没有可能造成内存泄露？

**View的postDelay最终调用的就是Handler的postDelay，根据上面问题的答案就知道Handler也会造成内存泄露。**

最后一个问题还没有解决，所以我们还需要继续往下阅读哦^_^



# Looper的无线循环会使线程卡死么？

[又一年对Android消息机制（Handler&Looper）的思考](https://dandanlove.blog.csdn.net/article/details/73730417)  这一篇文章中我们讲了 `Looper.loop` 方法和  `MessageQueue.next` 方法。在调用 `Looper.loop` 方法的过程中 `MessageQueue.next` 方法可能会产生阻塞，这个在源码的注释上都有。


![在这里插入图片描述](https://img-blog.csdnimg.cn/2019102516235598.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)

## MessageQueue.nativePollOnce和nativeWake

```java
//MessageQueue.java
private native void nativePollOnce(long ptr, int timeoutMillis); 
private native static void nativeWake(long ptr);
```

`nativePollOnce` 是一个 `C` 实现的方法，[从JNI_OnLoad看so的加载](https://dandanlove.blog.csdn.net/article/details/97624137) 这篇文章讲述了 `native方法的动态注册` 。

有睡眠就有唤醒，所以这里我们 `pollOnce` 和 `wake` 一起做分析。

### pollOnce和wake的native方法注册

```cpp
//AndroidRuntime.cpp
//启动AndroidRuntime运行时环境
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote);
//向VM注册Android的native方法
int AndroidRuntime::startReg(JNIEnv* env);
/**
 * gRegJNI，它是一个常量数组，里面是注册native方法的。
 * s这个方法循环遍历这个gRegJNI数组，依次调用数组中的方法
 */
static int register_jni_procs(const RegJNIRec array[], size_t count, JNIEnv* env){
    for (size_t i = 0; i < count; i++) {
        //mProc是注册的方法的函数指针
        if (array[i].mProc(env) < 0) {
#ifndef NDEBUG
            ALOGD("----------!!! %s failed to load\n", array[i].mName);
#endif
            return -1;
        }
    }
    return 0;
}
//android_os_MessageQueue.cpp
//需要注册的native方法以及Java端对应的方法名称以及函数的参数和返回值
static JNINativeMethod gMessageQueueMethods[] = {
    /* name, signature, funcPtr */
    { "nativeInit", "()J", (void*)android_os_MessageQueue_nativeInit },
    { "nativeDestroy", "(J)V", (void*)android_os_MessageQueue_nativeDestroy },
    { "nativePollOnce", "(JI)V", (void*)android_os_MessageQueue_nativePollOnce },
    { "nativeWake", "(J)V", (void*)android_os_MessageQueue_nativeWake },
    { "nativeIsIdling", "(J)Z", (void*)android_os_MessageQueue_nativeIsIdling }
};
//静态注册到gRegJNI数组中的方法，被在register_jni_procs中执行
int register_android_os_MessageQueue(JNIEnv* env) {
    //注册gMessageQueueMethods数组中的方法
    int res = jniRegisterNativeMethods(env, "android/os/MessageQueue",
            gMessageQueueMethods, NELEM(gMessageQueueMethods));
    LOG_FATAL_IF(res < 0, "Unable to register native methods.");
	  //对应的java端的类为android/os/MessageQueue
    jclass clazz;
    FIND_CLASS(clazz, "android/os/MessageQueue");
		//对应的java端的类android/os/MessageQueue中的mPtr对应的变量为J
    GET_FIELD_ID(gMessageQueueClassInfo.mPtr, clazz,"mPtr", "J");
    
    return 0;
}
```

### pollOnce和wake的native方法调用

```cpp
//android_os_MessageQueue.cpp
//{ "nativePollOnce", "(JI)V", (void*)android_os_MessageQueue_nativePollOnce },
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jclass clazz,
        jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, timeoutMillis);
}
//{ "nativeWake", "(J)V", (void*)android_os_MessageQueue_nativeWake }
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    return nativeMessageQueue->wake();
}
class NativeMessageQueue : public MessageQueue {
public:
    NativeMessageQueue();
    virtual ~NativeMessageQueue();
    virtual void raiseException(JNIEnv* env, const char* msg, jthrowable exceptionObj);
    void pollOnce(JNIEnv* env, int timeoutMillis);
    void wake();
private:
    bool mInCallback;
    jthrowable mExceptionObj;
}
void NativeMessageQueue::wake() {
    mLooper->wake();
}
void NativeMessageQueue::pollOnce(JNIEnv* env, int timeoutMillis) {
    mInCallback = true;
    mLooper->pollOnce(timeoutMillis);
    mInCallback = false;
    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```

我们可以看到 `nativePollOnce` 对中调用的是 `nativeMessageQueue->pollOnce` 方法最终调用 `Looper。pollOnce`，`nativeWake` 对中调用的是 `nativeMessageQueue->wake ` 方法最终调用 `Looper.wake` 。

### pollOnce和wake方法的声明

```cpp
//Looper.h
/**
 * Waits for events to be available, with optional timeout in milliseconds.
 * Invokes callbacks for all file descriptors on which an event occurred.
 *
 * If the timeout is zero, returns immediately without blocking.
 * If the timeout is negative, waits indefinitely until an event appears.
 *
 * Returns POLL_WAKE if the poll was awoken using wake() before
 * the timeout expired and no callbacks were invoked and no other file
 * descriptors were ready.
 *
 * Returns POLL_CALLBACK if one or more callbacks were invoked.
 *
 * Returns POLL_TIMEOUT if there was no data before the given
 * timeout expired.
 *
 * Returns POLL_ERROR if an error occurred.
 *
 * Returns a value >= 0 containing an identifier if its file descriptor has data
 * and it has no callback function (requiring the caller here to handle it).
 * In this (and only this) case outFd, outEvents and outData will contain the poll
 * events and data associated with the fd, otherwise they will be set to NULL.
 *
 * This method does not return until it has finished invoking the appropriate callbacks
 * for all file descriptors that were signalled.
 */
int pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData);
/**
 * Wakes the poll asynchronously.
 *
 * This method can be called on any thread.
 * This method returns immediately.
 */
void wake();
```

在这里的阅读代码的文档就对理解方法有很大的帮助，下面的源码分析中不懂的可以参考上面的文档。

### pollOnce和wake方法的实现

下边内容会涉及到一些Linux内核所提供的一种文件系统变化通知机制 `Epoll` 相关的知识点 ，[深入理解Android劵III-INofity与Epoll](https://www.kancloud.cn/alex_wsc/android-deep3/416419) 这篇文章讲的非常详细，建议大家阅读。

**pollOnce**和**wake**都是 `Looper` 的成员方法，那么在将具体方法之前我们先看看 `Looper` 的构造方法。

#### Looper的构造

```cpp
//Looper.cpp
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    //两个文件描述符引用，一个表示读端，一个表示写端
    int wakeFds[2];
    //调用pipe系统函数创建一个管道，
    int result = pipe(wakeFds);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not create wake pipe.  errno=%d", errno);
    //掌握着管道的写端
    mWakeReadPipeFd = wakeFds[0];
    //掌握着管道的读端
    mWakeWritePipeFd = wakeFds[1];
    //设置给mWakeReadPipeFd描述符状态非阻塞I/O标志
    result = fcntl(mWakeReadPipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake read pipe non-blocking.  errno=%d",errno);
    //设置给mWakeWritePipeFd描述符状态非阻塞I/O标志
    result = fcntl(mWakeWritePipeFd, F_SETFL, O_NONBLOCK);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake write pipe non-blocking.  errno=%d",errno);
    mIdling = false;
    //创建一个epoll对象的描述符，之后对epoll的操作均使用这个描述符完成。EPOLL_SIZE_HINT表示了此epoll对象可以监听的描述符的最大数量。
    // Allocate the epoll instance and register the wake pipe.
    mEpollFd = epoll_create(EPOLL_SIZE_HINT);
    LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance.  errno=%d", errno);
    //struct epoll_event {
    //  __uint32_tevents; /* 事件掩码，指明了需要监听的事件种类*/
    //  epoll_data_t data; /* 使用者自定义的数据，当此事件发生时该数据将原封不动地返回给使用者 */
    //};
    struct epoll_event eventItem;
    //将epoll_event类型结构的eventItem字段初始化为0
    memset(& eventItem, 0, sizeof(epoll_event)); // zero out unused members of data field union
    //EPOLLIN（可读），EPOLLOUT（可写），EPOLLERR（描述符发生错误），EPOLLHUP（描述符被挂起）
    eventItem.events = EPOLLIN;
    //eventItem需要监听的文件描述符mWakeReadPipeFd
    eventItem.data.fd = mWakeReadPipeFd;
    //将事件监听添加到epoll对象中去
    result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, & eventItem);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake read pipe to epoll instance.  errno=%d",errno);
}
```

> epoll_create(int max_fds)：创建一个epoll对象的描述符，之后对epoll的操作均使用这个描述符完成。max_fds参数表示了此epoll对象可以监听的描述符的最大数量。
>
> epoll_ctl (int epfd, int op,int fd, struct epoll_event *event)：用于管理注册事件的函数。这个函数可以增加/删除/修改事件的注册。

这里需要注意的是：我们往创建的 `mEpollFd` 添加的事件的 `events` 为 `EPOLLIN`  、 `data.fd` 是 `mWakeReadPipeFd`  ，这些东西我们后面会用到。

#### pollOnce

```cpp
//Looper.cpp
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    //处理eventItems中除过mWakeReadPipeFd之外的事件
    //(调用Looper::addFd方法添加的事件）队列mResponses中需要处理的事件
    //比如NativeDisplayEventReceiver的initialize方法，添加的文件描述符的ident=0
    for (;;) {
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            //inent>=0标识需要回调的事件，比如输入事件
            if (ident >= 0) {
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE
                ALOGD("%p ~ pollOnce - returning signalled identifier %d: "
                        "fd=%d, events=0x%x, data=%p",
                        this, ident, fd, events, data);
#endif
                if (outFd != NULL) *outFd = fd;
                if (outEvents != NULL) *outEvents = events;
                if (outData != NULL) *outData = data;
                return ident;
            }
        }
        //表示已经唤醒，或超时，或出错，或执行了消息回调
        if (result != 0) {
#if DEBUG_POLL_AND_WAKE
            ALOGD("%p ~ pollOnce - returning result %d", this, result);
#endif
            if (outFd != NULL) *outFd = 0;
            if (outEvents != NULL) *outEvents = 0;
            if (outData != NULL) *outData = NULL;
            return result;
        }
        //poll epool内部实现，方法会进行等待
        result = pollInner(timeoutMillis);
    }
}
```

- 先处理 `mResponses` 中需要返回结果的事件；
- 判断当前是否已经唤醒，或超时，或出错，或执行了消息回调，否则进行等待；

#### pollnner

这个方法比较长，请耐心阅读^_^。这个方法是正真调用 `epoll_wait` 方法进行等待事件的地方。

> int epoll_wait(int epfd, structepoll_event * events, int maxevents, int timeout)：用于等待事件的到来。当此函数返回时，events数组参数中将会包含产生事件的文件描述符。

```cpp
int Looper::pollInner(int timeoutMillis) {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - waiting: timeoutMillis=%d", this, timeoutMillis);
#endif
    //根据下一条消息的到期时间调整超时,
    // Adjust the timeout based on when the next message is due.
    //如果唤醒时间不等于0，而且下一条mMessageEnvelopes队列顶部消息的执行时间不为LLONG_MAX
    //mMessageEnvelopes相当于Java层的MessageQueue链表队列，队列中MessageEnvelope执行时间由小到大
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        //获取当前时间戳
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        //判断当前时间与下一条消息的执行时间的时间间隔，这里称为delay时间。
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        //如果下一条消息的delay时间在现在之后
        //而且本地休眠不需要唤醒（timeoutMillis < 0）或者下一条消息的delay时间比这次需要唤醒的时间靠前，那么修改本次唤醒时间
        //比如说这次休眠不需要唤醒，或者下一条消息是1s后唤醒，这次消息需要2s后唤醒，那么将唤醒时间修改为1s
        if (messageTimeoutMillis >= 0
                && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
            timeoutMillis = messageTimeoutMillis;
        }
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - next message in %lldns, adjusted timeout: timeoutMillis=%d",
                this, mNextMessageUptime - now, timeoutMillis);
#endif
    }
    //开始唤醒或等待唤醒
    // Poll.
    int result = POLL_WAKE;
    //回调容器清空，索引重置
    mResponses.clear();
    mResponseIndex = 0;
    // We are about to idle.
    mIdling = true;
    //eventItems为mEpollFd注册的时间
    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    //陷入等待状态，直到其注册的事件之一发生之后才会返回，并且携带了刚刚发生的事件的详细信息。
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
    // No longer idling.
    mIdling = false;
    // Acquire lock.
    mLock.lock();
    //epoll_wait返回值小于0表示错误
    // Check for poll error.
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        ALOGW("Poll failed with an unexpected error, errno=%d", errno);
        result = POLL_ERROR;
        goto Done;
    }
    //epoll_wait返回值等于0表示没有任何事件需要处理
    // Check for poll timeout.
    if (eventCount == 0) {
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - timeout", this);
#endif
        result = POLL_TIMEOUT;
        goto Done;
    }
    //epoll_wait返回值大于0，处理eventCount个事件
    // Handle all events.
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - handling events from %d fds", this, eventCount);
#endif
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        //判断是否是在构造函数中添加到epoll对象中的表示
        if (fd == mWakeReadPipeFd) {
            //管道中已经有了可读数据
            if (epollEvents & EPOLLIN) {
                //进行读数据唤醒线程，清理管道，以便于下一次管道写入信息进行唤醒looper
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake read pipe.", epollEvents);
            }
        } else {
            //其他事件(调用Looper::addFd方法添加的事件）添加到
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                // EPOLLIN ：可读（包括对端SOCKET正常关闭）；
                // EPOLLOUT：可写；
                // EPOLLERR：错误；
                // EPOLLHUP：中断；
                // EPOLLPRI：高优先级的可读（这里应该表示有带外数据到来）；
                // EPOLLET： 将EPOLL设为边缘触发模式，这是相对于水平触发来说的。
                // EPOLLONESHOT：只监听一次事件，当监听完这次事件之后就不再监听该事件
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                //将events添加到mResponses队列中
                pushResponse(events, mRequests.valueAt(requestIndex));
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                        "no longer registered.", epollEvents, fd);
            }
        }
    }
Done: ;
    //调用挂起的消息回调
    // Invoke pending message callbacks.
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        //获取当先时间
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        //获取链表顶部的MessageEnvelope对象
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        //如果该MessageEnvelope的执行时间比现在要早或是现在，那么处理这个MessageEnvelope，并移除这个MessageEnvelope
        if (messageEnvelope.uptime <= now) {
            // Remove the envelope from the list.
            // We keep a strong reference to the handler until the call to handleMessage
            // finishes.  Then we drop it so that the handler can be deleted *before*
            // we reacquire our lock.
            { // obtain handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                //移除这个消息
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();
#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
                ALOGD("%p ~ pollOnce - sending message: handler=%p, what=%d",
                        this, handler.get(), message.what);
#endif
                //处理这个消息
                handler->handleMessage(message);
            } // release handler
            mLock.lock();
            mSendingMessage = false;
            result = POLL_CALLBACK;
        } else {
            //直到顶部的MessageEnvelope的执行时间比现在晚
            //设置mNextMessageUptime为mMessageEnvelopes顶部消息的执行时间
            // The last message left at the head of the queue determines the next wakeup time.
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }
    // Release lock.
    mLock.unlock();
    //调用所有响应回调。 
    // Invoke all response callbacks.
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        //如果当前的Response不需要callback那么现在执行
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
            ALOGD("%p ~ pollOnce - invoking fd event callback %p: fd=%d, events=0x%x, data=%p",
                    this, response.request.callback.get(), fd, events, data);
#endif
            //处理所有回调的相应时间
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            //如果事件属于单次执行那么从mResponses删除这个文件描述符
            if (callbackResult == 0) {
                removeFd(fd);
            }
            ////立即清除响应结构中的回调引用，因为在下一次轮询之前不会清除响应向量本身。
            // Clear the callback reference in the response structure promptly because we
            // will not clear the response vector itself until the next poll.
            response.request.callback.clear();
            //需要回调响应
            result = POLL_CALLBACK;
        }
    }
    return result;
}
//进行读数据唤醒线程，清理管道，以便于下一次管道写入信息进行唤醒looper
void Looper::awoken() {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ awoken", this);
#endif
    char buffer[16];
    //成功返回读取的字节数，出错返回-1并设置errno
    ssize_t nRead;
    do {
        nRead = read(mWakeReadPipeFd, buffer, sizeof(buffer));
    } while ((nRead == -1 && errno == EINTR) || nRead == sizeof(buffer));
}
```

好吧，方法真长。让我们继续 `fuck the source code` ，用我们自己的语言叙述一下这个方法。

- 调整唤醒的超时时间，判断这个唤醒时间与 `MessageQueue` 链表头部消息的唤醒时间；
- 清除`mResponses` 内容重置索引，开始陷入等待事件中；
- epoll_wait返回值小于0，result = POLL_ERROR;
- epoll_wait返回值等于0，result = POLL_TIMEOUT;
- epoll_wait返回值大于0，处理已经发生的事件；
- - 如果文件描述符是 `mWakeReadPipeFd` 而且事件为 `EPOLLIN` ，这个标识管道有数据写入，唤醒线程。需要的操作是清楚管道数据，等待下一次被唤醒；
  - 否则将这个已经发送的事件添加到 `mResponses` 队列当中；
- 处理**C层**消息队列 `mMessageEnvelopes` 中执行时间已经到期的消息；
- 处理 `mResponses` 数组中不需要信息返回的事件；

#### wake

```cpp
//Looper.cpp
void Looper::wake() {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ wake", this);
#endif
    //写入文档的字节数（成功）；-1（出错）
    ssize_t nWrite;
    //Linux中系统调用的错误都存储于errno中
    //#define EPERM        1  /* Operation not permitted */
    do {
        nWrite = write(mWakeWritePipeFd, "W", 1);
    } while (nWrite == -1 && errno == EINTR);
    if (nWrite != 1) {
        //#define EAGAIN      11  /* Try again */
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal, errno=%d", errno);
        }
    }
}
```

当管道中写入数据的时候，管道的另一端就可以进行读操作，所以添加到 `Epoll` 中的事件就会进行处理从而唤起当前的线程。

## 结论

在 `Looper` 的循环中我们由消息就处理消息，没有消息使用 `epoll_wait` 挂起当前的线程，这个时候是不会消耗 `CPU` 资源（或者说消耗的非常少）。

所以**Looper的无线循环会使线程卡死么？**这个问题的答案我们已经得到了不是么~！



文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！