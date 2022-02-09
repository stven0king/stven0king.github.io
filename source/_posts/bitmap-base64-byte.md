---
title: Android：Base64生产Bitmap压缩和转byte[]
date: 2019-01-11 09:32:00
tags: [Bitmap, base64, byte]
categories: Android
description: "最近在做微信分享的时候遇到了分享图片的大小限制问题，需要对图片进行压缩。在过程中遇到几个有趣的地方在此记录。"
---


最近在做微信分享的时候遇到了分享图片的大小限制问题，需要对图片进行压缩。在过程中遇到几个有趣的地方在此记录。

> Bitmap.getByteCount的大小和转化为byte[]的大小差很多不是8倍，而是几十倍，我自测的为67倍

> 压缩Bitmap直接根据长宽比进行调用 `createScaledBitmap(@NonNull Bitmap src, int dstWidth, int dstHeight, boolean filter)` 方法进行缩放，只能保证长宽不能保证质量。

```java
public class BitmapUtils {

    /**
     * 获取bitmap转化为字节的大小
     * @param bitmap
     * @return
     */
    public static int getBitmapByteSize(Bitmap bitmap) {
        if (bitmap == null) {
            return 0;
        } else {
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            bitmap.compress(Bitmap.CompressFormat.JPEG, 100, baos);
            int size = baos.toByteArray().length;
            try {
                baos.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
            return size;
        }
    }

    /**
     * 根据压缩图片到固定的大小,因为会进行多次压缩可能会比较耗时，建议在异步线程调用
     * @param bitmap    原始图片
     * @param maxSize   压缩后的大小
     * @param needRecycle   是否需要回收被压的图片
     * @return
     */
    public static byte[] compressBitmap(Bitmap bitmap, double maxSize, boolean needRecycle) {
        if (bitmap == null) {
            return null;
        } else {
            int width = bitmap.getWidth();
            int height = bitmap.getHeight();
            //计算等比缩放
            double x = Math.sqrt(maxSize / (width * height));
            Bitmap tmp = Bitmap.createScaledBitmap(bitmap, (int) Math.floor(width * x), (int) Math.floor(height * x), true);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int options = 100;
            //生产byte[]
            tmp.compress(Bitmap.CompressFormat.JPEG, options, baos);
            //判断byte[]与上线存储空间的大小
            if (baos.toByteArray().length > maxSize) {
                //根据内存大小的比例，进行质量的压缩
                options = (int) Math.ceil((maxSize / baos.toByteArray().length) * 100);
                baos.reset();
                tmp.compress(Bitmap.CompressFormat.JPEG, options, baos);
                //循环压缩
                while (baos.toByteArray().length > maxSize) {
                    baos.reset();
                    options -= 1.5;
                    tmp.compress(Bitmap.CompressFormat.JPEG, options, baos);
                }
                recycle(tmp);
                if (needRecycle) {
                    recycle(bitmap);
                }
            }
            byte[] data = baos.toByteArray();
            try {
                baos.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
            return data;
        }
    }

    /**
     * 回收Bitmap
     * @param thumbBmp  需要被回收的bitmap
     */
    public static void recycle(Bitmap thumbBmp) {
        if (thumbBmp != null && !thumbBmp.isRecycled()) {
            thumbBmp.recycle();
        }
    }
    /**
     * base64数据转byte[]
     * @param imageUrl
     */
    public static byte[] getImageDataWithBase64(String imageUrl) {
        byte[] data;
        if (TextUtils.isEmpty(imageUrl)) {
            return null;
        } else if (imageUrl.startsWith("data:image")) {
            data = android.util.Base64.decode(imageUrl.split(",")[1], Base64.DEFAULT);
        } else {
            data = android.util.Base64.decode(imageUrl, Base64.DEFAULT);
        }
        return data;
    }
}
```

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！