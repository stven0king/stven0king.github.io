---
title: Android类加载之PathClassLoader和DexClassLoader
date: 2017-2-23 15:08:02
tags: [PathClassLoader,DexClassLoader]
categories: Android
description: " 上一章节我们讨论了JVM的类加载机制，那么Android中的类加载是什么样的呢？两者有什么相同和不同？"
---

![北京的初雪.jpg](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzEzMTk4NzktMTBjNDA5NjhiNmM2ZWRjZi5qcGc?x-oss-process=image/format,png#pic_center =500x400)
# 第二次修改
任何结论在没有经过实际检验之前都不能够确定一定没问题。三年前写的文章回过头来发现有些部分内容是有问题的（PS：这的确比较尴尬），再次对结果进行验证之后重新修改了之前的结论。幸亏文章阅读量不是很多，希望被误导的同学能够在其他地方得到正确结论。

----------

上一篇文章 [自定义ClassLoader和双亲委派机制](http://www.jianshu.com/p/a8371d26f848) 讲述了 `JVM` 中的类的加载机制，`Android` 也是类 `JVM` 虚拟机那么它的类加载机制是什么呢，我们来探究一下(PS：文章源码为 `Android5.1` )。

# 前言
`Android` 的 `Dalvik` 虚拟机和 `Java` 虚拟机的运行原理相同都是将对应的 `java` 类加载在内存中运行。而 `Java` 虚拟机是加载 `class` 文件，也可以将一段二进制流通过 `defineClass` 方法生产 `Class` 进行加载（PS: [自定义ClassLoader和双亲委派机制](http://www.jianshu.com/p/a8371d26f848) 文章后面的自定义类加载器就是通过这种方式实现的）。`Dalvik` 虚拟机加载的 `dex` 文件。`dex` 文件是 `Android` 对与 `Class` 文件做的优化，以便于提高手机的性能。可以想象 `dex` 为 `class` 文件的一个压缩文件。`dex` 在 `Android` 中的加载和 `class` 在 `jvm` 中的相同都是基于双亲委派模型，都是调用`ClassLoader` 的 `loadClass` 方法加载类。



# Android系统中类加载的双亲委派机制
- `Android5.1` 源码中 `ClassLoader` 的 `loadClass` 方法
```java
protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
        Class<?> clazz = findLoadedClass(className);
        if (clazz == null) {
            ClassNotFoundException suppressed = null;
            try {
                //先让父类加载器加载
                clazz = parent.loadClass(className, false);
            } catch (ClassNotFoundException e) {
                suppressed = e;
            }
            //当所有父类节点的类加载器都没有找到该类时，当前加载器调用findClass方法加载。
            if (clazz == null) {
                try {
                    clazz = findClass(className);
                } catch (ClassNotFoundException e) {
                    e.addSuppressed(suppressed);
                    throw e;
                }
            }
        }
```

- 想要动态加载类，可以用 [自定义ClassLoader和双亲委派机制](http://www.jianshu.com/p/a8371d26f848) 中自定义 `ClassLoader` 的方法加载自己定义的 `class` 文件么？看看 `Android` 源码中的` ClassLoader ` 的 `findClass` 方法：

```java
protected Class<?> findClass(String className) throws ClassNotFoundException {
        throw new ClassNotFoundException(className);
}
```

这个方法直接抛出了 `ClassNotFoundException` 异常，所以在 `Android` 中想通过这种方式实现类的加载时不行的。

# Android系统中的类加载器
- `Android` 系统屏蔽了 `ClassLoader` 的 `findClass` 加载方法，那么它自己的类加载时通过什么样的方式实现的呢？
 - `Android` 系统中有两个类加载器分别为 `PathClassLoader` 和 `DexclassLoader`。
 - `PathClassLoader` 和 `DexClassLoader` 都是继承与`BaseDexClassLoader`，`BaseDexClassLoader` 继承与 `ClassLoader`。

## 提出问题
在这里我们先提一个问题 `Android` 为什么会将自己的类加载器派生出两个不同的子类，它们各自有什么用？

## BaseDexClassLoader类加载
- 作为 `ClassLoader` 的子类，复写了父类的 `findClass` 方法。
```java
@Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        //在自己的成员变量DexPathList中寻找，找不到抛异常
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }
```

- `DexPathList` 的 `findClass` 方法

```java
public Class findClass(String name, List<Throwable> suppressed) {
        //循环便利成员变量dexElements，调用DexFile.loadClassBinaryName加载class
        for (Element element : dexElements) {
            DexFile dex = element.dexFile;

            if (dex != null) {
                Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
                if (clazz != null) {
                    return clazz;
                }
            }
        }
        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }
```

通过以上两段代码我们可以看出，虽然 `Android` 中的 `ClassLoader` 的`findClass` 方法的实现被取消了，但是 `ClassLoader` 的基类 `BaseDexClassLoader` 实现了 `findClass` 方法取加载指定的 `Class`。

## PathClassLoader和DexClassLoader比较

- `PathClassLoader`

```java
public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }
    public PathClassLoader(String dexPath, String libraryPath,
            ClassLoader parent) {
        super(dexPath, null, libraryPath, parent);
    }
}
```

- `DexClassLoader`

```java
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
}
```

- `BaseDexClassLoader` 的构造函数

```java
    /**
     * 构造方法
     * @param dexPath 包含类和资源的jar/apk文件列表，Android中使用“:”拆分
     * @param optimizedDirectory 优化的dex文件所在的目录，可以为空；
     * @param libraryPath 动态库路径，可以为空；
     * @param parent 父类加载器
     */
public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }
```

> - `dexPath`：指定的是`dex`文件地址，多个地址可以用":"进行分隔
>  - `optimizedDirectory`：制定输出 `dex` 优化后的 `odex` 文件，可以为`null`
>  - `libraryPath` ：动态库路径（将被添加到 `app` 动态库搜索路径列表中）
>  - `parent` ：制定父类加载器，以保证双亲委派机制从而实现每个类只加载一次。

可以看出 `PathClassLoader` 和 `DexClassLoader` 的区别就在于构造函数中 `optimizedDirectory` 这个参数。`PathClassLoader` 中 `optimizedDirectory` 为 `null`，`DexClassLoader` 中为 `new File(optimizedDirectory)`。

## optimizedDirectory 的作用

 `BaseDexClassLoader` 的构造函数利用 `optimizedDirectory` 创建了一个`DexPathList`，看看 `DexPathList` 中 `optimizedDirectory`:

```java
public DexPathList(ClassLoader definingContext, String dexPath,
        String libraryPath, File optimizedDirectory) {
    /******部分代码省略******/
    this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                       suppressedExceptions);
    /******部分代码省略******/
}
private static Element[] makeDexElements(ArrayList<File> files, File optimizedDirectory,
                                         ArrayList<IOException> suppressedExceptions) {
   /******部分代码省略******/
    for (File file : files) {
        /******部分代码省略******/
        if (file.isDirectory()) {
           /******部分代码省略******/
        } else if (file.isFile()){
            if (name.endsWith(DEX_SUFFIX)) {
                // Raw dex file (not inside a zip/jar).
                try {
                    dex = loadDexFile(file, optimizedDirectory);
                } catch (IOException ex) {
                    System.logE("Unable to load dex file: " + file, ex);
                }
            } else {
                zip = file;
                try {
                    dex = loadDexFile(file, optimizedDirectory);
                } catch (IOException suppressed) {
                    suppressedExceptions.add(suppressed);
                }
            }
        } else {
            System.logW("ClassLoader referenced unknown path: " + file);
        }
        if ((zip != null) || (dex != null)) {
            elements.add(new Element(file, false, zip, dex));
        }
    }
    return elements.toArray(new Element[elements.size()]);
}

private static DexFile loadDexFile(File file, File optimizedDirectory)
        throws IOException {
    if (optimizedDirectory == null) {
        return new DexFile(file);
    } else {
        String optimizedPath = optimizedPathFor(file, optimizedDirectory);
        return DexFile.loadDex(file.getPath(), optimizedPath, 0);
    }
}
```

从这里可以看出 `optimizedDirectory` 不同生产的 `DexFile` 对象不同，我们继续看看 `optimizedDirectory` 在 `DexFile` 中的作用：

```java
public DexFile(File file) throws IOException {
    this(file.getPath());
}
/**
 * 从给定的File对象打开一个DEX文件。这通常是一个ZIP/JAR 文件，其中包含“classes.dex”。
 * VM将在/data/dalvik-cache中生成相应文件的名称，然后打开它，如果允许系统权限，可能会首先创建或更新它。不要在/data/dalvik-cache中传递文件名，因为预期该命名文件为原始状态（pre-dexopt）。
 * @param fileName 引用实际DEX文件的File对象
 * @throws IOException 找不到文件或*打开文件缺少访问权限，回抛出io异常
 */
public DexFile(String fileName) throws IOException {
    mCookie = openDexFile(fileName, null, 0);
    mFileName = fileName;
    guard.open("close");
    //System.out.println("DEX FILE cookie is " + mCookie + " fileName=" + fileName);
}

/**
 * 打开一个DEX文件。返回的值是VM cookie
 * @param sourceName Jar或APK文件包含“ classes.dex”。
 * @param outputName 包含优化形式的DEX数据的文件。
 * @param flags 启用可选功能。
 */
private DexFile(String sourceName, String outputName, int flags) throws IOException {
    if (outputName != null) {
        try {
            String parent = new File(outputName).getParent();
            if (Libcore.os.getuid() != Libcore.os.stat(parent).st_uid) {
                throw new IllegalArgumentException("Optimized data directory " + parent
                        + " is not owned by the current user. Shared storage cannot protect"
                        + " your application from code injection attacks.");
            }
        } catch (ErrnoException ignored) {
            // assume we'll fail with a more contextual error later
        }
    }
    mCookie = openDexFile(sourceName, outputName, flags);
    mFileName = sourceName;
    guard.open("close");
    //System.out.println("DEX FILE cookie is " + mCookie + " sourceName=" + sourceName + " outputName=" + outputName);
}

static public DexFile loadDex(String sourcePathName, String outputPathName,
    int flags) throws IOException {
    return new DexFile(sourcePathName, outputPathName, flags);
}

//打开dex的native方法，/art/runtime/native/dalvik_system_DexFile.cc
private static native long openDexFileNative(String sourceName, String outputName, int flags);
//打开一个DEX文件。返回的值是VM cookie。 失败时，将引发IOException。
private static long openDexFile(String sourceName, String outputName, int flags) throws IOException {
    // Use absolute paths to enable the use of relative paths when testing on host.
    return openDexFileNative(new File(sourceName).getAbsolutePath(),
                                (outputName == null) ? null : new File(outputName).getAbsolutePath(),
                                flags);
}
```

从注释当中就可以看到 `new DexFile(file)` 的dex输出路径只能为 `/data/dalvik-cache`，而 `DexFile.loadDex()` 的 `dex` 输出路径为自己输入的`optimizedDirectory` 路径。

![dalvik-cache.jpg](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzEzMTk4NzktODk3NTY3NzQzZmMzMjVhZi5qcGc?x-oss-process=image/format,png#pic_center =300x500)

## 解决疑问
我们在文章开始提出的问题就这样一步步得到了答案。
>- `DexClassLoader`：能够加载 `jar/apk/dex` ，也可以指定对 `dex` 优化后的 `odex` 的输出文件目录；
>- `PathClassLoader`：只能够加载 `jar/apk/dex` ，但它的 `optimizedDirectory` 没法自己设定;

所以 `Android` 系统默认的类加载器为 `PathClassLoader`，而`DexClassLoader` 可以像 `JVM` 的 `ClassLoader` 一样提供动态加载。

## android 8.0上的修改

[android-26版本的BaseDexClassLoader.java](https://www.androidos.net.cn/android/8.0.0_r4/xref/libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java)

```java
/**
 * Constructs an instance.
 * Note that all the *.jar and *.apk files from {@code dexPath} might be
 * first extracted in-memory before the code is loaded. This can be avoided
 * by passing raw dex files (*.dex) in the {@code dexPath}.
 *
 * @param dexPath the list of jar/apk files containing classes and
 * resources, delimited by {@code File.pathSeparator}, which
 * defaults to {@code ":"} on Android.
 * @param optimizedDirectory this parameter is deprecated and has no effect
 * @param librarySearchPath the list of directories containing native
 * libraries, delimited by {@code File.pathSeparator}; may be
 * {@code null}
 * @param parent the parent class loader
 */
public BaseDexClassLoader(String dexPath, File optimizedDirectory,
        String librarySearchPath, ClassLoader parent) {
    super(parent);
    this.pathList = new DexPathList(this, dexPath, librarySearchPath, null);

    if (reporter != null) {
        reporter.report(this.pathList.getDexPaths());
    }
}
```

重点关注 `@param optimizedDirectory this parameter is deprecated and has no effect` ，表明在 `Android26` 上 `optimizedDirectory` 已经被弃用。
所以在 `Android26` 之后 `PathClassLoader` 和 `DexClassLoader` 已经没有了区别。

[从JVM到Dalivk再到ART（class,dex,odex,vdex,ELF）](https://dandanlove.blog.csdn.net/article/details/102620708) 这篇文章中讲述了 `Android26`  中其他 `dex` 相关的优化。

# 总结
- `ClassLoader` 的 `loadClass` 方法保证了双亲委派机。
- `BaseDexClassLoader` 提供了两种派生类使我们可以加载自定义类。

另外还有一个问题自己没太搞清楚，**默认的optimizedDirectory** 是哪个路径？
`data/app` 和 `data/dalvik-cache` 下面都没有我加载的外部的 `dex` 文件，有谁找到了结果可以分享一下。

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！