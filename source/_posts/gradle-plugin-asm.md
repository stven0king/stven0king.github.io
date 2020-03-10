---
title: Gradle自定义Plugin插件和ASM修改字节码
date: 2020-3-9 18:49:39
tags: [泛型]
categories: Java 
description: "在编译期间给某类方法添加代码，统计方法耗时。不用AspectJ可以么，不写注解可以么，动态配置方法可以么？学完这篇文章告诉你完全可以~！"
---



# 前言

先介绍一下 `AOP` 和 `ASM` 的概念，`AOP` 是一种**面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术**。和面向对象编程 的 `OOP` 相同。`ASM` 是一个框架可以看作 `AOP` 的工具，当然 `AOP` 也有其他工具，比如用的比较多的 `AspectJ` 、`Javassist` 、`Xposed` 和 `Dexposed` 等。

这里我们主要介绍一下在自定义`Plugin` 使用的 `ASM` :

> [ASM](https://asm.ow2.io/) 是一个通用的`Java`字节码操作和分析框架。它可以直接以二进制形式用于修改现有类或动态生成类。 `ASM`提供了一些常见的字节码转换和分析算法，可以从中构建定制的复杂转换和代码分析工具。 `ASM`提供与其他`Java字`节码框架类似的功能，但侧重于性能。因为它的设计和实现是尽可能的小和尽可能快，所以它非常适合在动态系统中使用（但当然也可以以静态方式使用，例如在编译器中）。

# 自定义plugin开发

`Gradle`从1.5开始，`Gradle`插件包含了一个叫`Transform`的API，这个API允许第三方插件在`class`文件转为为`dex`文件前操作编译好的`class`文件，这个API的目标是简化自定义类操作，而不必处理`Task`，并且在操作上提供更大的灵活性。并且可以更加灵活地进行操作。官方文档：http://google.github.io/android-gradle-dsl/javadoc/

## 创建插件

在 `AndroidStudio` 中创建一个纯净的`Module` ，删除 `res` 文件夹和 `AndroidManifest.xml` 文件。

**Gradle会默认在配置初始化前，编译buildSrc这个项目，可以是java、groovy、kotlin项目，并把项目配置到classpath下。**所以如果`Module` 命名为 `buildSrc` 那么不需要在 `Project` 级别的 `build.gradle` 文件中使用 `classpath` 引入，也不需要在 `app` 级别的 `build.gradle` 文件中使用 `apply plugin:` 进行应用。  

下面我们介绍的是自定义的 `Plugin` 插件，是编译之后需要引入项目的。

> 类继承`Plugin` 

```java
public class AmsPlugin implements Plugin<Project> {
    @Override
    public void apply(@NonNull Project project) {
        AppExtension appExtension = project.getExtensions().findByType(AppExtension.class);
        assert appExtension != null;
        //注册task任务
        appExtension.registerTransform(new AmsTransform(project));
    }
}
public class AmsTransform extends Transform{
  	/***部分代码省略***/
}
```

> `build.gradle` 文件配置

![uploadArchives.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xMzE5ODc5LWE4YTkxODBiODUzYzIxODkucG5n?x-oss-process=image/format,png#pic_center)


> 配置`properties`

在 `/src/main/resources/META-INF/gradle-plugins` 目录下创建`xxx.properties` 文件，这个 `xxx` 就是以后项目`build.gradle` 文件中需要`apply` 的插件名称。

```properties
implementation-class=com.tzx.ams.plugin.AmsPlugin
```

`com.tzx.ams.plugin.AmsPlugin` 就是插件对应的类。

![properties.png](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmppYW5zaHUuaW8vdXBsb2FkX2ltYWdlcy8xMzE5ODc5LTI4MmY0MTVhMzdkYzRlZTQucG5n?x-oss-process=image/format,png#pic_center)


## 生成插件

我们在执行 `uploadArchives` 的任务的时候就在我们对于的仓库生成了我们需要的插件。


## 在项目中引入插件

> 项目的根 `build.gradle`配置

```groovy
buildscript {
    repositories {
        maven {//添加repo本地仓库
            url uri("repo")
        }
        google()
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.3'
        //添加插件依赖
        classpath 'com.tzx.ams:ams-plugin:1.0.0'
    }
}
```

> 项目的 `build.gradle` 配置

```groovy
apply plugin: 'com.android.application'
apply plugin: 'amsplugin'
/***部分代码省略***/
```

# plugin自定义配置

> 配置类

```java
public class AmsConfig {
    //日志开关
    public boolean isDebug;
    //class包含str则不处理
    public String[] filterContainsClassStr;
    //class以str开头则不处理
    public String[] filterstartsWithClassStr;
    //拦截在这个文件中声明的class
    public String filterClassFile;
    public List<String> filterClassNameList;
    //需要进行注入的method
    public String amsMethodFile;
    //需要进行注入的method对应的tag
    public String amsMethodTag;
    public List<Pair<String, String>> amsMethodFileList;
}
```

> 创建`AmsConfig` 的类对象

```java
public class AmsPlugin implements Plugin<Project> {
    @Override
    public void apply(@NonNull Project project) {
        AppExtension appExtension = project.getExtensions().findByType(AppExtension.class);
        assert appExtension != null;
        //注册优先于task任务的添加
        project.getExtensions().create("AmsConfig", AmsConfig.class);
        appExtension.registerTransform(new AmsTransform(project));
    }
}
```

> `build.gradle` 文件中定义配置（如果gradle中没有定义，那么会反射构造一个对象）

```groovy
apply plugin: 'com.android.application'
apply plugin: 'amsplugin'
AmsConfig{
    isDebug = true
    filterContainsClassStr = ["R.class", ".R\$"]
    filterstartsWithClassStr = ["android"]
    filterClassFile = "amsfilterclass.text"//build.gradle相同目录级别的文件名
    amsMethodFile = "amsmethods.text"//build.gradle相同目录级别的文件名
    amsMethodTag = "TEST"
}
```

> 获取`gradle` 中定义的配置

```java
AmsConfig amsConfig = (AmsConfig) this.project.getExtensions().getByName(AmsConfig.class.getSimpleName());
//配置文件的获取
String projectDri = this.project.getProjectDir().getAbsolutePath()
String fileName = projectDri + File.separatorChar + amsConfig.filterClassFile;
```

这个 `amsConfig` 在使用的时候不用判空，如果调用`project.getExtensions().create` 添加了那么就会反射构造出这个对象。如果`gradle` 文件中定义了，那么这个对象就会被赋与相应的属性值。

如果在获取的时候，没有被`create`那么就会跑出一下异常：

```log
* Where:
Build file '/Users/tanzx/AndroidStudioWorkSpace/GitHub/AmsDemo/app/build.gradle' line: 3
* What went wrong:
A problem occurred evaluating project ':app'.
> Could not find method AmsConfig() for arguments [build_5n6idkxwtmzfflm5k30ynjblo$_run_closure1@2cf1f355] on project ':app' of type org.gradle.api.Project
```

[代码传送门，拿走不谢^^！](https://github.com/stven0king/AndroidPluginStudy)



文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！

想阅读作者的更多文章，可以查看我 [个人博客](http://dandanlove.com/) 和公共号：

![振兴书城](https://imgconvert.csdnimg.cn/aHR0cDovL3VwbG9hZC1pbWFnZXMuamlhbnNodS5pby91cGxvYWRfaW1hZ2VzLzEzMTk4NzktNjEyYzRjNjZkNDBjZTg1NS5qcGc?x-oss-process=image/format,png#pic_center)