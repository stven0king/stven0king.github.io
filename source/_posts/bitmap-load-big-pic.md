---
title: Bitmap图片压缩，大图加载防止OOM
date: 2020-4-22 18:55:00
tags: [Bitmap]
categories: Android
description: "Android官网中【处理位图】和【高效加载大型位图】两篇文章中已经做了很明确指出了如何高效的加载大图。这篇文章只是对其中的内容进行总结和扩展（比如图片内存计算、图片压缩等）。 "
---


![在这里插入图片描述](https://img-blog.csdnimg.cn/20200422150554242.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N0dmVuX2tpbmc=,size_16,color_FFFFFF,t_70#pic_center)

# 前言
Android官网中[处理位图](https://developer.android.com/topic/performance/graphics) 和 [高效加载大型位图
](https://developer.android.com/topic/performance/graphics/load-bitmap) 这两篇文章中已经做了很明确指出了如何高效的加载大图。这篇文章只是对其中的内容进行总结和扩展（比如图片内存计算、图片压缩等）。

为了防止加载 `Bitmap` 的时候造成 `OOM` 崩溃，我们首选要知道：
- 一张图片加载到 `Bitmap` 的时候的占用的是怎么内存计算；
- 占用内存过高的时候怎么进行图片压缩减小内存占用；

# RGB介绍
> RGB颜色模型: 最常见的颜色模型，设备相关。R、G、B分别代表红、绿和蓝色三种颜色通道，取值均为[0,255]。

> RGB 8位色: 表示使用8位(bit)表示颜色，一共能表示2^8 = 128种颜色。
依次类推RGB 16位色，RGB 24位色，RGB 32位色，使用的位数越多，能表示的颜色越多，24位能表示的颜色数量已经很多了，称之为“真彩色”。

> 32位和24位能表示的颜色一样多，多一个了透明度。

> Android Bitmap使用的三种颜色格式：
>- ALPHA_8–每个像素占1个字节，存储透明度信息，没有颜色信息。
>- RGB_565--每个像素占2个字节存储颜色信息，R 5位，G 6位，B 5位，能表示2^16种颜色。
>- ARGB_8888--每个像素占4个字节存储颜色信息，A R G B各一个字节，能表示2^24种颜色，还有一个字节存储透明度信息。

# 图片占用内存的计算

`Bitmap` 所占内存大小计算方式：图片长度 x 图片宽度 x 一个像素点占用的字节数。

## 读取位图尺寸和类型
`BitmapFactory` 类提供了几种用于从各种来源创建 `Bitmap` 的解码方法`（decodeByteArray()、decodeFile()、decodeResource() `等）。根据您的图片数据源选择最合适的解码方法。这些方法尝试为构造的位图分配内存，因此很容易导致 `OutOfMemory` 异常。每种类型的解码方法都有额外的签名，允许您通过 `BitmapFactory.Options` 类指定解码选项。在解码时将` inJustDecodeBounds` 属性设置为 `true` 可避免内存分配，为位图对象返回 `null`，但设置 `outWidth`、`outHeight` 和 `outMimeType`。此方法可让您在构造位图并为其分配内存之前读取图片数据的尺寸和类型。

```java
BitmapFactory.Options options = new BitmapFactory.Options();
options.inJustDecodeBounds = true;
BitmapFactory.decodeResource(getResources(), R.id.myimage, options);
int imageHeight = options.outHeight;
int imageWidth = options.outWidth;
String imageType = options.outMimeType;
```

为避免出现 `java.lang.OutOfMemory` 异常，请先检查位图的尺寸，然后再对其进行解码，除非您绝对信任该来源可为您提供大小可预测的图片数据，以轻松适应可用的内存。


## 内存中如果加载一张 `500*500` 的 `png` 高清图片.应该是占用多少的内存?  
`png` 图片应该有**alpha通道**，所以  `Bitmap.Config` 是 `ARGB_8888` 。4个8位一种占用32位。
最终答案： `500 * 500 * 4 = 1000000Bytes = 0.95MB`

## 如果这个图片为本地资源图片，是否还是0.95MB呢？

先看一些基础知识（后面有答案） [Android官网-提供备用位图](https://developer.android.google.cn/training/multiscreen/screendensities#TaskProvideAltBmp) 这篇文章链接中的有讲到：

> 要在像素密度不同的设备上提供良好的图形质量，您应该以相应的分辨率在应用中提供每个位图的多个版本（针对每个密度级别提供一个版本）。否则，Android 系统必须缩放位图，使其在每个屏幕上占据相同的可见空间，从而导致缩放失真，如模糊。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9kZXZlbG9wZXIuYW5kcm9pZC5nb29nbGUuY24vaW1hZ2VzL3NjcmVlbnNfc3VwcG9ydC9kZXZpY2VzLWRlbnNpdHlfMngucG5n?x-oss-process=image/format,png#pic_center)

例如，如果您有一个可绘制位图资源，它在中密度屏幕上的大小为 48x48 像素，那么它在其他各种密度的屏幕上的大小应该为：
*   36x36 (0.75x) - 低密度 (ldpi)
*   48x48（1.0x 基准）- 中密度 (mdpi)
*   72x72 (1.5x) - 高密度 (hdpi)
*   96x96 (2.0x) - 超高密度 (xhdpi)
*   144x144 (3.0x) - 超超高密度 (xxhdpi)
*   192x192 (4.0x) - 超超超高密度 (xxxhdpi)

然后，将生成的图片文件放在 `res/` 下的相应子目录中，系统将根据运行应用的设备的像素密度自动选取正确的文件。之后，每当您引用`@drawable/xxx`时，系统都会根据屏幕的 `dpi` 选择适当的位图。如果您没有为某个密度提供特定于密度的资源，那么系统会选取下一个最佳匹配项并对其进行缩放以适合屏幕。

实测：`1520 x 2688` 大小为 `334.28KB` 图片，屏幕密度为480的手机；
- 放在 `drawable-xxdpi` 下加载到 `Bitmap` 中占用内存为 `16343040(1520*2688*4)`，因为图片不需要进行缩放，所以只需要计算 `ARGB_8888` 占用的字节数就行；
- 放在 `drawable-mdpi` 下加载到 `Bitmap` 中占用内存为 `147087360(1520*3*2688*3*4)` ，因为 `mdip` 到 `xxdpi` 图片的宽高分别会放大4倍；

`nodpi` 目录中的资源被视为与密度无关，系统将不会对它们进行缩放。

# Bitmap压缩
## 压缩原理

在 `Android` 中进行图片压缩是非常常见的开发场景，主要的压缩方法有两种：其一是下 **采样压缩**，其二是 **质量压缩**。

- 前者是降低图像尺寸，改变图片的存储体积；
- 后者则是在不改变图片尺寸的情况下，通过损失颜色精度，达到相同目的;


## 压缩Bitmap磁盘占用空间的大小

```java
//如果成功地把压缩数据写入输出流，则返回true。
public boolean compress(
    Bitmap.CompressFormat format, //图像的压缩格式；
    int quality,//图像压缩率，0-100。 0 压缩100%，100意味着不压缩；
    OutputStream stream) ;//写入压缩数据的输出流；
```

- `Bitmap.CompressFormat.PNG` ，那不管第二个值如何变化，图片大小都不会变化，不支持 `png图片` 的压缩。因为 `PNG` 格式是无损的，它无法再进行质量压缩，`quality `这个参数就没有作用了，会被忽略，所以最后图片保存成的文件大小不会有变化；
- `CompressFormat.WEBP` ，这个格式是 `google` 推出的图片格式，它会比 `JPEG` 更加省空间。官方表示能节省 `25%-34%` 的空间；

## 压缩Bitmap占用内存的大小

图片尺寸的修改其实就是通过修改像素数，放大的过程称之为**上采样**，缩小的过程称之为**下采样**。


要知道怎么压缩才能使 `Bitmap` 占用的内存变小，首先需要知道 `Bitmap` 的内存占用怎么计算。 [计算图片的内存占用](https://www.kancloud.cn/book/stven\_king/stven\_king\_android\_interview\_topic/preview/Bitmap/%E8%AE%A1%E7%AE%97%E5%9B%BE%E7%89%87%E7%9A%84%E5%86%85%E5%AD%98%E5%8D%A0%E7%94%A8.md) 这篇文章有详细讲解。

### 使用inSampleSize进行压缩

既然图片尺寸已知，便可用于确定应将完整图片加载到内存中，还是应改为加载下采样版本。以下是需要考虑的一些因素：

- 在内存中加载完整图片的估计内存使用量。
- 根据应用的任何其他内存要求，您愿意分配用于加载此图片的内存量。
- 图片要载入到的目标 ImageView 或界面组件的尺寸。
- 当前设备的屏幕大小和密度。

例如，如果 1024x768 像素的图片最终会在 ImageView 中显示为 128x96 像素缩略图，则不值得将其加载到内存中。

要让解码器对图片进行下采样，以将较小版本加载到内存中，请在 `BitmapFactory.Options` 对象中将 `inSampleSize` 设置为 `true`。

例如，分辨率为 `2048x1536` 且以 `4` 作为 `inSampleSize` 进行解码的图片会生成大约 `512x384` 的位图。将此图片加载到内存中需使用 `0.75MB`，而不是完整图片所需的 `12MB`（假设位图配置为 `ARGB_8888`）。

下面的方法用于计算样本大小值，即基于目标宽度和高度的 `2` 的幂：

```java
public static int calculateInSampleSize(
            BitmapFactory.Options options, int reqWidth, int reqHeight) {
    // Raw height and width of image
    final int height = options.outHeight;
    final int width = options.outWidth;
    int inSampleSize = 1;
    if (height > reqHeight || width > reqWidth) {
        final int halfHeight = height / 2;
        final int halfWidth = width / 2;
        // Calculate the largest inSampleSize value that is a power of 2 and keeps both
        // height and width larger than the requested height and width.
        while ((halfHeight / inSampleSize) >= reqHeight
                && (halfWidth / inSampleSize) >= reqWidth) {
            inSampleSize *= 2;
        }
    }
    return inSampleSize;
}
```
> 注意：根据 `inSampleSize` 文档，计算 `2` 的幂的原因是解码器使用的最终值将向下舍入为最接近的 `2` 的幂。

要使用此方法，请先将 `inJustDecodeBounds` 设为 `true` 进行解码，传递选项，然后使用新的 `inSampleSize` 值并将 设为` false` 再次进行解码：

```java
public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,
        int reqWidth, int reqHeight) {
    // First decode with inJustDecodeBounds=true to check dimensions
    final BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(res, resId, options);
    // Calculate inSampleSize
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);
    // Decode bitmap with inSampleSize set
    options.inJustDecodeBounds = false;
    return BitmapFactory.decodeResource(res, resId, options);
}
```

`Android` 使用的 `inSampleSize` 计算采样率使用的采样算法是**邻近采样（Nearest Neighbour Resampling）**， `x`（`x` 为 2 的倍数）个像素最后对应一个像素。比如采样率设置为 `1/2` ，所以是两个像素生成一个像素。邻近采样的方式比较粗暴，直接选择其中的一个像素作为生成像素，另一个像素直接抛弃。

### 使用createScaledBitmap或Matrix

```java
Bitmap bitmap = BitmapFactory.decodeFile("/sdcard/test.png");
Bitmap compress = Bitmap.createScaledBitmap(bitmap, bitmap.getWidth()/2, bitmap.getHeight()/2, true);
//或者直接使用 matrix 进行缩放，查看Bitmap.createScaledBitmap源码其实就是使用 matrix 缩放
Bitmap bitmap = BitmapFactory.decodeFile("/sdcard/test.png");
Matrix matrix = new Matrix();
matrix.setScale(0.5f, 0.5f);
bm = Bitmap.createBitmap(bitmap, 0, 0, bit.getWidth(), bit.getHeight(), matrix, true);
```
同样是图片宽高各为原来的`1/2`，这种方式采用**双线性采样（Bilinear Resampling）**，这个算法不像邻近采样算法直接粗暴的选择一个像素，而是参考了源像素相应位置周围 `2x2` 个点的值，根据相对位置取对应的权重，经过计算之后得到目标图像。

不同的采样算法会产生不同效果，除了 `Android` 中这两种常用的采样算法之外，还有比较常见如：`双立方／双三次采样（Bicubic Resampling）` 和 `Lanczos Resampling` 等。如果对 `Android` 使用的这两种采样算法效果不满意，必要时可以引入其他的算法。

### BitmapFactory.Options三件套
> `inScaled` + `inDensity` + `inTargetDensity`

当**inScaled**设置为true时（设置此标志时），如果**inDensity**与**inTargetDensity**不为0，`Bitmap` 就会在加载的时候直接进行缩放以匹配 `inTargetDensity` ，而不是绘制的时候进行缩放。（加载到堆内存时已经缩放了大小了，`.9图` 会忽略此标志）

**inDensity**:加载图片的原始宽度，如果此密度与 `inTargetDensity` 不匹配，则在返回 `Bitmap`前会将它缩放至目标密度。
**inTargetDensity** :目标图片的显示宽度，它与 `inScaled` 与 `inDensity` 结合使用，确定如何在返回 `Bitmap` 前对其进行缩放。

前面讲述的计算 `Bitmap` 大小的第二个例子，就是将相同图片加载放到不同的 `drawable-dpi` 的文件目录下去加载到内存中的 `Bitmap` 大小不同，其原因就是 `inDensity` 和 `inTargetDensity` 不一致导致。

# Bitmap局部解码
[官网文档-BitmapRegionDecoder](https://developer.android.com/reference/android/graphics/BitmapRegionDecoder?hl=en) ，`BitmapRegionDecoder` 可用于解码图像中的矩形区域。当原始图像很大且只需要部分图像时，`BitmapRegionDecoder` 尤其有用。 要创建 `BitmapRegionDecoder`，请调用 `newInstance()` 。给定一个 `BitmapRegionDecoder`，用户可以重复调用 `encodeRegio()`以获取指定区域的解码后的 `Bitmap` 。

```java
try {
    inputStream = getResources().getAssets().open("qq.jpg");
    BitmapRegionDecoder mRegionDecoder = BitmapRegionDecoder.newInstance(inputStream, false);
    BitmapFactory.Options sOptions = new BitmapFactory.Options();
    sOptions.inPreferredConfig = Bitmap.Config.ARGB_8888;
    sOptions.inSampleSize = 2;
    Rect mRect = new Rect();
    mRect.top = 0;
    mRect.left = 0;
    mRect.right = 100;
    mRect.bottom = 100;
    Bitmap bitmap = mRegionDecoder.decodeRegion(mRect, sOptions);
    //bitmap.getByteCount()=40000
} catch (IOException e) {
    e.printStackTrace();
}
```

这里需要注意的是 `mRect` 的宽高不能太大，否则加载得到的 `Bitmap` 的时候也会出现 `OOM` 的异常。

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！
