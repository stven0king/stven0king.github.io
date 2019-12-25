---
title: Android中的数据库
date: 2019-11-02 20:27:00
tags: [数据库连接池]
categories: 数据库
description: "最近在看数据库相关的三方库的时候，我发现在Android应用开发的时候是可以并行操作数据库的读写，但Android默认的数据连接池中只有一个数据库链接。一个数据库连接能实现并发么？要是一个数据库链接可以实现并发，那么为什么需要数据库连接池？"
---


![在这里插入图片描述](https://img-blog.csdnimg.cn/20191102201512480.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)
最近在看数据库相关的三方库的时候，我发现在Android应用开发的时候是可以并行操作数据库的读写，但Android默认的数据连接池中只有一个数据库链接。一个数据库连接能实现并发么？要是一个数据库链接可以实现并发，那么为什么需要数据库连接池？

# 数据库连接池介绍

每次提到**连接池**我们很快能想到**线程池**。线程池的创建可以减少了创建和销毁线程的次数，每个工作线程都可以被重复利用，可执行多个任务。



**数据库连接**是一种关键的有限的昂贵的资源，对数据库连接的管理能显著影响到整个应用程序的伸缩性和健壮性,影响到程序的性能指标。**数据库连接池负责分配,管理和释放数据库连接,它允许应用程序重复使用一个现有的数据库连接，减少链接不断传销和销毁带来的资源浪费。**



**数据库连接池**在初始化时将创建一定数量的数据库连接放到连接池中,，这些数据库连接的数量是由最小数据库连接数来设定的。无论这些数据库连接是否被使用，连接池都将一直保证至少拥有这么多的连接数量。连接池的最大数据库连接数量限定了这个连接池能占有的最大连接数，当应用程序向连接池请求的连接数超过最大连接数量时，这些请求将被加入到等待队列中。

>  数据库连接池的最小连接数和最大连接数的设置要考虑到以下几个因素:

1. 最小连接数：是连接池一直保持的数据库连接，所以如果应用程序对数据库连接的使用量不大，将会有大量的数据库连接资源被浪费。

2. 最大连接数:是连接池能申请的最大连接数,如果数据库连接请求超过次数,后面的数据库连接请求将被加入到等待队列中,这会影响以后的数据库操作

3. 如果最小连接数与最大连接数相差很大：那么最先连接请求将会获利，之后超过最小连接数量的连接请求等价于建立一个新的数据库连接。不过，这些大于最小连接数的数据库连接在使用完不会马上被释放，他将被放到连接池中等待重复使用或是空间超时后被释放。



# Android数据库相关类介绍

- SQLiteOpenHelper：管理SQLite的帮助类，提供获取SQLIteDatabase实例的方法，它会在第一次使用数据库时调用获取实例方法时创建SQLiteDatabase实例，并且处理数据库版本变化，开发人员在实现ContentProvider时都要实现一个自定义的SQLiteOpenHelper类，处理数据的创建、升级和降级。
- SQLiteDatabase：代表一个打开的SQLite数据库，提供了执行数据库操作的接口方法。如果不需要在进程之间共享数据，应用程序也可以自行创建这个类的实例来读写SQLite数据库。 
- SQLiteSession：SQLiteSession负责管理数据库连接和事务的生命周期，通过SQLiteConnectionPool获取数据库连接来执行具体的数据库操作。 
- SQLiteConnectionPool：数据库连接池，管理所有打开的数据库连接（Connection）。所有数据库连接都是通过它来打开，打开后会加入连接池，在读写数据库时需要从连接池中获取一个数据库连接来使用。 
- SQLiteConnection：代表了数据库连接，每个Connection封装了一个native层的sqlite3实例，通过JNI调用SQLite动态库的接口方法操作数据库，Connection要么被Session持有，要么被连接池持有。 
- CursorFactory：可选的Cursor工厂，可以提供自定义工厂来创建Cursor。 
- DatabaseErrorHandler：可选的数据库异常处理器（目前仅处理数据库Corruption），如果不提供，将会使用默认的异常处理器。 
- SQLiteDatabaseConfiguration：数据库配置，应用程序可以创建多个到SQLite数据库的连接，这个类用来保证每个连接的配置都是相同的。 
- SQLiteQuery和SQLiteStatement：从抽象类SQLiteProgram派生，封装了SQL语句的执行过程，在执行时自动组装待执行的SQL语句，并调用SQLiteSession来执行数据库操作。这两个类的实现应用了设计模式中的命令模式。 

每个类的更加详细的介绍可以阅读 [**SQLite**数据库学习小结**——Frameworks**层实现](https://www.cnblogs.com/supersand/p/5560091.html) 这篇文章，我们这里主要学习 `SQLiteConnectionPool` 相关的知识。



# SQLiteConnectionPool

**数据库连接池**我们先看一下它的大小，每个链接的获取以及其他功能？

## 连接池大小

目前Android系统的实现中，如果以非WAL模式打开数据库，连接池中只会保持一个数据库连接，如果以WAL模式打开数据库，连接池中的最大连接数量则根据系统配置决定，默认配置是两个。

```java
//SQLiteConnectionPool.java
private SQLiteConnectionPool(SQLiteDatabaseConfiguration configuration) {
    //数据库的配置信息
    mConfiguration = new SQLiteDatabaseConfiguration(configuration);
    //设置最大的数据库链接个数
    setMaxConnectionPoolSizeLocked();
    //超时处理句柄设置，如果超时时间为MAX_VALUE，那么链接永远不关闭
    // If timeout is set, setup idle connection handler
    // In case of MAX_VALUE - idle connections are never closed
    if (mConfiguration.idleConnectionTimeoutMs != Long.MAX_VALUE) {
        setupIdleConnectionHandler(Looper.getMainLooper(),
                mConfiguration.idleConnectionTimeoutMs);
    }
}
private void setMaxConnectionPoolSizeLocked() {
    if (!mConfiguration.isInMemoryDb()
            && (mConfiguration.openFlags & SQLiteDatabase.ENABLE_WRITE_AHEAD_LOGGING) != 0){
        //获取debug.sqlite.wal.poolsize配置的大小，默认值是com.android.internal.R.integer.db_connection_pool_size
        //最小值为2
        mMaxConnectionPoolSize = SQLiteGlobal.getWALConnectionPoolSize();
    } else {
        // We don't actually need to always restrict the connection pool size to 1
        // for non-WAL databases.  There might be reasons to use connection pooling
        // with other journal modes. However, we should always keep pool size of 1 for in-memory
        // databases since every :memory: db is separate from another.
        // For now, enabling connection pooling and using WAL are the same thing in the API.
        //内存数据库和非WAL数据库时数据库连接池大小为1
        mMaxConnectionPoolSize = 1;
    }
}
```

虽然名为连接池，但是从源码来看，目前实现的池中只有一个数据库连接（以后的Android版本可能会扩展），所以如果应用程序中有大量的并发数据库读和写操作的话，每个操作的时长都可能受到影响，所以数据库操作应放在工作线程中执行，以免影响UI响应。



**这里有人可能产生疑问，我在进行Android应用开发的时候是可以并行操作数据库的读写，一个数据库连接能实现并发么？要是一个数据库链接可以实现并发，那么为什么需要数据库连接池？**

这里说一下我自己的理解：一个**数据库链接**是一个`Socket` 通道，当这个`Connection` 被其它 `Session`占用的时候后续的`Session` 的操作必须等待这个 `Connection` 被释放，所以数据库的 `Connection` 的工作其实是串行的，这个在 `MySql` 和 `Oracle` 的文档中也能找到描述。所以在Android中默认的**数据库连接池**只有一个数据库链接的时候，所有在这个数据库上的操作都是串行的。我们平时在多线程中的数据库操作都是串行的。



这些将会下下面代码分析的过程中一一体现出来^_^



## 数据库链接池的构造

这里主要讲数据库连接池的创建和池中的第一条链接的产生。

```java
//SQLiteConnectionPool.java
public final class SQLiteConnectionPool implements Closeable {
    private static final String TAG = "SQLiteConnectionPool";
    // Amount of time to wait in milliseconds before unblocking acquireConnection
    // and logging a message about the connection pool being busy.
    private static final long CONNECTION_POOL_BUSY_MILLIS = 30 * 1000; // 30 seconds
    private final CloseGuard mCloseGuard = CloseGuard.get();
    private final Object mLock = new Object();
    private final AtomicBoolean mConnectionLeaked = new AtomicBoolean();
    //数据库的配置信息
    private final SQLiteDatabaseConfiguration mConfiguration;
    //数据库连接池的最大链接数
    private int mMaxConnectionPoolSize;
    //数据库是否打开
    private boolean mIsOpen;
    //创建的链接id
    private int mNextConnectionId;
    //连接等待池其实是由等待的连接组成的链
    private ConnectionWaiter mConnectionWaiterPool;
    //连接等待队列
    private ConnectionWaiter mConnectionWaiterQueue;
    //非主链接的引用，强引用需要主动回收
    // Strong references to all available connections.
    private final ArrayList<SQLiteConnection> mAvailableNonPrimaryConnections =
            new ArrayList<SQLiteConnection>();
    //主链接
    private SQLiteConnection mAvailablePrimaryConnection;
    /***部分代码省略***/
}
```



## 打开数据库

当我们在进行 `SQLiteOpenHelper.getWritableDatabase` 和 `SQLiteOpenHelper.getReadableDatabase` 的时候如果数据库没有打开那么会打开数据库，打开数据库也就是创建数据库链接。

```java
//SQLiteDatabase.java
private static SQLiteDatabase openDatabase(@NonNull String path,
        @NonNull OpenParams openParams) {
    Preconditions.checkArgument(openParams != null, "OpenParams cannot be null");
    SQLiteDatabase db = new SQLiteDatabase(path, openParams.mOpenFlags,
            openParams.mCursorFactory, openParams.mErrorHandler,
            openParams.mLookasideSlotSize, openParams.mLookasideSlotCount,
            openParams.mIdleConnectionTimeout, openParams.mJournalMode, openParams.mSyncMode);
    //内部调用openInner
    db.open();
    return db;
}
//打开数据库
private void openInner() {
    synchronized (mLock) {
        assert mConnectionPoolLocked == null;
        mConnectionPoolLocked = SQLiteConnectionPool.open(mConfigurationLocked);
        mCloseGuardLocked.open("close");
    }
    synchronized (sActiveDatabases) {
        sActiveDatabases.put(this, null);
    }
}
```

在这里我们看到了`SQLiteConnectionPoll` 的调用，这里由数据库链接池创建数据库链接从而打开数据库。

```java
//SQLiteConnectionPool.java
public static SQLiteConnectionPool open(SQLiteDatabaseConfiguration configuration) {
    //校验数据库信息
    if (configuration == null) {
        throw new IllegalArgumentException("configuration must not be null.");
    }
    // Create the pool.创建连接池
    SQLiteConnectionPool pool = new SQLiteConnectionPool(configuration);
    pool.open(); // might throw
    return pool;
}
// Might throw，打开数据库
private void open() {
    // Open the primary connection.
    // This might throw if the database is corrupt.
    //获取主链接并打开数据库，如果数据库损坏可能抛出异常
    mAvailablePrimaryConnection = openConnectionLocked(mConfiguration,
            true /*primaryConnection*/); // might throw
    // Mark it released so it can be closed after idle timeout
    //释放当前链接，以便于被关闭或者被超时回收
    synchronized (mLock) {
        if (mIdleConnectionHandler != null) {
            mIdleConnectionHandler.connectionReleased(mAvailablePrimaryConnection);
        }
    }
    // Mark the pool as being open for business.
    mIsOpen = true;
    mCloseGuard.open("close");
}
// Might throw.
private SQLiteConnection openConnectionLocked(SQLiteDatabaseConfiguration configuration,
    boolean primaryConnection) {
    //connectionId作为链接id，每次新创建一个数据库链接id自增1
    final int connectionId = mNextConnectionId++;
    return SQLiteConnection.open(this, configuration,connectionId, primaryConnection); // might throw
}
//SQLiteConnection.java
// Called by SQLiteConnectionPool only.
static SQLiteConnection open(SQLiteConnectionPool pool, SQLiteDatabaseConfiguration configuration,
    int connectionId, boolean primaryConnection) {
    SQLiteConnection connection = new SQLiteConnection(pool, configuration,connectionId, primaryConnection);
    try {
        //建立数据库链接
        connection.open();
        return connection;
    } catch (SQLiteException ex) {
        connection.dispose(false);
        throw ex;
    }
}
```

在这里我们能看到**数据库连接池**第一条链接的创建是在打开数据库的时候。



## 创建数据库链接

除过在打开数据的时候创建数据库链接，我们还会在一下情况下可能创建数据库链接。

- 创建数据库，调用open
- 重新加载数据库配置，调用reconfigure
- 创建主链接，调用tryAcquirePrimaryConnectionLocked
- 创建非主链接，调用tryAcquireNonPrimaryConnectionLocked

这四种情况我们都可能会调用 `SQLiteConnectionPool.openConnectionLocked` 创建数据库链接，其中**创建数据库**和**重新加载数据库配置**都是创建的**主链接**。



## 数据库链接的使用

在这之前我们先回想 `Connection` 和 `Session` 的概念：

> 连接（Connection）：连接是从客户端到ORACLE实例的一条物理路径。连接可以在网络上建立，或者在本机通过IPC机制建立。通常会在客户端进程与一个专用服务器或一个调度器之间建立连接。

 

> 会话(Session) 是和连接(Connection)是同时建立的，两者是对同一件事情不同层次的描述。简单讲，连接(Connection)是物理上的客户端同服务器的通信链路，会话(Session)是逻辑上的用户同服务器的通信交互。



我们一般往数据库插入一条数据：

```java
//创建数据库的help
OpenHelper openHelper = new OpenHelper(getApplicationContext(), "demo", null, 1);
//打开数据库标识写操作
SQLiteDatabase writableDatabase = tanzhenxing.getWritableDatabase();
ContentValues contentValues = new ContentValues();
contentValues.put("TIME", System.currentTimeMillis());
//向SYSTEM_MSG表插入一条数据，TIME的当时间戳
writableDatabase.insert("SYSTEM_MSG", null, contentValues);
```

SQliteDatabase的内部调用：

```java
//SQLiteDatabase.java
public long insert(String table, String nullColumnHack, ContentValues values) {
    try {
        //内部封装SQLiteStatement，调用statement.executeInsert();
        return insertWithOnConflict(table, nullColumnHack, values, CONFLICT_NONE);
    } catch (SQLException e) {
        Log.e(TAG, "Error inserting " + values, e);
        return -1;
    }
}
//SQLiteStatement.java
public long executeInsert() {
    acquireReference();
    try {
        return getSession().executeForLastInsertedRowId(
                getSql(), getBindArgs(), getConnectionFlags(), null);
    } catch (SQLiteDatabaseCorruptException ex) {
        onCorruption();
        throw ex;
    } finally {
        releaseReference();
    }
}
//SQLiteStatement.java的父类SQLiteProgram的方法
protected final SQLiteSession getSession() {
    return mDatabase.getThreadSession();
}
```

在这里我们能看到最终的执行是有 `Session` 进行操作的。

```java
//SQLiteDatabase.java
// Thread-local for database sessions that belong to this database.
// Each thread has its own database session.
// INVARIANT: Immutable.
//属于当前数据库的会话，每个线程都有一会话，不可变。
private final ThreadLocal<SQLiteSession> mThreadSession = ThreadLocal
        .withInitial(this::createSession);
SQLiteSession createSession() {
    final SQLiteConnectionPool pool;
    synchronized (mLock) {
        throwIfNotOpenLocked();
        pool = mConnectionPoolLocked;
    }
    return new SQLiteSession(pool);
}
SQLiteSession getThreadSession() {
    return mThreadSession.get(); // initialValue() throws if database closed
}
```

好了扯了这么久到底什么时候使用 `Connection` 呢？

```java
//SQLiteSession.java
public long executeForLastInsertedRowId(String sql, Object[] bindArgs, int connectionFlags,
        CancellationSignal cancellationSignal) {
    //校验sql
    if (sql == null) {
        throw new IllegalArgumentException("sql must not be null.");
    }
    //对某些SQL语句（例如“ BEGIN”," COMMIT”和“ ROLLBACK”）执行特殊的重新解释,以确保事务状态不变式为保持。 
    if (executeSpecial(sql, bindArgs, connectionFlags, cancellationSignal)) {
        return 0;
    }
    //获取数据库链接
    acquireConnection(sql, connectionFlags, cancellationSignal); // might throw
    try {
        //使用数据库链接进行数据库操作
        return mConnection.executeForLastInsertedRowId(sql, bindArgs,
                cancellationSignal); // might throw
    } finally {
        //释放数据库链接
        releaseConnection(); // might throw
    }
}
//从数据库连接池中获取链接
private void acquireConnection(String sql, int connectionFlags,
        CancellationSignal cancellationSignal) {
    if (mConnection == null) {
        assert mConnectionUseCount == 0;
        mConnection = mConnectionPool.acquireConnection(sql, connectionFlags,
                cancellationSignal); // might throw
        mConnectionFlags = connectionFlags;
    }
    mConnectionUseCount += 1;
}
```

我们总结一下上述内容：我们进行数据库操作的时候每次操作都使用的`Session` ，多个线程执行数据库操作会有多个`Session` 。`Session` 的内部操作调用的是`Connection` ，`Connection` 是从**数据库连接池**中获取的。

如果**数据库连接池**有多个数据库链接，那么数据库的殂谢操作可以并发，否则只能串行操作。



## 从连接池中获取数据库链接

```java
//SQLiteConnectionPool.java
public SQLiteConnection acquireConnection(String sql, int connectionFlags,
        CancellationSignal cancellationSignal) {
    SQLiteConnection con = waitForConnection(sql, connectionFlags, cancellationSignal);
    synchronized (mLock) {
        if (mIdleConnectionHandler != null) {
            mIdleConnectionHandler.connectionAcquired(con);
        }
    }
    return con;
}
```

从上面的 `waitForConnection` 方法的名字我们可以猜测这个方法可能产生阻塞。

```java
//SQLiteConnectionPool.java
// Might throw.
private SQLiteConnection waitForConnection(String sql, int connectionFlags,
        CancellationSignal cancellationSignal) {
    //是否需要主链接
    final boolean wantPrimaryConnection =
            (connectionFlags & CONNECTION_FLAG_PRIMARY_CONNECTION_AFFINITY) != 0;
    final ConnectionWaiter waiter;
    final int nonce;
    synchronized (mLock) {
        throwIfClosedLocked();//如果数据库关闭，那么抛出异常
        // Abort if canceled.
        //如果取消信号的回调不为空，那么执行回调检测是否需要取消
        if (cancellationSignal != null) {
            cancellationSignal.throwIfCanceled();
        }
        // Try to acquire a connection.
        //尝试获得一个数据库链接
        SQLiteConnection connection = null;
        //如果不需要主链接，那么尝试获取非主链接
        if (!wantPrimaryConnection) {
            connection = tryAcquireNonPrimaryConnectionLocked(
                    sql, connectionFlags); // might throw
        }
        //如果获取不到非链接，那么尝试获取主链接
        if (connection == null) {
            connection = tryAcquirePrimaryConnectionLocked(connectionFlags); // might throw
        }
        if (connection != null) {
            return connection;
        }
        // No connections available.  Enqueue a waiter in priority order.
        //没有可用的连接。按优先级排队服务员。
        final int priority = getPriority(connectionFlags);
        final long startTime = SystemClock.uptimeMillis();
        //创建一个等待获取链接的对象
        waiter = obtainConnectionWaiterLocked(Thread.currentThread(), startTime,
                priority, wantPrimaryConnection, sql, connectionFlags);
        ConnectionWaiter predecessor = null;
        ConnectionWaiter successor = mConnectionWaiterQueue;
        //按照优先级查找插入的位置
        while (successor != null) {
            if (priority > successor.mPriority) {
                waiter.mNext = successor;
                break;
            }
            predecessor = successor;
            successor = successor.mNext;
        }
        //插入等待队列
        if (predecessor != null) {
            predecessor.mNext = waiter;
        } else {
            mConnectionWaiterQueue = waiter;
        }
        nonce = waiter.mNonce;
    }
    // Set up the cancellation listener.
    //设置取消监听器，在等待的过程中如果取消等待那么执行cancelConnectionWaiterLocked
    if (cancellationSignal != null) {
        cancellationSignal.setOnCancelListener(new CancellationSignal.OnCancelListener() {
            @Override
            public void onCancel() {
                synchronized (mLock) {
                    if (waiter.mNonce == nonce) {
                        //从等待队列中删除这个节点数据
                        //给waiter添加OperationCanceledException异常信息
                        //唤醒waiter对应线程的阻塞
                        //调用wakeConnectionWaitersLocked判断队列其他waiter是否状态有更新
                        cancelConnectionWaiterLocked(waiter);
                    }
                }
            }
        });
    }
    try {
        // Park the thread until a connection is assigned or the pool is closed.
        // Rethrow an exception from the wait, if we got one.
        //驻留线程，直到分配了连接或关闭了池。
        //如果有异常，则从等待中抛出异常。 
        long busyTimeoutMillis = CONNECTION_POOL_BUSY_MILLIS;
        long nextBusyTimeoutTime = waiter.mStartTime + busyTimeoutMillis;
        for (;;) {
            // Detect and recover from connection leaks.
            //是否需要从泄露中进行恢复，之前被调用onConnectionLeaked
            if (mConnectionLeaked.compareAndSet(true, false)) {
                synchronized (mLock) {
                    //为等待数据库链接队列进行链接赋值
                    wakeConnectionWaitersLocked();
                }
            }
            // Wait to be unparked (may already have happened), a timeout, or interruption.
            //阻塞busyTimeoutMillis毫秒，或者中间被执行LockSupport.unpark
            //被执行cancelConnectionWaiterLocked进行取消
            //或者被执行wakeConnectionWaitersLocked进行链接分配
            LockSupport.parkNanos(this, busyTimeoutMillis * 1000000L);
            // Clear the interrupted flag, just in case.
            Thread.interrupted();//重置当前线程的中断状态
            // Check whether we are done waiting yet.
            //检查我们是否已经完成等待。
            synchronized (mLock) {
                throwIfClosedLocked();//如果数据库关闭，那么抛出异常
                final SQLiteConnection connection = waiter.mAssignedConnection;
                final RuntimeException ex = waiter.mException;
                //如果已经分配链接，或者发送异常
                if (connection != null || ex != null) {
                    recycleConnectionWaiterLocked(waiter);//回收waiter
                    if (connection != null) {//返回分配链接
                        return connection;
                    }
                    throw ex; // rethrow!重新抛出异常
                }

                final long now = SystemClock.uptimeMillis();
                if (now < nextBusyTimeoutTime) {
                    //parkNanos阻塞时间不够busyTimeoutMillis毫秒，被执行LockSupport.unpark
                    busyTimeoutMillis = now - nextBusyTimeoutTime;
                } else {
                    //输出日志
                    logConnectionPoolBusyLocked(now - waiter.mStartTime, connectionFlags);
                    //重置下次阻塞时间
                    busyTimeoutMillis = CONNECTION_POOL_BUSY_MILLIS;
                    nextBusyTimeoutTime = now + busyTimeoutMillis;
                }
            }
        }
    } finally {
        // Remove the cancellation listener.
        //有异常，或者获取等到了分配的链接那么解绑回调信息
        if (cancellationSignal != null) {
            cancellationSignal.setOnCancelListener(null);
        }
    }
}
```

这里利用`LockSupport.parkNanos` 循环**判断是否获得了数据库链接否则继续睡眠**，直到这次操作被取消或者获得数据库链接。


### 主链接的获取

```java
//SQLiteConnectionPool.java
// Might throw.
@GuardedBy("mLock")
private SQLiteConnection tryAcquirePrimaryConnectionLocked(int connectionFlags) {
    // If the primary connection is available, acquire it now.
    //如果主要连接可用，请立即获取。
    SQLiteConnection connection = mAvailablePrimaryConnection;
    if (connection != null) {
        mAvailablePrimaryConnection = null;
        finishAcquireConnectionLocked(connection, connectionFlags); // might throw
        return connection;
    }
    // Make sure that the primary connection actually exists and has just been acquired.
    //确保主要连接确实存在并且刚刚被获取。
    for (SQLiteConnection acquiredConnection : mAcquiredConnections.keySet()) {
        if (acquiredConnection.isPrimaryConnection()) {
            return null;
        }
    }
    // Uhoh.  No primary connection!  Either this is the first time we asked
    // for it, or maybe it leaked?
    //第一次创建数据库主链接，或者主链接被回收
    connection = openConnectionLocked(mConfiguration,
            true /*primaryConnection*/); // might throw
    finishAcquireConnectionLocked(connection, connectionFlags); // might throw
    return connection;
}
```

我们可以看到**主数据库链接**只会有一个，如果被占用那么需要等待，如果没有那么就需要创建。

### 获取非主链接

```java
//SQLiteConnectionPool.java
// Might throw.
private SQLiteConnection tryAcquireNonPrimaryConnectionLocked(
        String sql, int connectionFlags) {
    // Try to acquire the next connection in the queue.
    SQLiteConnection connection;
    //尝试获取队列中的下一个连接。
    final int availableCount = mAvailableNonPrimaryConnections.size();
    if (availableCount > 1 && sql != null) {
        // If we have a choice, then prefer a connection that has the
        // prepared statement in its cache.
        //检查我们是否可以在其缓存中选择具有prepare语句的连接。
        for (int i = 0; i < availableCount; i++) {
            connection = mAvailableNonPrimaryConnections.get(i);
            if (connection.isPreparedStatementInCache(sql)) {
                mAvailableNonPrimaryConnections.remove(i);
                finishAcquireConnectionLocked(connection, connectionFlags); // might throw
                return connection;
            }
        }
    }
    //否则获取可以非用连接队列中的最后一个链接
    if (availableCount > 0) {
        // Otherwise, just grab the next one.
        connection = mAvailableNonPrimaryConnections.remove(availableCount - 1);
        finishAcquireConnectionLocked(connection, connectionFlags); // might throw
        return connection;
    }
    //如果没有可以的非主链接，那么就需要扩展数据库连接池
    // Expand the pool if needed.
    int openConnections = mAcquiredConnections.size();
    if (mAvailablePrimaryConnection != null) {
        openConnections += 1;
    }
    //如果数据库连接池已经达到上限那么，返回null
    if (openConnections >= mMaxConnectionPoolSize) {
        return null;
    }
    //否则创建新的非主链接
    connection = openConnectionLocked(mConfiguration,
            false /*primaryConnection*/); // might throw
    finishAcquireConnectionLocked(connection, connectionFlags); // might throw
    return connection;
}
```

**非主数据库链接**数量的多少受限于数据库连接池的大小。

## 数据库链接释放

有创建获取就会有释放回收。

```java
//SQLiteConnectionPool.java
//释放数据库链接返回连接池
public void releaseConnection(SQLiteConnection connection) {
    synchronized (mLock) {
        //idle链接句柄事件处理connectionReleased
        if (mIdleConnectionHandler != null) {
            mIdleConnectionHandler.connectionReleased(connection);
        }
        //获取这个链接的状态
        //NORMAL,正常返回连接池
        //RECONFIGURE,必须先重新配置连接，然后才能返回。
        //DISCARD,连接必须关闭并丢弃。
        AcquiredConnectionStatus status = mAcquiredConnections.remove(connection);
        if (status == null) {
            throw new IllegalStateException("Cannot perform this operation "
                    + "because the specified connection was not acquired "
                    + "from this pool or has already been released.");
        }
        //简历是否已经关闭连接池
        if (!mIsOpen) {
            closeConnectionAndLogExceptionsLocked(connection);
        } else if (connection.isPrimaryConnection()) {//如果是主链接
            //判断这个数据库链接是否需要回收
            if (recycleConnectionLocked(connection, status)) {
                assert mAvailablePrimaryConnection == null;
                mAvailablePrimaryConnection = connection;//标识主链接可用，被占用的时候为null
            }
            ////判断队列其他waiter是否状态有更新
            wakeConnectionWaitersLocked();
        } else if (mAvailableNonPrimaryConnections.size() >= mMaxConnectionPoolSize - 1) {
            //可用的非主链接数+主链接大于或等于数据库连接池的最大链接数的时候关闭这个链接
            closeConnectionAndLogExceptionsLocked(connection);
        } else {
            //判断这个数据库链接是否需要回收
            if (recycleConnectionLocked(connection, status)) {
                //将这个链接添加到非主链接容器中
                mAvailableNonPrimaryConnections.add(connection);
            }
            //判断队列其他waiter是否状态有更新
            wakeConnectionWaitersLocked();
        }
    }
}

//取消所有具有我们可以满足的要求的waiter的park，即唤醒该waiter对应的线程
//这个方法并不会抛异常，而是将异常赋值给waiter进行抛出
// Can't throw.
@GuardedBy("mLock")
private void wakeConnectionWaitersLocked() {
    // Unpark all waiters that have requests that we can fulfill.
    // This method is designed to not throw runtime exceptions, although we might send
    // a waiter an exception for it to rethrow.
    ConnectionWaiter predecessor = null;
    //链表的头结点
    ConnectionWaiter waiter = mConnectionWaiterQueue;
    boolean primaryConnectionNotAvailable = false;
    boolean nonPrimaryConnectionNotAvailable = false;
    while (waiter != null) {
        boolean unpark = false;
        //是否关闭了数据库，如果关闭了那么唤醒所有waiter的线程
        if (!mIsOpen) {
            unpark = true;
        } else {
            try {
                SQLiteConnection connection = null;
                //如果该waiter需要非主链接，而且现在有可用的非主链接
                if (!waiter.mWantPrimaryConnection && !nonPrimaryConnectionNotAvailable) {
                    //获取非主链接
                    connection = tryAcquireNonPrimaryConnectionLocked(
                            waiter.mSql, waiter.mConnectionFlags); // might throw
                    //获取为空，标识现在没有可用的非主链接        
                    if (connection == null) {
                        nonPrimaryConnectionNotAvailable = true;
                    }
                }
                //主链接可以用
                if (connection == null && !primaryConnectionNotAvailable) {
                    //尝试获取主链接
                    connection = tryAcquirePrimaryConnectionLocked(
                            waiter.mConnectionFlags); // might throw
                    //获取为空，标识现在主链接不可用
                    if (connection == null) {
                        primaryConnectionNotAvailable = true;
                    }
                }
                //获取到了数据库链接
                if (connection != null) {
                    waiter.mAssignedConnection = connection;//改waiter赋值链接
                    unpark = true;//唤醒该waiter的对应线程
                } else if (nonPrimaryConnectionNotAvailable && primaryConnectionNotAvailable) {
                    // There are no connections available and the pool is still open.
                    // We cannot fulfill any more connection requests, so stop here.
                    //连接池任然可用，但是没有可用的链接没法对其他的waiter状态做更新直接返回
                    break;
                }
            } catch (RuntimeException ex) {
                // Let the waiter handle the exception from acquiring a connection.
                waiter.mException = ex;
                unpark = true;
            }
        }

        final ConnectionWaiter successor = waiter.mNext;
        //如果需要唤醒，那么从链表中删除这个waiter，并进行对应线程唤醒操作
        if (unpark) {
            if (predecessor != null) {
                predecessor.mNext = successor;
            } else {
                mConnectionWaiterQueue = successor;
            }
            waiter.mNext = null;
            LockSupport.unpark(waiter.mThread);
        } else {
            predecessor = waiter;
        }
        waiter = successor;
    }
}
```

链接的释放有时候是为了回收，有时候为了重用。重用的时候还需要唤醒等待链接队列中获得这个链接的`waiter` 。



## 数据库链接池的关闭

说到**数据库连接池**的关闭，我们会联想到**数据库的关闭**和**数据库链接的关闭**。

```java
//SQLiteClosable.java，它是SQLiteDatabase的父类
/**
 * Releases a reference to the object, closing the object if the last reference
 * was released.
 *
 * Calling this method is equivalent to calling {@link #releaseReference}.
 *
 * @see #releaseReference()
 * @see #onAllReferencesReleased()
 */
//释放引用的对象，直到所有的引用都被释放了那么关闭数据库
public void close() {
    releaseReference();
}
public void releaseReference() {
    boolean refCountIsZero = false;
    synchronized(this) {
        refCountIsZero = --mReferenceCount == 0;
    }
    if (refCountIsZero) {
        onAllReferencesReleased();
    }
}
//SQLiteDatabase.java
@Override
protected void onAllReferencesReleased() {
    dispose(false);
}
private void dispose(boolean finalized) {
    final SQLiteConnectionPool pool;
    synchronized (mLock) {
        if (mCloseGuardLocked != null) {
            if (finalized) {
                mCloseGuardLocked.warnIfOpen();
            }
            mCloseGuardLocked.close();
        }
        //连接池置空，无法进行新操作
        pool = mConnectionPoolLocked;
        mConnectionPoolLocked = null;
    }
    if (!finalized) {
        //删除当前数据库的引用
        synchronized (sActiveDatabases) {
            sActiveDatabases.remove(this);
        }
        //关闭数据库连接池
        if (pool != null) {
            pool.close();
        }
    }
}
```

我们可以看到**数据库连接池的关闭**是由**数据库关闭**引起的。

### 数据库连接池的关闭

```java
//SQLiteConnectionPool.java
/**
 * Closes the connection pool.
 * <p>
 * When the connection pool is closed, it will refuse all further requests
 * to acquire connections.  All connections that are currently available in
 * the pool are closed immediately.  Any connections that are still in use
 * will be closed as soon as they are returned to the pool.
 * </p>
 *
 * @throws IllegalStateException if the pool has been closed.
 */
//关闭数据库连接池，停止接受新的数据库链接的请求。
//链接池中的可用链接立即被关闭，其他正在使用的链接被归还到数据的时候关闭
public void close() {
    dispose(false);
}
private void dispose(boolean finalized) {
    if (mCloseGuard != null) {
        if (finalized) {
            mCloseGuard.warnIfOpen();
        }
        mCloseGuard.close();
    }
    if (!finalized) {
        // Close all connections.  We don't need (or want) to do this
        // when finalized because we don't know what state the connections
        // themselves will be in.  The finalizer is really just here for CloseGuard.
        // The connections will take care of themselves when their own finalizers run.
        synchronized (mLock) {
            throwIfClosedLocked();//检测是否已经被关闭
            mIsOpen = false;//标识数据库连接池关闭
            //关闭数据库连接池中目前可用的链接（空闲的数据库链接、包括空闲的主链接）
            closeAvailableConnectionsAndLogExceptionsLocked();
            final int pendingCount = mAcquiredConnections.size();
            //任然有链接正在使用中
            if (pendingCount != 0) {
                Log.i(TAG, "The connection pool for " + mConfiguration.label
                        + " has been closed but there are still "
                        + pendingCount + " connections in use.  They will be closed "
                        + "as they are released back to the pool.");
            }
            //判断队列其他waiter是否状态有更新
            wakeConnectionWaitersLocked();
        }
    }
}
```

### 关闭数据库链接

```java
//SQLiteConnectionPool.java
// Can't throw.
@GuardedBy("mLock")
private void closeAvailableConnectionsAndLogExceptionsLocked() {
    //关闭可用的非主链接
    closeAvailableNonPrimaryConnectionsAndLogExceptionsLocked();
    //如果主链接可用，那么关闭主链接
    if (mAvailablePrimaryConnection != null) {
        closeConnectionAndLogExceptionsLocked(mAvailablePrimaryConnection);
        mAvailablePrimaryConnection = null;
    }
}
// Can't throw.
@GuardedBy("mLock")
private void closeAvailableNonPrimaryConnectionsAndLogExceptionsLocked() {
    //循环遍历可用的非主链接，进行数据库链接的关闭
    final int count = mAvailableNonPrimaryConnections.size();
    for (int i = 0; i < count; i++) {
        closeConnectionAndLogExceptionsLocked(mAvailableNonPrimaryConnections.get(i));
    }
    mAvailableNonPrimaryConnections.clear();
}
// Can't throw.
@GuardedBy("mLock")
private void closeConnectionAndLogExceptionsLocked(SQLiteConnection connection) {
    try {
        connection.close(); // might throw
        if (mIdleConnectionHandler != null) {
            mIdleConnectionHandler.connectionClosed(connection);
        }
    } catch (RuntimeException ex) {
        Log.e(TAG, "Failed to close connection, its fate is now in the hands "
                + "of the merciful GC: " + connection, ex);
    }
}
```

- 数据库关闭的时候引用次数自减，若引用次数归零则真正执行关闭数据库；
- 数据库关闭清楚引用后进行的是数据库连接池的关闭；
- 数据库的关闭先状态，然后关闭所有的空闲链接，使用中的连接回归连接池后被关闭；

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：

![振兴书城](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzEzMTk4NzktNjEyYzRjNjZkNDBjZTg1NS5qcGc?x-oss-process=image/format,png#pic_center)