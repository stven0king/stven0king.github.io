
---
title: JNI方法注册源码分析(JNI_OnLoad|动态注册|静态注册|方法替换)
date: 2022-04-29 10:17:58
tags: [安全,JNI]
categories: Android
description:  "文章指在学会使用JNI方法动态注册，静态注册，方法替换，且在这个过程中稍微了解一下`native`层的动态库加载，方法加载等知识。"
---

![请添加图片描述](https://img-blog.csdnimg.cn/0cdffc517b1c4ae285389c4ff3680283.webp#pic_center)

## 背景

开发`Android`应用时，有时候Java层的编码不能满足实际需求，需要通过JNI的方式利用C/C++实现重要功能并生成SO文件，再通过`System.loadLibrary()`加载进行调用。常见的场景如：加解密算法、音视频编解码、数据采集、设备指纹等。通常核心代码都封装在SO文件中，也自然成为“黑客”攻击的目标对象，利用IDA Pro等逆向工具，可以轻松反编译未采取任何保护措施的SO文件，生成近似源代码的C代码，业务逻辑、核心技术将直接暴露在攻击者的眼前。进一步造成核心技术泄漏、隐私数据泄漏、业务逻辑恶意篡改等危害。

高级选手可以编译链加固，采用花指令等方案。入门选手可以采用Native方法动态注册，混淆方名。

文章指在学会使用JNI方法动态注册，静态注册，方法替换，且在这个过程中稍微了解一下`native`层的动态库加载，方法加载等知识。

## 举例说明

通过`javah`,获取一组带签名函数，然后实现这些函数。对应的`native`层的函数是：`Java_类名_方法名`，样式如下：

```c++
JNIEXPORT jstring JNICALL Java_com_jni_tzx_utils_JNIUitls_getNameString(JNIEnv *env,jclass type){
    return env->NewStringUTF("JNIApplication");
}
JNIEXPORT jint JNICALL
Java_com_jni_tzx_utils_JNIUitls_getNumber(JNIEnv *env,jobject instance,jint num){
    return 0;
}
```

但这样存在两个问题：

- 第一个问题是在IDA工具查看so文件，或者执行`nm -D libxxx.so`

```c++
00000000000006d8 T Java_com_jni_tzx_utils_JNIUitls_getNameString
000000000000073c T Java_com_jni_tzx_utils_JNIUitls_getNumber
0000000000000708 W _ZN7_JNIEnv12NewStringUTFEPKc
                 U __cxa_atexit@LIBC
                 U __cxa_finalize@LIBC
```

我们会得到类似上面的输出结果。可以明显的看到该so是对应的那个Java类的那个方法。

- 第二个问题是恶意攻击者可以得到这个so文件之后，查看这个native方法的参数和返回类型，也就是方法的签名，然后自己在`Java`层写个Demo程序，然后构造一个和so文件中对应的native方法，就可以执行这个`native`方法，如果有一个校验密码或者是获取密码的方法是个`native`的，那么这个时候就会很容易被攻击者执行方法后获取结果。

上面的两个问题可以看到，如果`native`层的函数遵循了这样的格式，无疑给破解提供的简单的一种方式。

## 手动注册native方法

先看结果：

原有的`native`方法：

```c++
JNIEXPORT jstring JNICALL Java_com_jni_tzx_MainActivity_stringFromJNI(JNIEnv *env, jobject /* this */) {
    return env->NewStringUTF("Hello from C++ dynamic\n");
}
```

经过动态注册之后：

```c++
0000000000000a2c T JNI_OnLoad
00000000000009f8 W _ZN7_JNIEnv12NewStringUTFEPKc
0000000000000bb8 W _ZN7_JNIEnv15RegisterNativesEP7_jclassPK15JNINativeMethodi
0000000000000b84 W _ZN7_JNIEnv9FindClassEPKc
0000000000000b48 W _ZN7_JavaVM6GetEnvEPPvi
                 U __android_log_print
                 U __cxa_atexit@LIBC
                 U __cxa_finalize@LIBC
                 U __stack_chk_fail@LIBC
00000000000009c8 T test
```

`test` 是动态注册的方法名，对应`com.jni.tzx.MainActivity#stringFromJNI`方法。

**手动注册native方法**这个手段其实不太常用，因为它的安全措施不是很强大，但是也可以起到一定的作用。聊这个知识点之前，先了解一下so加载的流程。

## so方法加载

![请添加图片描述](https://img-blog.csdnimg.cn/b219f30b022c4126b9e838035cf2524c.png#pic_center)

> 在Android中，当程序在Java成运行`System.loadLibrary("jnitest");`这行代码后，程序会去载入`libjnitset.so`文件。于此同时，产生一个Load事件，这个事件触发后，程序默认会在载入的`.so`文件的函数列表中查找`JNI_OnLoad`函数并执行，与Load事件相对，在载入的.so文件被卸载时，Unload事件被触发。此时，程序默认会去载入的.so文件的函数列表中查找`JNI_OnLoad`函数并执行，然后卸载.so文件。
>
> 需要注意的是`JNI_OnLoad`和`JNI_OnUnLoad`这两个函数在.so组件中并不是强制要求的，可以不用去实现。
>
> 我们可以将`JNI_OnLoad`函数看做**构造函数**在初始化时候调用，可以将`JNI_OnUnLoad`函数看做**析构函数**在被卸载的时候调用；
>
> 应用层的`Java`程序需要调用本地方法时，虚拟机在加载的动态文件中定位并链接该本地方法，从而得以执行本地方法。这中定位`native`方法是按照命名规范来的。如果找不到就会崩溃。

### jni方法查找失败

//这个是找到方法

```java
Process: com.jni.tzx, PID: 1598
java.lang.UnsatisfiedLinkError: No implementation found for java.lang.String com.jni.tzx.utils.JNIUitls.test() (tried Java_com_jni_tzx_utils_JNIUitls_test and Java_com_jni_tzx_utils_JNIUitls_test__)
	at com.jni.tzx.utils.JNIUitls.test(Native Method)
	at com.jni.tzx.MainActivity.onCreate(MainActivity.java:24)
```

### 加载so文件

//java.lang.System.java

```java
public static void loadLibrary(String libname) {
    Runtime.getRuntime().loadLibrary0(VMStack.getCallingClassLoader(), libname);
}
```

//java.lang.Runtime.java
```java
synchronized void loadLibrary0(ClassLoader loader, String libname) {
    if (libname.indexOf((int)File.separatorChar) != -1) {
        throw new UnsatisfiedLinkError(
"Directory separator should not appear in library name: " + libname);
    }
    String libraryName = libname;
    if (loader != null) {
        String filename = loader.findLibrary(libraryName);
        if (filename == null) {
            // It's not necessarily true that the ClassLoader used
            // System.mapLibraryName, but the default setup does, and it's
            // misleading to say we didn't find "libMyLibrary.so" when we
            // actually searched for "liblibMyLibrary.so.so".
            throw new UnsatisfiedLinkError(loader + " couldn't find \"" +
                                            System.mapLibraryName(libraryName) + "\"");
        }
        String error = nativeLoad(filename, loader);
        if (error != null) {
            throw new UnsatisfiedLinkError(error);
        }
        return;
    }
}
private static native String nativeLoad(String filename, ClassLoader loader);
```

### so文件所在目录

![请添加图片描述](https://img-blog.csdnimg.cn/b76d7d8f046a49dca396edab9ecd3ce0.png#pic_center)

```java
public class BaseDexClassLoader extends ClassLoader {
  	private final DexPathList pathList;
  	@Override
    public String findLibrary(String name) {
        return pathList.findLibrary(name);
    }
}
final class DexPathList {
  	public DexPathList(ClassLoader definingContext, String dexPath,
       String libraryPath, File optimizedDirectory) {
		    /********部分代码省略*******/
    		this.nativeLibraryDirectories = splitLibraryPath(libraryPath);
		}
  	private static File[] splitLibraryPath(String path) {
			  //System.getProperty("java.library.path")=/system/lib:/system/product/lib
		  	ArrayList<File> result = splitPaths(path, System.getProperty("java.library.path"), true);
    		return result.toArray(new File[result.size()]);
		}
  	public String findLibrary(String libraryName) {
        //前面添加lib，后面添加.so。具体实现可以参考下面的so名称获取
        String fileName = System.mapLibraryName(libraryName);
       for (File directory : nativeLibraryDirectories) {
            String path = new File(directory, fileName).getPath();
               if (IoUtils.canOpenReadOnly(path)) {
                   return path;
               }
           }
        return null;
    }
}
//java_lang_System.cpp
static jstring System_mapLibraryName(JNIEnv* env, jclass, jstring javaName) {
    ScopedUtfChars name(env, javaName);
    if (name.c_str() == NULL) {
        return NULL;
    }
    char* mappedName = NULL;
  	//#define OS_SHARED_LIB_FORMAT_STR    "lib%s.so"
    asprintf(&mappedName, OS_SHARED_LIB_FORMAT_STR, name.c_str());
    jstring result = env->NewStringUTF(mappedName);
    free(mappedName);
    return result;
}
```

通过反射获取出`BaseDexClassLoader`的`getLdLibraryPath`是`/data/app/com.jni.tzx--DSAI2wgWJ3_R3i0iUUeJA==/lib/arm64:/data/app/com.jni.tzx--DSAI2wgWJ3_R3i0iUUeJA==/base.apk!/lib/arm64-v8a`

`LoadedApk` 的`getClassLoader`方法调用`ApplicationLoaders.getClassLoader`方法创建`PathClassLoader`。

构造`PathClassLoader`的`libraryDir`是来自`ApplicationInfo.nativeLibraryDir`。

![请添加图片描述](https://img-blog.csdnimg.cn/16db414936974630b76c563ef24ab1a6.png#pic_center)

`nativeLibararyDir`的赋值主要是在`PackageManagerService`中，通过`apkRoot也就是sourceDir`。

```java
VMRuntime.is64BitInstructionSet(getPrimaryInstructionSet(info));
```

再判断是否为`64`位，确定是安装包的`lib`目录。最后根据`AIB`确定`nativeLibararyDir`。

`ABI`相关初始化流程可以参考：[Android中app进程ABI确定过程](https://zhuanlan.zhihu.com/p/29801087)

### nativeLoad

https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/native/java_lang_Runtime.cpp

```cpp
/*
 * static String nativeLoad(String filename, ClassLoader loader)
 * 将指定的完整路径加载为动态库 JNI 兼容的方法。 成功或失败返回 null 失败消息。
 */
static void Dalvik_java_lang_Runtime_nativeLoad(const u4* args,
    JValue* pResult)
{
    StringObject* fileNameObj = (StringObject*) args[0];
    Object* classLoader = (Object*) args[1];
    char* fileName = NULL;
    StringObject* result = NULL;
    char* reason = NULL;
    bool success;
    assert(fileNameObj != NULL);
    fileName = dvmCreateCstrFromString(fileNameObj);
    success = dvmLoadNativeCode(fileName, classLoader, &reason);
    if (!success) {
        const char* msg = (reason != NULL) ? reason : "unknown failure";
        result = dvmCreateStringFromCstr(msg);
        dvmReleaseTrackedAlloc((Object*) result, NULL);
    }
    free(reason);
    free(fileName);
    RETURN_PTR(result);
}
```

https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/Native.cpp

```cpp
bool dvmLoadNativeCode(const char* pathName, Object* classLoader,
        char** detail)
{
    SharedLib* pEntry;
    void* handle;
    bool verbose;
    /********部分代码省略***********/
    Thread* self = dvmThreadSelf();
    ThreadStatus oldStatus = dvmChangeStatus(self, THREAD_VMWAIT);
    //打开pathName对应的动态链接库，配合dlsym函数使用
    handle = dlopen(pathName, RTLD_LAZY);
    dvmChangeStatus(self, oldStatus);
    if (handle == NULL) {
        *detail = strdup(dlerror());
        ALOGE("dlopen(\"%s\") failed: %s", pathName, *detail);
        return false;
    }
  
    /* create a new entry */
    SharedLib* pNewEntry;
    pNewEntry = (SharedLib*) calloc(1, sizeof(SharedLib));
    pNewEntry->pathName = strdup(pathName);
    pNewEntry->handle = handle;
    pNewEntry->classLoader = classLoader;
    dvmInitMutex(&pNewEntry->onLoadLock);
    pthread_cond_init(&pNewEntry->onLoadCond, NULL);
    pNewEntry->onLoadThreadId = self->threadId;

    /* try to add it to the list */
    SharedLib* pActualEntry = addSharedLibEntry(pNewEntry);
    if (pNewEntry != pActualEntry) {
        ALOGI("WOW: we lost a race to add a shared lib (%s CL=%p)",
            pathName, classLoader);
        freeSharedLibEntry(pNewEntry);
        return checkOnLoadResult(pActualEntry);
    } else {
        if (verbose)
            ALOGD("Added shared lib %s %p", pathName, classLoader);
        bool result = true;
        void* vonLoad;
        int version;
        //获取JNI_OnLoad的地址
        vonLoad = dlsym(handle, "JNI_OnLoad");
        //这是用javah风格的代码了，推迟native方法的解析，相当于构造为空不需要进行初始化
        if (vonLoad == NULL) {
            ALOGD("No JNI_OnLoad found in %s %p, skipping init",
                pathName, classLoader);
        } else {
            /*
             * Call JNI_OnLoad.  We have to override the current class
             * loader, which will always be "null" since the stuff at the
             * top of the stack is around Runtime.loadLibrary().  (See
             * the comments in the JNI FindClass function.)
             */
            OnLoadFunc func = (OnLoadFunc)vonLoad;
            Object* prevOverride = self->classLoaderOverride;
            self->classLoaderOverride = classLoader;
            oldStatus = dvmChangeStatus(self, THREAD_NATIVE);
            if (gDvm.verboseJni) {
                ALOGI("[Calling JNI_OnLoad for \"%s\"]", pathName);
            }
            //调用JNI_OnLoad，并获取返回的版本信息
            version = (*func)(gDvmJni.jniVm, NULL);
            dvmChangeStatus(self, oldStatus);
            self->classLoaderOverride = prevOverride;
            //对版本进行判断，这是为什么要返回正确版本的原因。现在一般都是JNI_VERSION_1_6
            if (version != JNI_VERSION_1_2 && version != JNI_VERSION_1_4 &&
                version != JNI_VERSION_1_6)
            {
                ALOGW("JNI_OnLoad returned bad version (%d) in %s %p",
                    version, pathName, classLoader);
                /*
                 * It's unwise to call dlclose() here, but we can mark it
                 * as bad and ensure that future load attempts will fail.
                 *
                 * We don't know how far JNI_OnLoad got, so there could
                 * be some partially-initialized stuff accessible through
                 * newly-registered native method calls.  We could try to
                 * unregister them, but that doesn't seem worthwhile.
                 */
                result = false;
            } else {
                if (gDvm.verboseJni) {
                    ALOGI("[Returned from JNI_OnLoad for \"%s\"]", pathName);
                }
            }
        }
        if (result)
            pNewEntry->onLoadResult = kOnLoadOkay;
        else
            pNewEntry->onLoadResult = kOnLoadFailed;
        pNewEntry->onLoadThreadId = 0;
        /*
         * Broadcast a wakeup to anybody sleeping on the condition variable.
         */
        dvmLockMutex(&pNewEntry->onLoadLock);
        pthread_cond_broadcast(&pNewEntry->onLoadCond);
        dvmUnlockMutex(&pNewEntry->onLoadLock);
        return result;
    }
}
```
- `JNI_OnLoad` 需要返回值，但是只能选择返回后三个 `JNI_VERSION_1_2` ,` JNI_VERSION_1_4 `, `JNI_VERSION_1_6` , 返回上述三个值任意一个没有区别 ;

返回 JNI_VERSION_1_1 会报错 ：

```c
#define JNI_VERSION_1_1 0x00010001
#define JNI_VERSION_1_2 0x00010002
#define JNI_VERSION_1_4 0x00010004
#define JNI_VERSION_1_6 0x00010006
```

讲`so`库加载之后，补充两个函数分别是：`dlopen`和`dlsym`函数。

### dlopen函数

> 功能：打开一个动态链接库

- 包含头文件：`#include <dlfcn.h>`
- 函数定义：`void * dlopen( const char * pathname, int mode );`
- 函数描述：在`dlopen()`函数以指定模式打开指定的动态连接库文件，并返回一个句柄给调用进程。通过这个句柄来使用库中的函数和类。使用`dlclose()`来卸载打开的库。
- `mode`：分为这两种
  - `RTLD_LAZY` 暂缓决定，等有需要时再解出符号;
  - `RTLD_NOW` 立即决定，返回前解除所有未决定的符号;
  - `RTLD_LOCAL`
  - `RTLD_GLOBAL` 允许导出符号;
  - `RTLD_GROUP`
  - `RTLD_WORLD`
- 返回值:

  - 打开错误返回NULL；

  - 成功，返回库引用；

### dlsym函数

 函数原型是`void* dlsym(void* handle,const char* symbol)`, 该函数在`<dlfcn.h>`文件中。

` handle`是由`dlopen`打开动态链接库后返回的指针，`symbol`就是要求获取的函数的名称，函数返回值是`void*`,指向函数的地址，供调用使用。

### so方法延迟解析

上面的代码说明，`JNI_OnLoad`是一种更加灵活，而且处理及时的机制。用`javah`风格的代码，则推迟解析，直到需要调用的时候才会解析。这样的函数，是`dvmResolveNativeMethod(dalvik/vm/Native.cpp)`。

https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/Native.cpp
```cpp
/*
 * Resolve a native method and invoke it.
 * //解析native方法并调用它。
 * This is executed as if it were a native bridge or function.  If the
 * resolution succeeds, method->insns is replaced, and we don't go through
 * here again unless the method is unregistered.
 *
 * Initializes method's class if necessary.
 *
 * An exception is thrown on resolution failure.
 *
 * (This should not be taking "const Method*", because it modifies the
 * structure, but the declaration needs to match the DalvikBridgeFunc
 * type definition.)
 */
void dvmResolveNativeMethod(const u4* args, JValue* pResult,
    const Method* method, Thread* self)
{
    ClassObject* clazz = method->clazz;
    /*
     * If this is a static method, it could be called before the class
     * has been initialized.
     */
    if (dvmIsStaticMethod(method)) {
        if (!dvmIsClassInitialized(clazz) && !dvmInitClass(clazz)) {
            assert(dvmCheckException(dvmThreadSelf()));
            return;
        }
    } else {
        assert(dvmIsClassInitialized(clazz) ||
               dvmIsClassInitializing(clazz));
    }
    /* start with our internal-native methods */
    DalvikNativeFunc infunc = dvmLookupInternalNativeMethod(method);
    if (infunc != NULL) {
        /* resolution always gets the same answer, so no race here */
        IF_LOGVV() {
            char* desc = dexProtoCopyMethodDescriptor(&method->prototype);
            LOGVV("+++ resolved native %s.%s %s, invoking",
                clazz->descriptor, method->name, desc);
            free(desc);
        }
        if (dvmIsSynchronizedMethod(method)) {
            ALOGE("ERROR: internal-native can't be declared 'synchronized'");
            ALOGE("Failing on %s.%s", method->clazz->descriptor, method->name);
            dvmAbort();     // harsh, but this is VM-internal problem
        }
        DalvikBridgeFunc dfunc = (DalvikBridgeFunc) infunc;
        dvmSetNativeFunc((Method*) method, dfunc, NULL);
        dfunc(args, pResult, method, self);
        return;
    }
    /* now scan any DLLs we have loaded for JNI signatures */
  	//根据signature在所有已经打开的.so中寻找此函数实现
    void* func = lookupSharedLibMethod(method);
    if (func != NULL) {
        /* found it, point it at the JNI bridge and then call it */
      	//找到方法，并执行调用
        dvmUseJNIBridge((Method*) method, func);
        (*method->nativeFunc)(args, pResult, method, self);
        return;
    }
    IF_ALOGW() {
        char* desc = dexProtoCopyMethodDescriptor(&method->prototype);
        ALOGW("No implementation found for native %s.%s:%s",
            clazz->descriptor, method->name, desc);
        free(desc);
    }
    //抛出java.lang.UnsatisfiedLinkError异常
    dvmThrowUnsatisfiedLinkError("Native method not found", method);
}
```

- 根据方法的`signature`在所有已经打开的`.so`中寻找此函数实现；

https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/Native.cpp

```c++
/*
 * See if the requested method lives in any of the currently-loaded
 * shared libraries.  We do this by checking each of them for the expected
 * method signature.
 */
static void* lookupSharedLibMethod(const Method* method)
{
    if (gDvm.nativeLibs == NULL) {
        ALOGE("Unexpected init state: nativeLibs not ready");
        dvmAbort();
    }
  	//从已经加载的nativeLibs中轮询执行findMethodInLib方法寻找method
    return (void*) dvmHashForeach(gDvm.nativeLibs, findMethodInLib,
        (void*) method);
}
```

- 对哈希表中的每个条目执行一个函数。如果`func`返回一个非零值，提前终止并返回该值。

https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/Hash.cpp

```c++
/*
 * Execute a function on every entry in the hash table.
 *
 * If "func" returns a nonzero value, terminate early and return the value.
 */
int dvmHashForeach(HashTable* pHashTable, HashForeachFunc func, void* arg)
{
    int i, val;
    for (i = 0; i < pHashTable->tableSize; i++) {
        HashEntry* pEnt = &pHashTable->pEntries[i];
        if (pEnt->data != NULL && pEnt->data != HASH_TOMBSTONE) {
            val = (*func)(pEnt->data, arg);
            if (val != 0)
                return val;
        }
    }
    return 0;
}
```

在讲从依赖库中搜索匹配的方法之前，先看一下`Method`结构体的定义：

https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/oo/Object.h

```cpp
struct Method {
    ClassObject*    clazz;
    u4              accessFlags;
    u2             methodIndex;
    u2              registersSize;
    u2              outsSize;
    u2              insSize;
    const char*     name;
    DexProto        prototype;
    const char*     shorty;
    const u2*       insns;
    int             jniArgInfo;
    DalvikBridgeFunc nativeFunc;
    bool fastJni;
    bool noRef;
    bool shouldTrace;
    const RegisterMap* registerMap;
    bool            inProfile;
};
```

https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/Native.cpp

```cpp
/*
 * (This is a dvmHashForeach callback.)
 * //在依赖库中搜索匹配的方法
 * Search for a matching method in this shared library.
 *
 * TODO: we may want to skip libraries for which JNI_OnLoad failed.
 */
static int findMethodInLib(void* vlib, void* vmethod)
{
    const SharedLib* pLib = (const SharedLib*) vlib;
    const Method* meth = (const Method*) vmethod;
    char* preMangleCM = NULL;
    char* mangleCM = NULL;
    char* mangleSig = NULL;
    char* mangleCMSig = NULL;
    void* func = NULL;
    int len;
    if (meth->clazz->classLoader != pLib->classLoader) {
        ALOGV("+++ not scanning '%s' for '%s' (wrong CL)",
            pLib->pathName, meth->name);
        return 0;
    } else
        ALOGV("+++ scanning '%s' for '%s'", pLib->pathName, meth->name);
    /*
     * First, we try it without the signature.
     */
  	//把java的native方法的名字进行转换，生成和javah一致的名字
    preMangleCM =
        createJniNameString(meth->clazz->descriptor, meth->name, &len);
    if (preMangleCM == NULL)
        goto bail;
    mangleCM = mangleString(preMangleCM, len);
    if (mangleCM == NULL)
        goto bail;
    ALOGV("+++ calling dlsym(%s)", mangleCM);
		//dlsym清晰的表明，这里才是获取函数的地方。
    func = dlsym(pLib->handle, mangleCM);
    if (func == NULL) {
      	//获取native方法的返回类型和参数类型
        mangleSig = createMangledSignature(&meth->prototype);
        if (mangleSig == NULL)
            goto bail;
        mangleCMSig = (char*) malloc(strlen(mangleCM) + strlen(mangleSig) +3);
        if (mangleCMSig == NULL)
            goto bail;
       //将native的方法名和方法的返回类型和参数类型通过__进行拼接
        sprintf(mangleCMSig, "%s__%s", mangleCM, mangleSig);
        ALOGV("+++ calling dlsym(%s)", mangleCMSig);
        func = dlsym(pLib->handle, mangleCMSig);
        if (func != NULL) {
            ALOGV("Found '%s' with dlsym", mangleCMSig);
        }
    } else {
        ALOGV("Found '%s' with dlsym", mangleCM);
    }
bail:
    free(preMangleCM);
    free(mangleCM);
    free(mangleSig);
    free(mangleCMSig);
    return (int) func;
}

```

从上面看到一般通过VM去寻找`*.so`里的`native`函数。如果需连续调用很多次，每次度需要寻找一遍，回花很多时间。

此时，C组件开发者可以将本地函数向VM进行注册，以便能加快后续调用`native`函数的效率。可以这么想象一下，假设VM内部一个`native`函数链表，初始时是空的，在未显示注册之前，此`native`数链表是空的，每次`java`调用`native`函数之前会首先在此链表中查找需要调用的`native`函数，如果找到就直接调用，如果未找到，得再通过载入的`.so`文件中的函数列表中去查找，且每次`Java`调用`native`函数都是进行这样的流程。因此，效率就自然会下降。

为了客服这个问题，我们可以通过在`.so`文件载入初始化时，即`JNI_OnLoad`函数中，先行将`native`函数注册VM的`native`函数链表中去，这样一来，后续每次`Java`调用`native`函数都会在VM中`native`函数链表中找到对应的函数，从而加快速度。

#### 优点

简单明了

### so方法动态注册

这种方式，写的代码稍微多点，但好处很明显，函数映射关系配置灵活，执行效率要比第一种方式高。

先看一个简单的动态注册实现Demo：

```cpp
package com.jni.tzx;
public class MainActivity extends Activity {
    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }
    /*****部分代码删除********/
    /**
     * A native method that is implemented by the 'native-lib' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI();
    public native String testNative();
}
```

对应的`natieve-lib.cpp`文件：

注册`navtive`方法之前我们需要了解`JavaVM`,` JNIEnv`:

`JavaVM` 和 `JNIEnv` 是`JNI`提供的结构体.

`JavaVM` 提供了允许你创建和销毁`JavaVM`的`invokation interface`。理论上在每个进程中你可以创建多个`JavaVM`， 但是`Android`只允许创造一个。

`JNIEnv` 提供了大部分`JNI`中的方法。在你的`Native`方法中的第一个参数就是`JNIEnv`.

`JNIEnv` 用于线程内部存储。 因此， 不能多个线程共享一个`JNIEnv`. 在一段代码中如果无法获取`JNIEnv`， 你可以通过共享`JavaVM`并调用`GetEnv()`方法获取。

`JNINativeMethod`是动态注册方法需要的结构体：

```cpp
typedef struct {
		const char* name;//在java中声明的native函数名
	  const char* signature;//函数的签名，可以通过javah获取
  	void* fnPtr;//对应的native函数名
}JNINativeMethod
```

![请添加图片描述](https://img-blog.csdnimg.cn/5309107116df4b5ab21ec322e045fdfb.png#pic_center)

```cpp
#include <jni.h>
#include <string>
//作用：避免编绎器按照C++的方式去编绎C函数
extern "C"
//用来表示该函数是否可导出（即：方法的可见性）
JNIEXPORT jstring
//用来表示函数的调用规范（如：__stdcall）
JNICALL
Java_com_jni_tzx_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {////非static的方法参数类型是jobject instance,而static的方法参数类型是jclass type
    std::string hello = "Hello from C++ dynamic\n";
    return env->NewStringUTF(hello.c_str());
}
extern "C"
JNIEXPORT jstring JNICALL
//Java_com_jni_tzx_MainActivity_test(JNIEnv *env, jobject thiz) {
test(JNIEnv *env, jobject thiz) {
    return env->NewStringUTF("Java_com_jni_tzx_MainActivity_test");
}
extern "C"
JNIEXPORT jint JNICALL
//System.loadLibrary方法会调用载入的.so文件的函数列表中查找JNI_OnLoad函数并执行
JNI_OnLoad(JavaVM* vm, void* reserved) {
    static JNINativeMethod methods[] = {
            {
              "testNative", //在java中声明的native函数名
              "()Ljava/lang/String;", //函数的签名，可以通过javah获取
              (void *)test//对应的native函数名
            }
    };
    JNIEnv *env = NULL;
    jint result = -1;
    // 获取JNI env变量
    if (vm->GetEnv((void**) &env, JNI_VERSION_1_6) != JNI_OK) {
        // 失败返回-1
        return result;
    }
    // 获取native方法所在Java类，包名和类名之间使用“/”风格
    const char* className = "com/jni/tzx/MainActivity";
    //这个可以找到要注册的类，提前是这个类已经加载到Java虚拟机中；
    //这里说明，动态库和有native方法的类之间，没有任何关系。
    jclass clazz = env->FindClass(className);
    if (clazz == NULL) {
        return result;
    }
    // 动态注册native方法
    if (env->RegisterNatives(clazz, methods, sizeof(methods) / sizeof(methods[0])) < 0) {
        return result;
    }

    // 返回成功
    result = JNI_VERSION_1_6;
    return result;
}
```

如果使用`C`语言，那么需要有几个地方进行修改：

```c
(*vm)->GetEnv(vm, (void **)&env, JNI_VERSION_1_4)
(*env)->FindClass(env, kClassName);
(*env)->RegisterNatives(env, clazz, gMethods, sizeof(gMethods) / sizeof(gMethods[0]))
```

#### FindClass

这个可以找到要注册的类，提前是这个类已经加载到`Java虚拟机`中；

https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/Jni.cpp

https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/Misc.cpp

```cpp
/*
 * Find a class by name.
 *
 * We have to use the "no init" version of FindClass here, because we might
 * be getting the class prior to registering native methods that will be
 * used in <clinit>.
 *
 * We need to get the class loader associated with the current native
 * method.  If there is no native method, e.g. we're calling this from native
 * code right after creating the VM, the spec says we need to use the class
 * loader returned by "ClassLoader.getBaseClassLoader".  There is no such
 * method, but it's likely they meant ClassLoader.getSystemClassLoader.
 * We can't get that until after the VM has initialized though.
 */
static jclass FindClass(JNIEnv* env, const char* name) {
    ScopedJniThreadState ts(env);
  	//通过检查堆栈获取当前正在执行的方法。
    const Method* thisMethod = dvmGetCurrentJNIMethod();
    assert(thisMethod != NULL);
    Object* loader;
    Object* trackedLoader = NULL;
    //获取当前的类加载器
    if (ts.self()->classLoaderOverride != NULL) {
        /* hack for JNI_OnLoad */
        assert(strcmp(thisMethod->name, "nativeLoad") == 0);
        loader = ts.self()->classLoaderOverride;
    } else if (thisMethod == gDvm.methDalvikSystemNativeStart_main ||
               thisMethod == gDvm.methDalvikSystemNativeStart_run) {
        /* start point of invocation interface */
        if (!gDvm.initializing) {
            loader = trackedLoader = dvmGetSystemClassLoader();
        } else {
            loader = NULL;
        }
    } else {
        loader = thisMethod->clazz->classLoader;
    }
    //获取类的描述符，它是类的完整名称（包名+类名）,将原来的 . 分隔符换成 / 分隔符。
    char* descriptor = dvmNameToDescriptor(name);
    if (descriptor == NULL) {
        return NULL;
    }
    //获取已经加载的类，或者使用类加载加载该类
    ClassObject* clazz = dvmFindClassNoInit(descriptor, loader);
    free(descriptor);
    jclass jclazz = (jclass) addLocalReference(ts.self(), (Object*) clazz);
    dvmReleaseTrackedAlloc(trackedLoader, ts.self());
    return jclazz;
}
```

https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/oo/Class.cpp

```cpp
/*
 * Find the named class (by descriptor), using the specified
 * initiating ClassLoader.
 *
 * The class will be loaded if it has not already been, as will its
 * superclass.  It will not be initialized.
 *
 * If the class can't be found, returns NULL with an appropriate exception
 * raised.
 */
ClassObject* dvmFindClassNoInit(const char* descriptor,
        Object* loader)
{
    assert(descriptor != NULL);
    //assert(loader != NULL);
    LOGVV("FindClassNoInit '%s' %p", descriptor, loader);
    if (*descriptor == '[') {
        /*
         * Array class.  Find in table, generate if not found.
         */
        return dvmFindArrayClass(descriptor, loader);
    } else {
        /*
         * Regular class.  Find in table, load if not found.
         */
        if (loader != NULL) {
            return findClassFromLoaderNoInit(descriptor, loader);
        } else {
            return dvmFindSystemClassNoInit(descriptor);
        }
    }
}
/*
 * Load the named class (by descriptor) from the specified class
 * loader.  This calls out to let the ClassLoader object do its thing.
 *
 * Returns with NULL and an exception raised on error.
 */
static ClassObject* findClassFromLoaderNoInit(const char* descriptor,
    Object* loader)
{
    //ALOGI("##### findClassFromLoaderNoInit (%s,%p)",
    //        descriptor, loader);
    Thread* self = dvmThreadSelf();
    assert(loader != NULL);
    //是否已经加载过该类
    ClassObject* clazz = dvmLookupClass(descriptor, loader, false);
    if (clazz != NULL) {
        LOGVV("Already loaded: %s %p", descriptor, loader);
        return clazz;
    } else {
        LOGVV("Not already loaded: %s %p", descriptor, loader);
    }
    char* dotName = NULL;
    StringObject* nameObj = NULL;
    /* convert "Landroid/debug/Stuff;" to "android.debug.Stuff" */
    dotName = dvmDescriptorToDot(descriptor);
    if (dotName == NULL) {
        dvmThrowOutOfMemoryError(NULL);
        return NULL;
    }
    nameObj = dvmCreateStringFromCstr(dotName);
    if (nameObj == NULL) {
        assert(dvmCheckException(self));
        goto bail;
    }
    dvmMethodTraceClassPrepBegin();
   	//调用 loadClass()。 这可能会导致几个抛出异
  	//因为 ClassLoader.loadClass()实现最终调用 VMClassLoader.loadClass 看是否引导类加载器可以在自己加载之前找到它。
    LOGVV("--- Invoking loadClass(%s, %p)", dotName, loader);
    {
        const Method* loadClass =
            loader->clazz->vtable[gDvm.voffJavaLangClassLoader_loadClass];
        JValue result;
        dvmCallMethod(self, loadClass, loader, &result, nameObj);
        clazz = (ClassObject*) result.l;
        dvmMethodTraceClassPrepEnd();
        Object* excep = dvmGetException(self);
        if (excep != NULL) {
#if DVM_SHOW_EXCEPTION >= 2
            ALOGD("NOTE: loadClass '%s' %p threw exception %s",
                 dotName, loader, excep->clazz->descriptor);
#endif
            dvmAddTrackedAlloc(excep, self);
            dvmClearException(self);
            dvmThrowChainedNoClassDefFoundError(descriptor, excep);
            dvmReleaseTrackedAlloc(excep, self);
            clazz = NULL;
            goto bail;
        } else if (clazz == NULL) {
            ALOGW("ClassLoader returned NULL w/o exception pending");
            dvmThrowNullPointerException("ClassLoader returned null");
            goto bail;
        }
    }
    /* not adding clazz to tracked-alloc list, because it's a ClassObject */
    dvmAddInitiatingLoader(clazz, loader);
    LOGVV("--- Successfully loaded %s %p (thisldr=%p clazz=%p)",
        descriptor, clazz->classLoader, loader, clazz);
bail:
    dvmReleaseTrackedAlloc((Object*)nameObj, NULL);
    free(dotName);
    return clazz;
}
```

#### RegisterNatives

注册一个类的一个或者多个`native`方法。

- `clazz`：指定的类，即 native 方法所属的类

- `methods`：方法数组，这里需要了解一下 JNINativeMethod 结构体

- `nMethods`：方法数组的长度

https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/Jni.cpp

```cpp
/*
 * Register one or more native functions in one class.
 *
 * This can be called multiple times on the same method, allowing the
 * caller to redefine the method implementation at will.
 */
static jint RegisterNatives(JNIEnv* env, jclass jclazz,
    const JNINativeMethod* methods, jint nMethods)
{
    ScopedJniThreadState ts(env);
    ClassObject* clazz = (ClassObject*) dvmDecodeIndirectRef(ts.self(), jclazz);
    if (gDvm.verboseJni) {
        ALOGI("[Registering JNI native methods for class %s]",
            clazz->descriptor);
    }
    for (int i = 0; i < nMethods; i++) {
        if (!dvmRegisterJNIMethod(clazz, methods[i].name,
                methods[i].signature, methods[i].fnPtr))
        {
            return JNI_ERR;
        }
    }
    return JNI_OK;
}
```

老罗的Android之旅：[Dalvik虚拟机JNI方法的注册过程分析](http://blog.csdn.net/luoshengyang/article/details/8923483)

```cpp
/*
 * Register a method that uses JNI calling conventions.
 */
static bool dvmRegisterJNIMethod(ClassObject* clazz, const char* methodName,
    const char* signature, void* fnPtr)
{
    if (fnPtr == NULL) {
        return false;
    }
    // If a signature starts with a '!', we take that as a sign that the native code doesn't
    // need the extra JNI arguments (the JNIEnv* and the jclass).
    bool fastJni = false;
  	//如果签名以'!’时，我们认为这表明native代码没有
    if (*signature == '!') {
        fastJni = true;
        ++signature;
        ALOGV("fast JNI method %s.%s:%s detected", clazz->descriptor, methodName, signature);
    }
    //检查methodName是否是clazz的一个非虚成员函数
    Method* method = dvmFindDirectMethodByDescriptor(clazz, methodName, signature);
    if (method == NULL) {
        //检查methodName是否是clazz的一个虚成员函数。
        method = dvmFindVirtualMethodByDescriptor(clazz, methodName, signature);
    }
    //方法没找到
    if (method == NULL) {
        dumpCandidateMethods(clazz, methodName, signature);
        return false;
    }
    //确保类clazz的成员函数methodName确实是声明为JNI方法，即带有native修饰符
    if (!dvmIsNativeMethod(method)) {
        ALOGW("Unable to register: not native: %s.%s:%s", clazz->descriptor, methodName, signature);
        return false;
    }
    if (fastJni) {
      	//一个JNI方法如果声明为同步方法，即带有synchronized修饰符
        // In this case, we have extra constraints to check...
        if (dvmIsSynchronizedMethod(method)) {
            // Synchronization is usually provided by the JNI bridge,
            // but we won't have one.
            ALOGE("fast JNI method %s.%s:%s cannot be synchronized",
                    clazz->descriptor, methodName, signature);
            return false;
        }
        if (!dvmIsStaticMethod(method)) {
            // There's no real reason for this constraint, but since we won't
            // be supplying a JNIEnv* or a jobject 'this', you're effectively
            // static anyway, so it seems clearer to say so.
            ALOGE("fast JNI method %s.%s:%s cannot be non-static",
                    clazz->descriptor, methodName, signature);
            return false;
        }
    }
  	//获得Method对象method，用来描述要注册的JNI方法所对应的Java类成员函数。
  	//当一个Method对象method描述的是一个JNI方法的时候，它的成员变量nativeFunc保存的就是该JNI方法的地址，但是在对应的JNI方法注册进来之前，该成员变量的值被统一设置为dvmResolveNativeMethod。
    //函数dvmResolveNativeMethod此时会在Dalvik虚拟内部以及当前所有已经加载的共享库中检查是否存在对应的JNI方法。
    if (method->nativeFunc != dvmResolveNativeMethod) {
        /* this is allowed, but unusual */
        ALOGV("Note: %s.%s:%s was already registered", clazz->descriptor, methodName, signature);
    }
    method->fastJni = fastJni;
    //一个JNI方法是可以重复注册的，无论如何，函数dvmRegisterJNIMethod都是调用另外一个函数dvmUseJNIBridge来继续执行注册JNI的操作。
    dvmUseJNIBridge(method, fnPtr);
    ALOGV("JNI-registered %s.%s:%s", clazz->descriptor, methodName, signature);
    return true;
}
//如果我们在Dalvik虚拟机启动的时候，通过-Xjnitrace选项来指定了要跟踪参数method所描述的JNI方法，那么函数dvmUseJNIBridge为该JNI方法选择的Bridge函数就为dvmTraceCallJNIMethod，否则的话，就再通过另外一个函数dvmSelectJNIBridge来进一步选择一个合适的Bridge函数。
static bool shouldTrace(Method* method) {
    const char* className = method->clazz->descriptor;
    // Return true if the -Xjnitrace setting implies we should trace 'method'.
    if (gDvm.jniTrace && strstr(className, gDvm.jniTrace)) {
        return true;
    }
    // Return true if we're trying to log all third-party JNI activity and 'method' doesn't look
    // like part of Android.
    if (gDvmJni.logThirdPartyJni) {
        for (size_t i = 0; i < NELEM(builtInPrefixes); ++i) {
            if (strstr(className, builtInPrefixes[i]) == className) {
                return false;
            }
        }
        return true;
    }
    return false;
}

/*
 * Point "method->nativeFunc" at the JNI bridge, and overload "method->insns"
 * to point at the actual function.
 */
//它主要就是根据Dalvik虚拟机的启动选项来为即将要注册的JNI选择一个合适的Bridge函数。
void dvmUseJNIBridge(Method* method, void* func) {
    method->shouldTrace = shouldTrace(method);
    // Does the method take any reference arguments?
    method->noRef = true;
    const char* cp = method->shorty;
    while (*++cp != '\0') { // Pre-increment to skip return type.
        if (*cp == 'L') {
            method->noRef = false;
            break;
        }
    }
    DalvikBridgeFunc bridge = gDvmJni.useCheckJni ? dvmCheckCallJNIMethod : dvmCallJNIMethod;
  	//选择好Bridge函数之后，函数dvmUseJNIBridge最终就调用函数dvmSetNativeFunc来执行真正的JNI方法注册操作。
    dvmSetNativeFunc(method, bridge, (const u2*) func);
}
```

https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/oo/Class.cpp

```cpp
//参数method表示要注册JNI方法的Java类成员函数
//参数func表示JNI方法的Bridge函数
//参数insns表示要注册的JNI方法的函数地址。
void dvmSetNativeFunc(Method* method, DalvikBridgeFunc func,
    const u2* insns)
{
    ClassObject* clazz = method->clazz;
    assert(func != NULL);
    /* just open up both; easier that way */
    dvmLinearReadWrite(clazz->classLoader, clazz->virtualMethods);
    dvmLinearReadWrite(clazz->classLoader, clazz->directMethods);
    //当参数insns的值不等于NULL的时候，函数dvmSetNativeFunc就分别将参数insns和func的值分别保存在参数method所指向的一个Method对象的成员变量insns和nativeFunc中
    if (insns != NULL) {
        /* update both, ensuring that "insns" is observed first */
        method->insns = insns;
        android_atomic_release_store((int32_t) func,
            (volatile int32_t*)(void*) &method->nativeFunc);
    } else {
      	//当insns的值等于NULL的时候，函数dvmSetNativeFunc就只将参数func的值保存在参数method所指向的一个Method对象成员变量nativeFunc中。
        /* only update nativeFunc */
        method->nativeFunc = func;
    }
    dvmLinearReadOnly(clazz->classLoader, clazz->virtualMethods);
    dvmLinearReadOnly(clazz->classLoader, clazz->directMethods);
}
```

动态注册阅读到这里就可以了。

### so方法延迟解析出错

上面抛出的`UnsatisfiedLinkError`也许会有人疑问，为什么是`Native method not found`而不是我们开始遇到的崩溃 `No implementation found for`。可能是因为系统版本不一致导致的，但因为目前没有找到`Android10`这块的代码所有没有办法一查到底。

https://android.googlesource.com/platform/dalvik.git/+/android-4.2.2_r1/vm/Exception.cpp

```cpp
void dvmThrowUnsatisfiedLinkError(const char* msg) {
    dvmThrowException(gDvm.exUnsatisfiedLinkError, msg);
}
void dvmThrowUnsatisfiedLinkError(const char* msg, const Method* method) {
    char* desc = dexProtoCopyMethodDescriptor(&method->prototype);
    char* className = dvmDescriptorToDot(method->clazz->descriptor);
    dvmThrowExceptionFmt(gDvm.exUnsatisfiedLinkError, "%s: %s.%s:%s",
        msg, className, method->name, desc);
    free(className);
    free(desc);
}
```

https://android.googlesource.com/platform/dalvik.git/+/android-4.3_r3/vm/Globals.h

```cpp
/*
 * All fields are initialized to zero.
 *
 * Storage allocated here must be freed by a subsystem shutdown function.
 */
struct DvmGlobals {
    /******部分代码省略*****/
    ClassObject* exUnsatisfiedLinkError;
}
```

`DvmGlobals` 这个是`Dalvik`虚拟机在进行初始化的时候所加载的一些基础类，它们在Java的Libcore里面定义。

```cpp
static struct { ClassObject** ref; const char* name; } classes[] = {
		/*
     * Note: The class Class gets special treatment during initial
     * VM startup, so there is no need to list it here.
     */
    { &gDvm.exUnsatisfiedLinkError,            "Ljava/lang/UnsatisfiedLinkError;" },
    /******部分代码省略*****/
}
```

## 利用JNI_OnLoad替换系统的native方法

目标：尝试替换`android/util/Log`的`isLoggable`放方法；

```cpp
#include <jni.h>
#include <string>
#include <android/log.h>
#define TAG "tanzhenxing22-jni" // 这个是自定义的LOG的标识
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG,TAG ,__VA_ARGS__) // 定义LOGD类型

extern "C"
JNIEXPORT jboolean JNICALL
isLoggable(JNIEnv *env, jclass clazz, jstring tag, jint level) {
    LOGD("call isLoggable");
    return false;
}

extern "C"
JNIEXPORT jint JNICALL
//System.loadLibrary方法会调用载入的.so文件的函数列表中查找JNI_OnLoad函数并执行
JNI_OnLoad(JavaVM* vm, void* reserved) {
    LOGD("JNI_OnLoad");
    static JNINativeMethod methodsLog[] = {
            {"isLoggable", "(Ljava/lang/String;I)Z", (void *)isLoggable}
    };
    JNIEnv *env = NULL;
    jint result = -1;
    // 获取JNI env变量
    if (vm->GetEnv((void**) &env, JNI_VERSION_1_6) != JNI_OK) {
        // 失败返回-1
        return result;
    }

    // 获取native方法所在类
    const char* classNameLog = "android/util/Log";
    jclass clazzLog = env->FindClass(classNameLog);
    if (clazzLog == NULL) {
        return result;
    }
    // 动态注册native方法
    if (env->RegisterNatives(clazzLog, methodsLog, sizeof(methodsLog) / sizeof(methodsLog[0])) < 0) {
        return result;
    }
    // 返回成功
    result = JNI_VERSION_1_6;
    return result;
}

```

Demo地址：https://github.com/stven0king/JNIApplication

参考文章：

https://blog.csdn.net/fireroll/article/details/50102009

https://www.kancloud.cn/alex_wsc/androids/473614

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！
