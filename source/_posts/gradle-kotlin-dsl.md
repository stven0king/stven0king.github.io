---
title: 将构建配置从 Groovy 迁移到 KTS
date: 2021-6-28 19:29:19
tags: [Groovy,kotlin,gradle]
categories: Android
description: "作为Android开发习惯了面向对象编程，习惯了IDEA提供的各种辅助开发快捷功能。
那么带有陌生的常规语法的Groovy脚本对于我来说一向敬而远之。
Kotlin DSL的出现感觉是为了我们量身定做的，因为采用 Kotlin 编写的代码可读性更高，并且 Kotlin 提供了更好的编译时检查和 IDE 支持。
"
---
# 将构建配置从 Groovy 迁移到 KTS

![icon.jpg](https://img-blog.csdnimg.cn/img_convert/86211657b18d27817f8b26ffd277be43.png#pic_center)
@[TOC](文章目录)

## 前言

作为`Android`开发习惯了面向对象编程，习惯了`IDEA`提供的各种辅助开发快捷功能。

那么带有陌生的常规语法的`Groovy`脚本对于我来说一向敬而远之。

`Kotlin DSL`的出现感觉是为了我们量身定做的，因为采用 Kotlin 编写的代码可读性更高，并且 Kotlin 提供了更好的编译时检查和 IDE 支持。

<hr style=" border:solid; width:100px; height:1px;" color=#000000 size=1">

> 名词概念解释

- **Gradle**: 自动化构建工具. 平行产品: `Maven`.

- **Groovy**: 语言, 编译后变为`JVM byte code`, 兼容`Java`平台.

- **DSL**: `Domain Specific Language`, 领域特定语言.

- **Groovy DSL**: `Gradle`的API是Java的,` Groovy DSL`是在其之上的脚本语言. `Groovy DS`脚本文件后缀: `.gradle`.

- **KTS**：是指 Kotlin 脚本，这是 Gradle 在构建配置文件中使用的一种 [Kotlin 语言形式](https://kotlinlang.org/docs/tutorials/command-line.html#run-scripts)。Kotlin 脚本是[可从命令行运行](https://kotlinlang.org/docs/tutorials/command-line.html#using-the-command-line-to-run-scripts)的 Kotlin 代码。

- **Kotlin DSL**：主要是指 [Android Gradle 插件 Kotlin DSL](https://developer.android.com/reference/tools/gradle-api?hl=zh-cn)，有时也指[底层 Gradle Kotlin DSL](https://guides.gradle.org/migrating-build-logic-from-groovy-to-kotlin/)。

在讨论从 Groovy 迁移时，术语“KTS”和“Kotlin DSL”可以互换使用。换句话说，“将 Android 项目从 Groovy 转换为 KTS”与“将 Android 项目从 Groovy 转换为 Kotlin DSL”实际上是一个意思。

## Groovy和KTS对比

| 类型         | **Kotlin** | **Groovy** |
| ------------ | ---------- | ---------- |
| 自动代码补全 | 支持       | 不支持     |
| 是否类型安全 | 是         | 不是       |
| 源码导航     | 支持       | 不支持     |
| 重构         | 自动关联   | 手动修改   |

### 优点:

- 可以使用`Kotlin`, 开发者可能对这个语言更熟悉更喜欢.
- `IDE`支持更好, 自动补全提示, 重构,` imports`等.
- 类型安全: `Kotlin`是静态类型.
- 不用一次性迁移完: 两种语言的脚本可以共存, 也可以互相调用.

### 缺点和已知问题：

- 目前，采用 `KTS` 的构建速度可能比采用 `Groovy` 慢（自测小demo耗时增加约40%(约8s)）。

- `Project Structure` 编辑器不会展开在 `buildSrc` 文件夹中定义的用于库名称或版本的常量。
- `KTS` 文件目前在项目视图中[不提供文本提示](https://issuetracker.google.com/119757694?hl=zh-cn)。

## Android构建配置从Groovy迁移KTS

### 准备工作

1. `Groovy` 字符串可以用单引号 `'string'` 或双引号 `"string"` 引用，而 `Kotlin` 需要双引号 `"string"`。

2. `Groovy` 允许在调用函数时省略括号，而 `Kotlin` 总是需要括号。

3. `Gradle Groovy DSL` 允许在分配属性时省略 `=` 赋值运算符，而 `Kotlin` 始终需要赋值运算符。

所以在`KTS`中需要统一做到：

- 使用双引号统一引号.

![groovy-kts-diff1.png](https://img-blog.csdnimg.cn/img_convert/4bced2be902c8a5f512472cdfc41d0e2.png#pic_center)


- 消除函数调用和属性赋值的歧义（分别使用括号和赋值运算符）。

![groovy-kts-diff2.png](https://img-blog.csdnimg.cn/img_convert/3a47c02e5057af5f8630107889029901.png#pic_center)


### 脚本文件名

**Groovy DSL** 脚本文件使用 `.gradle` 文件扩展名。

**Kotlin DSL** 脚本文件使用 `.gradle.kt`s 文件扩展名。

### 一次迁移一个文件

由于您可以在项目中结合使用 `Groovy build` 文件和 `KTS build` 文件，因此将项目转换为 `KTS` 的一个简单方法是先选择一个简单的 `build` 文件（例如 `settings.gradle`），将其重命名为 `settings.gradle.kts`，然后将其内容转换为 `KTS`。之后，确保您的项目在迁移每个 `build` 文件之后仍然可以编译。

### 自定义Task

由于`Koltin` 是静态类型语言，`Groovy`是动态语言，前者是类型安全的，他们的性质区别很明显的体现在了 task 的创建和配置上。详情可以参考[Gradle官方迁移教程](https://guides.gradle.org/migrating-build-logic-from-groovy-to-kotlin/#configuring-tasks)

```kotlin
// groovy
task clean(type: Delete) {
    delete rootProject.buildDir
}
// kotiln-dsl
tasks.register("clean", Delete::class) {
    delete(rootProject.buildDir)
}
val clean by tasks.creating(Delete::class) {
    delete(rootProject.buildDir)
}
```

```kotlin
open class GreetingTask : DefaultTask() {
    var msg: String? = null
    @TaskAction
    fun greet() {
        println("GreetingTask:$msg")
    }
}
val msg by tasks.creating(GreetingTask::class) {}
val testTask: Task by tasks.creating {
   doLast {
       println("testTask:Run")
   }
}
val testTask2: Task = task("test2") {
    doLast { 
      println("Hello, World!") 
    }
}
val testTask3: Task = tasks.create("test3") {
    doLast {
        println("testTask:Run")
    }
}
```

### 使用 `plugins` 代码块

如果您在` build` 文件中使用 `plugins` 代码块，`IDE` 将能够获知相关上下文信息，即使在构建失败时也是如此。`IDE` 可使用这些信息执行代码补全并提供其他实用建议，从而帮助您解决 `KTS` 文件中存在的问题。

在您的代码中，将命令式 `apply plugin` 替换为声明式 `plugins` 代码块。`Groovy` 中的以下代码…

```groovy
apply plugin: 'com.android.application'
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-kapt'
apply plugin: 'androidx.navigation.safeargs.kotlin'
```

在 KTS 中变为以下代码：

```kotlin
plugins {
    id("com.android.application")
    id("kotlin-android")
    id("kotlin-kapt")
    id("androidx.navigation.safeargs.kotlin")
 }
```

如需详细了解 `plugins` 代码块，请参阅 [Gradle 的迁移指南](https://docs.gradle.org/nightly/userguide/migrating_from_groovy_to_kotlin_dsl.html#applying_plugins)。

**注意**：`plugins` 代码块仅解析 Gradle 插件门户中提供的插件或使用 [`pluginManagement`](https://docs.gradle.org/current/userguide/plugins.html#sec:plugin_management) 代码块指定的自定义存储库中提供的插件。如果插件来自插件门户中不存在的 `buildScript` 依赖项，那么这些插件在 Kotlin 中就必须使用 `apply` 才能应用。例如：

```kotlin
apply(plugin = "kotlin-android")
apply {
    from("${rootDir.path}/config.gradle")
    from("${rootDir.path}/version.gradle.kts")
}
```

如需了解详情，请参阅 [Gradle 文档](https://docs.gradle.org/current/userguide/plugins.html#sec:applying_plugins_buildscript)。

> 强烈建议您`plugins {}`优先使用块而不是`apply()`函数。
>
> 有两个关键的最佳实践可以更轻松地在 `Kotlin DSL` 的静态上下文中工作：
>
>- 使用`plugins {}`块
>- 将本地构建逻辑放在构建的**buildSrc**目录中
>
> 该[plugins {}块](https://docs.gradle.org/current/userguide/plugins.html#sec:plugins_block)是关于保持您的构建脚本声明性，以便充分利用` Kotlin DSL`。
>
> 使用[*buildSrc*项目](https://docs.gradle.org/current/userguide/organizing_gradle_projects.html#sec:build_sources)是关于将您的构建逻辑组织成共享的本地插件和约定，这些插件和约定易于测试并提供良好的 IDE 支持。

### 依赖管理

> 常见依赖

```kotlin
// groovy
implementation project(':library')
implementation 'com.xxxx:xxxx:8.8.1'

// kotlin
implementation(project(":library"))
implementation("com.xxxx:xxx:8.8.1")
```

> freeTree

```kotlin
// groovy
implementation fileTree(include: '*.jar', dir: 'libs')

//kotlin
implementation(fileTree(mapOf("include" to listOf("*.jar"), "dir" to "libs")))
```

> 特别类型库依赖

```kotlin
//groovy
implementation(name: 'splibrary', ext: 'aar')

//kotlin
implementation (group="",name="splibrary",ext = "aar")
```

### 构建变体

#### 显式和隐式 `buildTypes`

在 Kotlin DSL 中，某些 `buildTypes`（如 `debug` 和 `release,`）是隐式提供的。但是，其他 `buildTypes` 则必须手动创建。

例如，在 Groovy 中，您可能有 `debug`、`release` 和 `staging` `buildTypes`：

```groovy
buildTypes
  debug {
    ...
  }
  release {
    ...
  }
  staging {
    ...
  }
```

在 KTS 中，仅 `debug` 和 `release` `buildTypes` 是隐式提供的，而 `staging` 则必须由您手动创建：

```kotlin
buildTypes
  getByName("debug") {
    ...
  }
  getByName("release") {
    ...
  }
  create("staging") {
    ...
  }
```

#### 举例说明

`Grovvy`编写：

```groovy
productFlavors {
        demo {
            dimension "app"
        }
        full {
            dimension "app"
            multiDexEnabled true
        }
    }

buildTypes {
        release {
            signingConfig signingConfigs.signConfig
            minifyEnabled true
            debuggable false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }

        debug {
            minifyEnabled false
            debuggable true
        }
    }
signingConfigs {
        release {
            storeFile file("myreleasekey.keystore")
            storePassword "password"
            keyAlias "MyReleaseKey"
            keyPassword "password"
        }
        debug {
            ...
        }
    }
```

`kotlin-KTL`编写：

```kotlin
productFlavors {
    create("demo") {
        dimension = "app"
    }
    create("full") {
        dimension = "app"
        multiDexEnabled = true
    }
}

buildTypes {
        getByName("release") {
            signingConfig = signingConfigs.getByName("release")
            isMinifyEnabled = true
            isDebuggable = false
            proguardFiles(getDefaultProguardFile("proguard-android.txtt"), "proguard-rules.pro")
        }
        
        getByName("debug") {
            isMinifyEnabled = false
            isDebuggable = true
        }
    }
    
signingConfigs {
        create("release") {
            storeFile = file("myreleasekey.keystore")
            storePassword = "password"
            keyAlias = "MyReleaseKey"
            keyPassword = "password"
        }
        getByName("debug") {
            ...
        }
    }
```

### 访问配置

#### gradle.properties

我们通常会把签名信息、版本信息等配置写在`gradle.properties`中，在kotlin-dsl中我们可以通过一下方式访问：

1. `rootProject.extra.properties`
2. `project.extra.properties`
3. `rootProject.properties`
4. `properties`
5. `System.getProperties()`

> `System.getProperties()`使用的限制比较多

- 参数名必须按照`systemProp.xxx`格式(例如：`systemProp.kotlinVersion=1.3.72`);
- 与当前执行的task有关（`> Configure project :buildSrc`和`> Configure project :`的结果不同，后者无法获取的`gradle.properties`中的数据）;

#### local.properties

获取工程的`local.properties`文件
> `gradleLocalProperties(rootDir)`

> `gradleLocalProperties(projectDir)`

#### 获取系统环境变量的值

`val JAVA_HOME:String = System.getenv("JAVA_HOME") ?: "default_value"`

### 关于Ext

Google 官方推荐的一个 Gradle 配置[最佳实践](https://developer.android.com/studio/build/gradle-tips?hl=zh-cn#configure-project-wide-properties)是在项目最外层 build.gradle 文件的`ext`代码块中定义项目范围的属性，然后在所有模块间共享这些属性，比如我们通常会这样存放依赖的版本号。

```groovy
// build.gradle

ext {
    compileSdkVersion = 28
    buildToolsVersion = "28.0.3"
    supportLibVersion = "28.0.0"
    ...
}
```

但是由于缺乏IDE的辅助(跳转查看、全局重构等都不支持)，实际使用体验欠佳。

在`KTL`中用`extra`来代替`Groovy`中的`ext`

```kotlin
// The extra object can be used for custom properties and makes them available to all
// modules in the project.
// The following are only a few examples of the types of properties you can define.
extra["compileSdkVersion"] = 28
// You can also create properties to specify versions for dependencies.
// Having consistent versions between modules can avoid conflicts with behavior.
extra["supportLibVersion"] = "28.0.0"
```

```kotlin
android {
    // Use the following syntax to access properties you defined at the project level:
    // rootProject.extra["property_name"]
    compileSdkVersion(rootProject.extra["sdkVersion"])

    // Alternatively, you can access properties using a type safe delegate:
    val sdkVersion: Int by rootProject.extra
    ...
    compileSdkVersion(sdkVersion)
}
...
dependencies {
    implementation("com.android.support:appcompat-v7:${rootProject.ext.supportLibVersion}")
    ...
}
```

>  `build.gralde`中的`ext`数据是可以在`build.gradle.kts`中使用`extra`进行访问的。

### 修改生成apk名称和BuildConfig中添加apk支持的cpu架构

```kotlin
val abiCodes = mapOf("armeabi-v7a" to 1, "x86" to 2, "x86_64" to 3)
android.applicationVariants.all {
    val buildType = this.buildType.name
    val variant = this
    outputs.all {
        val name =
            this.filters.find { it.filterType == com.android.build.api.variant.FilterConfiguration.FilterType.ABI.name }?.identifier
        val baseAbiCode = abiCodes[name]
        if (baseAbiCode != null) {
          	//写入cpu架构信息
            variant.buildConfigField("String", "CUP_ABI", "\"${name}\"")
        }
        if (this is com.android.build.gradle.internal.api.ApkVariantOutputImpl) {
            //修改apk名称
            if (buildType == "release") {
                this.outputFileName = "KotlinDSL_${name}_${buildType}.apk"
            } else if (buildType == "debug") {
                this.outputFileName = "KotlinDSL_V${variant.versionName}_${name}_${buildType}.apk"
            }
        }
    }
}
```

## buildSrc

我们在使用`Groovy`语言构建的时候，往往会抽取一个`version_config.gradle`来作为全局的变量控制，而`ext`扩展函数则是必须要使用到的，而在我们的`Gradle Kotlin DSL`中，如果想要使用全局控制，则需要建议使用`buildSrc`。

复杂的构建逻辑通常很适合作为自定义任务或二进制插件进行封装。自定义任务和插件实现不应存在于构建脚本中。`buildSrc`则不需要在多个独立项目之间共享代码，就可以非常方便地使用该代码了。

`buildSrc`被视为构建目录。编译器发现目录后，`Gradle`会自动编译并测试此代码，并将其放入构建脚本的类路径中。

> 1. 先创建`buildSrc`目录；
> 2. 在该目录下创建`build.gradle.kts`文件；
> 3. 创建一个`buildSrc/src/main/koltin`目录；
> 4. 在该目录下创建`Dependencies.kt`文件作为版本管理类；

需要注意的是`buildSrc`的`build.gradle.kts`：

```kotlin
plugins {
    `kotlin-dsl`
}
repositories {
    jcenter()
}
```

或者

```kotlin
apply {
    plugin("kotlin")
}
buildscript {
    repositories {
        gradlePluginPortal()
    }
    dependencies {
        classpath(kotlin("gradle-plugin", "1.3.72"))
    }
}
//dependencies {
//    implementation(gradleKotlinDsl())
//    implementation(kotlin("stdlib", "1.3.72"))
//}
repositories {
    gradlePluginPortal()
}
```

不同版本之间`buildSrc`下的`build.gradle`文件执行顺序：

> `gradle-wrapper.properties:5.6.4`
>
>  `com.android.tools.build:gradle:3.2.0`

1. `BuildSrc:build.gradle`
2. `setting.gradle`
3. `Project:build.gradle`
4. `Moudle:build.gradle`

> `gradle-wrapper.properties:6.5` 
>
> `com.android.tools.build:gradle:4.1.1`

1. `setting.gradle`
2. `BuildSrc:build.gradle`
3. `Project:build.gradle`
4. `Moudle:build.gradle`

所以在非`buildSrc`目录下的`build.gradle.kts`文件中我们使用`Dependencies.kt`需要注意其加载顺序。

## 参考文档

[Android官网-将构建配置从 Groovy 迁移到 KTS](https://developer.android.com/studio/build/migrate-to-kts?hl=zh-cn)

[Migrating build logic from Groovy to Kotlin](https://guides.gradle.org/migrating-build-logic-from-groovy-to-kotlin/)

[GitHub:kotlin-dsl-samples](https://github.com/gradle/kotlin-dsl-samples)

[GitHub:kotlin-dsl-samples/samples/hello-android](https://github.com/gradle/kotlin-dsl-samples/tree/master/samples/hello-android)

[Kotlin DSL: Gradle scripts in Android made easy](https://medium.com/android-dev-hacks/kotlin-dsl-gradle-scripts-in-android-made-easy-b8e2991e2ba)

[buildSrc官方文档](https://docs.gradle.org/current/userguide/organizing_gradle_projects.html#sec:build_sources)

[Gradle’s Kotlin DSL BuildSrc](https://medium.com/swlh/gradles-kotlin-dsl-buildsrc-4434100a07d7)


文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦\~！~！