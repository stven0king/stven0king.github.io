---
title: Java版的7种单例模式
date: 2019-09-30 14:59:00
tags: [单例模式]
categories: 设计模式
description: "今天看到某一篇文章的一句话单例DCL前面加V。就这句话让我把 `单例模式` 又仔细看了一遍。Java中的单例模式是我们一直且经常使用的设计模式之一，大家都很熟悉，所以这篇文章仅仅做我自己记忆。"
---


# 前言

<center>![宗介-波妞](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xMzE5ODc5LTljZjJhNGI0ZGVkMDYzZjEuanBlZw?x-oss-process=image/format,png)</center>

今天看到某一篇文章的一句话 `单例DCL` 前面加 `V` 。就这句话让我把 `单例模式` 又仔细看了一遍。

`Java` 中的 `单例模式` 是我们一直且经常使用的设计模式之一，大家都很熟悉，所以这篇文章仅仅做我自己记忆。

单例模式（Singleton Pattern）是 Java 中最简单的设计模式之一。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。

`单例模式` 涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。

>- 单例类只能有一个实例。
>- 单例类必须自己创建自己的唯一实例。
>- 单例类必须给所有其他对象提供这一实例。

## Java版七种单例模式写法

## 一：懒汉，线程不安全
> 这种写法lazy loading很明显，但是致命的是在多线程不能正常工作。
```java
public class Singleton{
    private static Singleton instance;
    private Singleton(){};
    public static Singleton getInstance(){
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

## 二：懒汉，线程安全
> 这种写法能够在多线程中很好的工作，而且看起来它也具备很好的lazy loading，但是，遗憾的是，效率很低，99%情况下不需要同步。
```java
public class Singleton{
    private static Singleton instance;
    private Singleton(){};
    public static synchronized Singleton getInstance(){
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

## 三：饿汉
> 这种方式基于classloder机制避免了多线程的同步问题，不过，instance在类装载时就实例化，虽然导致类装载的原因有很多种，在单例模式中大多数都是调用getInstance方法， 但是也不能确定有其他的方式（或者其他的静态方法）导致类装载，这时候初始化instance显然没有达到lazy loading的效果。
```java
public class Singleton{
    private static Singleton instance = new Singleton();
    private Singleton(){};
    public static Singleton getInstance(){
        return instance;
    }
}
```

## 四：饿汉，变种
> 表面上看起来差别挺大，其实更第三种方式差不多，都是在类初始化即实例化instance。
```java
public class Singleton{
    private static Singleton instance = null;
    private Singleton(){};
    static {
        instance = new Singleton();
    }
    public static Singleton getInstance(){
        return instance;
    }
}
```

## 五：静态内部类
> 这种方式同样利用了classloder的机制来保证初始化instance时只有一个线程，它跟第三种和第四种方式不同的是（很细微的差别）：第三种和第四种方式是只要Singleton类被装载了，那么instance就会被实例化（没有达到lazy loading效果），而这种方式是Singleton类被装载了，instance不一定被初始化。因为SingletonHolder类没有被主动使用，只有显示通过调用getInstance方法时，才会显示装载SingletonHolder类，从而实例化instance。想象一下，如果实例化instance很消耗资源，我想让他延迟加载，另外一方面，我不希望在Singleton类加载时就实例化，因为我不能确保Singleton类还可能在其他的地方被主动使用从而被加载，那么这个时候实例化instance显然是不合适的。这个时候，这种方式相比第三和第四种方式就显得很合理。

```java
public class Singleton{
    private static class SingletonHolder{
        private static final Singleton INSTANCE = new Singleton();
    }
    private Singleton(){};
    public static Singleton getInstance(){
        return SingletonHolder.INSTANCE;
    }
}
```

似乎静态内部类看起来已经是最完美的方法了，其实不是，可能还存在反射攻击或者反序列化攻击。且看如下代码：

```java
public static void main(String[] args) throws Exception {
    Singleton singleton = Singleton.getInstance();
    Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
    constructor.setAccessible(true);
    Singleton newSingleton = constructor.newInstance();
    System.out.println(singleton == newSingleton);
}
```

## 六：枚举
> 这种方式是Effective Java作者Josh Bloch 提倡的方式，最佳的单例实现模式就是枚举模式。利用枚举的特性，让JVM来帮我们保证线程安全和单一实例的问题，而且还能防止反序列化重新创建新的对象。除此之外，写法还特别简单。

```java
public enum Singleton {
    INSTANCE;
    public void get() {
        System.out.println("");
    }
}
```
> 通过反编译我们看到，枚举是在 `static` 块中进行的对象的创建。
```java
public final class com.loadclass.test.Singleton extends java.lang.Enum<com.loadclass.test.Singleton> {
  public static final com.loadclass.test.Singleton INSTANCE;

  public static com.loadclass.test.Singleton[] values();
    Code:
       0: getstatic     #1                  // Field $VALUES:[Lcom/loadclass/test/Singleton;
       3: invokevirtual #2                  // Method "[Lcom/loadclass/test/Singleton;".clone:()Ljava/lang/Object;
       6: checkcast     #3                  // class "[Lcom/loadclass/test/Singleton;"
       9: areturn

  public static com.loadclass.test.Singleton valueOf(java.lang.String);
    Code:
       0: ldc           #4                  // class com/loadclass/test/Singleton
       2: aload_0
       3: invokestatic  #5                  // Method java/lang/Enum.valueOf:(Ljava/lang/Class;Ljava/lang/String;)Ljava/lang/Enum;
       6: checkcast     #4                  // class com/loadclass/test/Singleton
       9: areturn

  public void get();
    Code:
       0: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #8                  // String
       5: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return

  static {};
    Code:
       0: new           #4                  // class com/loadclass/test/Singleton
       3: dup
       4: ldc           #10                 // String INSTANCE
       6: iconst_0
       7: invokespecial #11                 // Method "<init>":(Ljava/lang/String;I)V
      10: putstatic     #12                 // Field INSTANCE:Lcom/loadclass/test/Singleton;
      13: iconst_1
      14: anewarray     #4                  // class com/loadclass/test/Singleton
      17: dup
      18: iconst_0
      19: getstatic     #12                 // Field INSTANCE:Lcom/loadclass/test/Singleton;
      22: aastore
      23: putstatic     #1                  // Field $VALUES:[Lcom/loadclass/test/Singleton;
      26: return
}
```

## 七：双重校验锁（ DCL：double-checked locking)

```java
public class Singleton {
    // jdk1.6及之后，只要定义为private volatile static SingleTon instance 就可解决DCL失效问题。
    // volatile确保instance每次均在主内存中读取，这样虽然会牺牲一点效率，但也无伤大雅。
    // volatile可以保证即使java虚拟机对代码执行了指令重排序，也会保证它的正确性。
    private volatile static Singleton singleton;
    private Singleton(){};
    public static Singleton getSingleton() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

### DCL及解决办法&说明：
针对延迟加载法的同步实现所产生的性能低的问题，可以采用DCL，即双重检查加锁（Double Check Lock）的方法来避免每次调用getInstance()方法时都同步。

Double-Checked Locking看起来是非常完美的。但是很遗憾，根据Java的语言规范，上面的代码是不可靠的。
 出现上述问题, 最重要的2个原因如下:
>- 编译器优化了程序指令, 以加快cpu处理速度.
>- 多核cpu动态调整指令顺序, 以加快并行运算能力.

 问题出现的顺序:
>- 线程A, 发现对象未实例化, 准备开始实例化
>- 由于编译器优化了程序指令, 允许对象在构造函数未调用完前, 将共享变量的引用指向部分构造的对象, 虽然对象未完全实例化, 但已经不为null了.
>- 线程B, 发现部分构造的对象已不是null, 则直接返回了该对象.

 解决办法:
    可以将instance声明为volatile，即 private volatile static Singleton instance
    在线程B读一个volatile变量后，线程A在写这个volatile变量之前，所有可见的共享变量的值都将立即变得对线程B可见。

# 总结：

 >- 如果单例由不同的类装载器装入，那便有可能存在多个单例类的实例。假定不是远端存取，例如一些servlet容器对每个servlet使用完全不同的类  装载器，这样的话如果有两个servlet访问一个单例类，它们就都会有各自的实例。

```java
private static Class getClass(String classname) throws ClassNotFoundException {
    ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    if (classLoader == null) {
        classLoader = Singleton.class.getClassLoader();
    }
    return (classLoader.loadClass(classname));
}
```

 >- 如果Singleton实现了java.io.Serializable接口，那么这个类的实例就可能被序列化和复原。不管怎样，如果你序列化一个单例类的对象，接下来复原多个那个对象，那你就会有多个单例类的实例。

```java
public class Singleton implements Serializable {
    public static Singleton INSTANCE = new Singleton();
    private Singleton(){}
    //ObjectInputStream.readObject调用
    private Object readResolve() {
        return INSTANCE;
    }
}
```

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！