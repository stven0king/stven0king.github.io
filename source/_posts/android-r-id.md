---
title: Android中R文件ID值
date: 2021-01-11 17:29:58
tags: [R文件]
categories: Android
description: "前端时间在学习Qigsaw相关的源码，思考到一个问题。动态加载的feature包里的资源id是否会与主包中的资源id冲突。因为主包的apk文件不一定是和加载的feature包是一起打包生成的，feature包是可以进行升级的。查看Qigsaw编译脚本对old.apk进行增量编译feature的时候也没有发现对R文件做特殊的处理。 "
---


# Android中R文件ID值

![](https://img-blog.csdnimg.cn/img_convert/017dd03576db45793220ffc5655369da.png#pic_center)

## 前言

前端时间在学习 `Qigsaw` 相关的源码，思考到一个问题。动态加载的`feature` 包里的 `资源id` 是否会与主包中的 `资源id` 冲突。因为主包的 `apk` 文件不一定是和加载的`feature` 包是一起打包生成的，`feature` 包是可以进行升级的。查看 ` Qigsaw`编译脚本对 `old.apk` 进行增量编译 `feature` 的时候也没有发现对 `R文件` 做特殊的处理。

那么 `Android` 怎么保证两次编译出的 `feature` 包中的 `资源id` 不与主包中的冲突呢？我们带着问题阅读文章进行答案的探索（`Android`中资源属于一个大模块，我们本地只讨论其中与`R文件`相关的部分）。



>- 不同的 `android-gradle` 版本可能对 `R文件` 的格式以及生成目录会略作修改，本文只选了两个版本做参考。
>- 同步的反编译工具反编译出来的结果不仅相关，我们主要以 `AndroidStudio` 结果为主。



## 知识回顾

### 在代码中访问资源

您可以以方法参数的形式传递资源 ID，进而在代码中使用资源。例如，您可以设置一个 `ImageView`，从而借助 `setImageResource()` 使用 `res/drawable/myimage.png` 资源：

```java
ImageView imageView = (ImageView) findViewById(R.id.myimageview);
imageView.setImageResource(R.drawable.myimage);
```

您还可利用 `Resources` 中的方法检索个别资源，并且您可通过 `getResources()` 获得该资源的实例。

### 语法

以下是在代码中引用资源的语法：

```
[<package_name>.]R.<resource_type>.<resource_name>
```

- *`<package_name>`* 是资源所在包的名称（如果引用的资源来自您自己的资源包，则不需要）。
- *`<resource_type>`* 是资源类型的 `R` 子类。
- *`<resource_name>`* 是不带扩展名的资源文件名，或 XML 元素中的 `android:name` 属性值（若资源是简单值）。

其实到这里我们已经解决了我们阅读本文的目的。

主包的 `资源文件ID` 和 `feature` 包的 `资源文件ID` 值是由于 `<package_name>` 不一致导致最后 `ID` 值不会相同。

有时间的小伙伴可以继续往下阅读，后面更精彩。

## R文件

> 主工程R文件结构

![R.png](https://img-blog.csdnimg.cn/img_convert/b34f2d9feb43e10e61b869541333f0ca.png#pic_center)


> 插件的R文件结构

![Qigsaw-feature-R.png](https://img-blog.csdnimg.cn/img_convert/87d98b315d092e922acaf45e7bfe0e8f.png#pic_center)


R文件中每个`资源ID值`一共4个字段，由三部分组成：`PackageId+TypeId+EntryId`

* `PackageId`：是包的`Id`值，`Android` 中如果第三方应用的话，这个默认值是 `0x70` ，系统应用的话就是 `0x01` ，插件的话那么就是给插件分配的`id值`，占用一个字节。
* `TypeId`: 是资源的类型`Id`值，一般 `Android` 中有这几个类型：`attr`，`drawable`，`layout`，`anim`，`raw`，`dimen`，`string`，`bool`，`style`，`integer`，`array`，`color`，`id`，`menu` 等。【应用程序所有模块中的资源类型名称，按照字母排序之后。值是从1开支逐渐递增的，而且顺序不能改变（每个模块下的`R文件`的相同资源类型`id值`相同)。比如：`anim=0x01`占用1个字节，那么在这个编译出的所有R文件中`anim` 的值都是 `0x01`】
* `EntryId`:是在具体的类型下资源实例的`id`值，从0开始，依次递增，他占用四个字节。

### Lib库的R文件

> `com.android.tools.build:gradle:3.2.0`
>
> `releasePath` :`/build/generated/not_namespaced_r_class_sources_release_generateReleaseRFile/out/R.java` (`Debug包与其对应`)

![3.2.0-module-lib-r.png](https://img-blog.csdnimg.cn/img_convert/f9915699b98300df2d399a40dab7cec2.png#pic_center)


> `com.android.tools.build:gradle:3.4.1`
>
> `debugPath` :`/build/intermediates/complile_only_not_namespaced_r_class_jar_debug_generateDebugRFile/R.jar` (`Release包与其对应`)

![3.4.1-module-lib-r.png](https://img-blog.csdnimg.cn/img_convert/fc6a8feaf6b508c61764b461bf56721b.png#pic_center)


我们可以看到 `Lib` 中的 `R` 文件都是 `public static` 不是常量。 这和我们刚开始查看的 `主工程` 以及 `插件` 的 `R文件` 相比缺少了 `final` 关键词的修饰。

> `Lib` 库中资源`id` 的使用为引用类型；

![module-lib-r-source-layout.png](https://img-blog.csdnimg.cn/img_convert/572f73aa94dd43fe228d6cf17c8a511d.png#pic_center)


（PS:至于`资源ID`为什么不是常量，使用为引用类型，我们继续往后看~！）

### AAR中的R文件

![AAR-R.png](https://img-blog.csdnimg.cn/img_convert/937bf55d2dbe0a47ab2d4f5eefc9a9f4.png#pic_center)


我们可以看到打包了的 `Lib/Module` 为 `arr包` 之后，我们是找不到 `R.java` 文件的。只有一个 `R.txt`。

> `aar` 依赖库中资源`id` 的使用为引用类型；

![module-lib-r-jar-layout.png](https://img-blog.csdnimg.cn/img_convert/f56b1f4531d5d582e386c2ecabda14c7.png#pic_center)


### 依赖库R文件的生成

- 源码依赖的 `Lib` 库的 `R` 文件中的 `ID` 不是常量；
- `aar` 依赖的 `Lib` 库的` R` 文件是 `.txt` 文件；
- 源码依赖的 `Lib` 库和 `aar` 依赖的 `Lib` 库中的 `资源ID` 的使用都是引用类型；

源码依赖的 `Lib` 库和 `aar` 依赖的 `Lib` 库中的 `R` 文件的相关产物都是由于：如果依赖库的 `R` 文件中的 `资源ID` 在打包之前设置为常量，那么不同依赖库以及主工程的 `R` 文件必然会产生冲突。所有项目中的 `R文件以及其资源ID` 都是所有的代码合并之后重新赋值的或者生成的。

- 源码依赖的 `Lib` 库的 `R` 文件会重新在 `app` 模块的 `build` 目录中重新生成一个相同的`R` 文件只不过 `资源ID` 前面添加了 `final` 关键词变成了常量；
- `aar` 依赖的 `Lib` 库的` R` 文件会更具 `.txt` 文件中的内容，在 `app` 模块的 `build` 目录中重新生成一个`R` 文件而且 `资源ID` 是添加了 `final` 关键词的常量；
- 其 `R` 文件的生成目录和 `主app` 的 `R` 文件是相同的；

![APP_R_JAVA.png](https://img-blog.csdnimg.cn/img_convert/92362b2dfc7bb649abc8e3c9d7b9c051.png#pic_center)


这个目录在`com.android.tools.build:gradle:3.4.1`和`com.android.tools.build:gradle:3.2.0` 版本下都是相同的。

![APP_CLASS_R_ID.png](https://img-blog.csdnimg.cn/img_convert/5e6c3d1eaf4d0558b8877b2e918b2fcd.png#pic_center)


- `AAR的class文件` 在主工程编译时，不会再次进行编译，也就是说**AAR的class文件原封不动的打包进apk**。
- 主工程的代码编译时在`R` 文件生成之后的，所以主工程的资源引用值都是**常量且内联为常量值**。

其实这一点也和之前 `R` 文件结构中的知识点对应起来。`R文件` 是在编译主工程的时候进行合并、排序、赋值的。在这之后又返过来生成 `R.java` 文件，给 `资源ID` 赋予已经生成好的常量值。

### R文件的数量

每个 `aar` 或者 `lib库` 都会有一个 `R文件`，那么一个项目的 `R文件` 数量为：

> **app中R文件数量=依赖的module/aar数量 + 1(自身的R文件)**
>
> **module的R文件数 = 依赖的module/aar数量 + 1(自身的R文件)**



## 后续疑问

我们大概了解的 `R` 文件的生成和使用。但通过本篇文章的了解我们也许会有更多的疑问？

- 为什么要有那么多 `R.java` 文件，而且不同模块的的资源名称还有重复值？
- 资源名称重复的时候会报异常，但这里的部分模块的`资源名称`明显有相同的为什么没有报异常？
- 在编译的时候如果遇到资源重复，那么到底该使用哪个资源，有优先级规则是什么？
- 为什么 `aar` 或者 `lib库` 中使用资源的 `class` 没有进行 `ID值` 的内联？
- `R文件` 可以混淆么，有什么好处或者什么坑？



## 官网参考资料

[添加应用资源](https://developer.android.com/studio/write/add-resources.html)

[应用资源概览](https://developer.android.com/guide/topics/resources/providing-resources?hl=zh-cn)


文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！