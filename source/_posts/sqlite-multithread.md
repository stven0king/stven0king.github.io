---
title: Android数据库多线程并发操作异常
date: 2019-11-06 20:55:00
tags: [数据库,多线程,并发]
categories: 数据库
description: "在我们做项目的过程中经常会有多线程异步处理的情况，那么`Android`中多线程操作数据我们一般会遇到什么样的问题？"
---

在我们做项目的过程中经常会有多线程异步处理的情况，那么`Android`中多线程操作数据我们一般会遇到什么样的问题？

# 多个数据库对象执行并发

指由不同的`SQLiteOpenHelper`打开的相同数据库对象，默认`enableWriteAheadLogging=false`。

## 多线程

**单进程和多进程结果一样。**

- 同时进行数据库的读操作不会产生任何问题；

- 如果都需要创建表，那么多次创建可能会出现问题；

```log
android.database.sqlite.SQLiteException：table key_value_alerady exits (code 1)
```

- 如果表已经创建，那么同时进行读写操作；

```log
14:48:41.039#[androidcode@]#29329#E#SQLiteDatabase #Error inserting TITLE=1572590918524
android.database.sqlite.SQLiteDatabaseLockedException: database is locked (code 5)
```

因为Android的数据库默认配置是不支持多个多线程读写的，`enableWriteAheadLogging=true` 可以进行多线程的读写。



## 一个数据库对象执行并发


多线程操作问题：已经打开的数据库在进行读写的时候被其他地方调用了`close`关闭了数据库。
```log
java.lang.IllegalStateException: attempt to re-open an already-closed object
```

同一个`SQLiteOpenHelper`实例获取的`database`是相同的，多在线程的情况下应该进行统一的`open`和`close`，所以一般都通过**单例**去管理`database` 的打开和关闭。



# 数据库连接池

如果 `SQLiteOpenHelper` 使用的是单例，`SQLiteDatabase` 对` CRUD` 操作都是从同一个连接池中获取连接. 默认情况下, 连接池中只有一条主连接, 所以同一时间只能进行一项操作，多线程读写几乎是无用功；

> `enableWriteAheadLogging() `方法可以使得多链接并发查询可行，但默认没有开启该功能, 该方法会根据配置在连接池中创建多条连接；


为什么`Android`数据库链接池默认只有一条链接，请阅读 [Android中的数据库连接池](https://dandanlove.blog.csdn.net/article/details/102876043) 这篇文章~！


文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：

<center>
![振兴书城](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzEzMTk4NzktNjEyYzRjNjZkNDBjZTg1NS5qcGc?x-oss-process=image/format,png#pic_center)</center>