---
title: Android数据库开源框架GreenDao分析
date: 2019-11-06 20:58:00
tags: [greendao]
categories: 数据库
description: "前段时间写Demo的时候遇到了数据库的并发问题 [Android数据库多线程并发操作异常] ，然后研究了一下 [Android中的数据库连接池]。在看相关代码的时候阅读了我们项目使用的数据库框架`GreenDao` 。哈哈，挺有意思的^ _ ^。
"
---

前段时间写Demo的时候遇到了数据库的并发问题 [Android数据库多线程并发操作异常](https://dandanlove.blog.csdn.net/article/details/102876043) ，然后研究了一下 [Android中的数据库连接池](https://dandanlove.blog.csdn.net/article/details/102942832) 。在看相关代码的时候阅读了我们项目使用的数据库框架`GreenDao` 。哈哈，挺有意思的^ _ ^。



# Android原始数据库的使用

## 创建数据库

```java
public class DatabaseHelper extends SQLiteOpenHelper {
    public static final String USER_TABLE_NAME = "user";
    public static final String USER_NAME = "username";
    public static final String AGE = "age";
    public DatabaseHelper(Context context) {
        super(context, "demo.db", null, 1);
    }
    @Override
    public void onCreate(SQLiteDatabase db) {
        //创建数据库
        db.execSQL("create table " + USER_TABLE_NAME + "(" + USER_NAME + " varchar(20) not null," + AGE + " varchar(10) not null" + ")");
    }
    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        //todo 更新数据库
    }
}
```

## 增删改查

```java
mDatabaseHelper = new DatabaseHelper(this);
mSqLiteDatabase = mDatabaseHelper.getWritableDatabase();
//插入
ContentValues values = new ContentValues();
values.put(mDatabaseHelper.USER_NAME, "老张");
values.put(mDatabaseHelper.AGE, "18岁");
mSqLiteDatabase.insert(mDatabaseHelper.USER_TABLE_NAME, null, values);
//删除
String conditions = "username=?";
String[] args = {"老张"};
mSqLiteDatabase.delete(DatabaseHelper.USER_TABLE_NAME, conditions, args);
//更新
ContentValues contentValues = new ContentValues();
contentValues.put(DatabaseHelper.AGE, "20岁");
String conditions = "age=?";
String[] valueStr = {"18岁"};
int affectNum = mSqLiteDatabase.update(DatabaseHelper.USER_TABLE_NAME, contentValues, conditions, valueStr);
//查询
Cursor cursor = mSqLiteDatabase.query(mDatabaseHelper.USER_TABLE_NAME, new String[] {DatabaseHelper.USER_NAME}, null, null, null, null, null, null);
if (cursor.moveToFirst()) {
    int count = cursor.getCount();
    for (int i = 0; i < count; i++) {
        String userName = cursor.getString(cursor.getColumnIndex(DatabaseHelper.USER_NAME));
        Log.i(TAG,  i + " --> " + userName);
    }
}
```

其内部实现为：

```java
//插入
SQLiteStatement.executeInsert
//更新、删除
SQLiteStatement.executeUpdateDelete
//查询
SQLiteCursor
```

Android原生的数据库操作默认是没有开启事务的，我们自己使用的时候可以开启。

## 性能优化

>1. 预编译SQL语句，重复的操作使用SQLiteStatement；
>2. 显示使用事务操作，做数据库更新修改操作时用事物能够提高写入性能；
>3. 查询数据优化，少用cursor.getColumnIndex；
>4. ContentValues的容量调整，内部是HashMap每次扩容进行double，最好能预估大小；
>5. 及时关闭Cursor；
>6. 耗时异步化；



Android平台上的数据库框架非常多，但是有一个共同特点就是基于对象关系映射`(ORM)`模型的。实现的目标也都是不需要写`SQL`语句，通过对对象的操作保存和操作数据。

`GreenDAO`是基于`AndroidSQLite`的最快、性能最强悍的数据库框架之一，因为他不涉及反射，靠的是代码辅助生成。



# GreenDao框架分析

[GreenDao3.0官网介绍](http://greenrobot.org/greendao/documentation/updating-to-greendao-3-and-annotations/)
[GreenDao文档](http://greenrobot.org/greendao/documentation/)
[GreenDao的Github仓库](https://github.com/greenrobot/greenDAO)

`GreenDao` 的使用在这里就不介绍了，上面的文档链接或者网络上的各种使用教程讲的都很详细。这里主要分析、对比一下`GreenDao框架` 在原生的基础之上进行了怎么样的封装。

在进行源码分析之前我们先说一下`GreenDao` 的优缺点，然后在下面的阅读过程中自己进行体会。

**优点：**

1. 库文件比较小，小于100K，编译时间低，而且可以避免65K方法限制；
2. 性能最大化(官方词汇)；
3. API 非常易用，提升了开发效率；
4. 最小的内存开销(这个没有实际测试过)；
5. 可支持原生语句，从Android原生 `SQLite` 过度到 `GreenDao` 相对还是比较容易；
6. 数据表结构和`Entity`数据结构`convert`支持，`Entity`的不同数据结构和数据库存储结构之间做一个灵活的转换；

**缺点:**

1. 不支持组合主键，这个很少用到。
2. 数据库表有关系时，在第一次请求上会有延迟并且之后的更新都不会自动同步，需要主动更新或者清楚缓存之后再请求，写的时候需要主动同时更新。当然这个也不算缺点，现在很多时候在数据库建表的时候很很少使用关联，要么建索引，要么查询的时候自己做关联。
3. 不支持`min`、`max`等函数，需要自己写`sql`执行`execSQL`。但这些都可以通过其他方式进行实现。
4. 不支持合并写，写入的时候判断已有这条数据那么进行更新，没有则实现插入。

## 数据库框架设计

文章前面简单的用代码进行数据库操作，我们可以从中看到一般在Android中操作数据库所需要的对象有：

- **SQLiteOpenHelper**：数据库的创建、更新的操作对象；

- **SQLiteDatabase**：执行数据的增删改查的操作对象；

- **SQLiteStatement**：`SQL` 执行的操作对象；

所以首先任何一个数据框架都需要对这几个对象做封装，其次就是对于`ORM模式` 的数据库框架来说对象和数据库之间映射的元数据`Entity` 的管理，以及对外提供建议操作数据的API的封装。

<center>![greendao-framework.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xMzE5ODc5LTA0MzI1YWI1OWY1MjVjNjEucG5n?x-oss-process=image/format,png#pic_center)</center>

## GreenDao底层封装

### Database

`StandardDatabase`实现`Database`接口，内部代理`SQLiteDatabase`。
- 提供数据库操作对象；
- 执行`SQL` 语句；
- 进行**事务**操作；
- 数据库的关闭和打开；

```java
public interface Database {
    Cursor rawQuery(String sql, String[] selectionArgs);
    void execSQL(String sql) throws SQLException;
    void beginTransaction();
    void endTransaction();
    boolean inTransaction();
    void setTransactionSuccessful();
    void execSQL(String sql, Object[] bindArgs) throws SQLException;
    DatabaseStatement compileStatement(String sql);
    boolean isDbLockedByCurrentThread();
    boolean isOpen();
    void close();
    Object getRawDatabase();
}
public class StandardDatabase implements Database {
    private final SQLiteDatabase delegate;
    /***部分代码省略***/
}
```

### DatabaseStatement

`StandardDatabaseStatement`实现`DatabaseStatement`，内部代理`SQLiteStatement`

- 数据对象的**增删改查**；

```java
public interface DatabaseStatement {
    void execute();
    long simpleQueryForLong();
    void bindNull(int index);
    long executeInsert();
    void bindString(int index, String value);
    void bindBlob(int index, byte[] value);
    void bindLong(int index, long value);
    void clearBindings();
    void bindDouble(int index, double value);
    void close();
    Object getRawStatement();
}
public class StandardDatabaseStatement implements DatabaseStatement {
    private final SQLiteStatement delegate;
    /***部分代码省略***/
}
```

### DatabaseOpenHelper

`DatabaseOpenHelper`内部`SQLiteDataBase`改为`StandardDatabase`进行代理。

- 数据库的创建；
- 数据库的更新以及版本管理；

```java
public abstract class DatabaseOpenHelper extends SQLiteOpenHelper {
    /***部分代码省略***/
    public Database getWritableDb() {
        return wrap(getWritableDatabase());
    }
    public Database getReadableDb() {
        return wrap(getReadableDatabase());
    }
    protected Database wrap(SQLiteDatabase sqLiteDatabase) {
        return new StandardDatabase(sqLiteDatabase);
    }
}
```

<center>![greendao.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xMzE5ODc5LTc3NGUxOTFjYjhhMGRiMDEucG5n?x-oss-process=image/format,png#pic_center)</center>

## GreenDao访问层

提供 `XXEntity` 数据模型对象、数据模型对象的`Properties`用来做每个字段的快速访问以及操作数据模型的`XXEntityDao`。

<center>![green-entity.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xMzE5ODc5LTVmN2Y4MTJmMDM5YWIyMzIucG5n?x-oss-process=image/format,png#pic_center)</center>

上图为`XXEntity`、`XXEntity.Properties`、`XXEntityDao` 的关系和类的相关功能。

除此之外还未查询提供了 [QueryBuilder](http://greenrobot.org/greendao/documentation/queries/) 方便查询，可进行`分页`和`偏移量`的查询设置。还有 [join](http://greenrobot.org/greendao/documentation/joins/) 方法可以进行 **表的关联查询** 。

```java
QueryBuilder<User> queryBuilder = userDao.queryBuilder();
queryBuilder.join(Address.class, AddressDao.Properties.userId)
     .where(AddressDao.Properties.Street.eq("Sesame Street"));
List<User> users = queryBuilder.list();
```

## GreenDao中间层

- 数据操作者`XXEntityDao` 的具体操作 `AbstractDao`；
- `XXEntityDao` 的管理者 `AbstractDaoSession`；
- 封装过程中的性能优化；

<center>![在这里插入图片描述](https://img-blog.csdnimg.cn/20191106205012741.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9kYW5kYW5sb3ZlLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70#pic_center)</center>

# GreenDao的优化

`GreenDao` 的优化主要体现在 `DaoConfig` 这个类中。

```java
public final class DaoConfig implements Cloneable {
    public final Database db;//数据库
    public final String tablename;//表名
    public final Property[] properties;//表的属性
    public final String[] allColumns;//表的字段名
    public final String[] pkColumns;//表主键字段名
    public final String[] nonPkColumns;//表的非主键字段名
    /** Single property PK or null if there's no PK or a multi property PK. */
    public final Property pkProperty;//表的主键属性，如果有多个或者没有那么为null
    public final boolean keyIsNumeric;//主键是否为数字类型，用来区别缓存容器类型，long和Object
    public final TableStatements statements;//sql语句预编译的Statement
    private IdentityScope<?, ?> identityScope;//对应数据对象的缓存容器
    public DaoConfig(Database db, Class<? extends AbstractDao<?, ?>> daoClass) {
        this.db = db;
        try {
            this.tablename = (String) daoClass.getField("TABLENAME").get(null);
            //读取对应表的字段属性
            Property[] properties = reflectProperties(daoClass);
            this.properties = properties;
            allColumns = new String[properties.length];
            List<String> pkColumnList = new ArrayList<String>();
            List<String> nonPkColumnList = new ArrayList<String>();
            Property lastPkProperty = null;
            //循环遍历所有字段，为pkProperty，allColumns，pkColumns，nonPkColumns赋值
            for (int i = 0; i < properties.length; i++) {
                Property property = properties[i];
                String name = property.columnName;
                allColumns[i] = name;
                if (property.primaryKey) {
                    pkColumnList.add(name);
                    lastPkProperty = property;
                } else {
                    nonPkColumnList.add(name);
                }
            }
            //字段赋值
            String[] nonPkColumnsArray = new String[nonPkColumnList.size()];
            nonPkColumns = nonPkColumnList.toArray(nonPkColumnsArray);
            String[] pkColumnsArray = new String[pkColumnList.size()];
            pkColumns = pkColumnList.toArray(pkColumnsArray);
            pkProperty = pkColumns.length == 1 ? lastPkProperty : null;
            //sql增删改查语句的预编译，Statement的缓存
            statements = new TableStatements(db, tablename, allColumns, pkColumns);
            if (pkProperty != null) {
                Class<?> type = pkProperty.type;
                keyIsNumeric = type.equals(long.class) || type.equals(Long.class) || type.equals(int.class)
                        || type.equals(Integer.class) || type.equals(short.class) || type.equals(Short.class)
                        || type.equals(byte.class) || type.equals(Byte.class);
            } else {
                keyIsNumeric = false;
            }

        } catch (Exception e) {
            throw new DaoException("Could not init DAOConfig", e);
        }
    }
    /***部分代码省略***/
}
```

- 增删改查的`SQL`的预编译的`Statement`的缓存：
`insertStatement`、`insertOrReplaceStatement`、`updateStatement`、`deleteStatement`和`countStatement` 。

```java
public class TableStatements {
    private final Database db;
    private final String tablename;
    private final String[] allColumns;
    private final String[] pkColumns;
    private DatabaseStatement insertStatement;
    private DatabaseStatement insertOrReplaceStatement;
    private DatabaseStatement updateStatement;
    private DatabaseStatement deleteStatement;
    private DatabaseStatement countStatement;
    private volatile String selectAll;
    private volatile String selectByKey;
    private volatile String selectByRowId;
    private volatile String selectKeys;
    /***部分代码省略***/
    public DatabaseStatement getInsertStatement() {
        if (insertStatement == null) {
            String sql = SqlUtils.createSqlInsert("INSERT INTO ", tablename, allColumns);
            DatabaseStatement newInsertStatement = db.compileStatement(sql);
            synchronized (this) {
                if (insertStatement == null) {
                    insertStatement = newInsertStatement;
                }
            }
            if (insertStatement != newInsertStatement) {
                newInsertStatement.close();
            }
        }
        return insertStatement;
    }
}
```

- 每次数据库操作都使用了事务提高的性能。
我们来看一个插入操作的执行过程：

```java
//AbstractDao.java
public long insert(T entity) {
    return executeInsert(entity, statements.getInsertStatement(), true);
}
private long executeInsert(T entity, DatabaseStatement stmt, boolean setKeyAndAttach) {
    long rowId;
    if (db.isDbLockedByCurrentThread()) {
        rowId = insertInsideTx(entity, stmt);
    } else {
        // Do TX to acquire a connection before locking the stmt to avoid deadlocks
        //开起事务
        db.beginTransaction();
        try {
            rowId = insertInsideTx(entity, stmt);
            //提交事务
            db.setTransactionSuccessful();
        } finally {
            //关闭事务
            db.endTransaction();
        }
    }
    if (setKeyAndAttach) {
        updateKeyAfterInsertAndAttach(entity, rowId, true);
    }
    return rowId;
}
//stmt绑定对应entity的value，并进行数据库写入
private long insertInsideTx(T entity, DatabaseStatement stmt) {
    synchronized (stmt) {
        if (isStandardSQLite) {
            SQLiteStatement rawStmt = (SQLiteStatement) stmt.getRawStatement();
            bindValues(rawStmt, entity);
            return rawStmt.executeInsert();
        } else {
            bindValues(stmt, entity);
            return stmt.executeInsert();
        }
    }
}
```

- 数据对象的缓存

```java
//提供两种配置，缓存和不缓存，在生成DaoSession的时候做的配置
public enum IdentityScopeType {
    Session, None
}
public class IdentityScopeObject<K, T> implements IdentityScope<K, T> {
    private final HashMap<K, Reference<T>> map;
    private final ReentrantLock lock;
    /***部分代码省略***/
}
public class IdentityScopeLong<T> implements IdentityScope<Long, T> {
    //LongHashMap内部是一个数组，对于索引做了优化
    private final LongHashMap<Reference<T>> map;
    private final ReentrantLock lock;
    /***部分代码省略***/
}
```

使用这个缓存有一个问题需要注意，如果直接使用`SQL`进行的操作是这里的缓存是不会进行更新的。但可以执行`refresh`更新，或者执行`clearIdentityScope`之后进行重新`load`。

# 数据库多线程并发操作

[Android数据库多线程并发操作异常](https://dandanlove.blog.csdn.net/article/details/102942832)


文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！