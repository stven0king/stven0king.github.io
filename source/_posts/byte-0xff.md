---
title: byte和0XFF的基础
date: 2019-6-14 14:25:00
tags: [byte,0xff,补码]
categories: 计算机&网络
description: "最近在做代码相关的优化，找到了一个二进制转十六进制的方法。感觉很有意思找回了一部分还给老师的知识~！~！ "
---

# 前言
最近在做代码相关的优化，找到了一个二进制转十六进制的方法：

```java
/**
 * 二进制转16进制
 * @param bin
 * @return 16进制字符串
 */
public static String asHex(byte[] bin) {
    //一个byte为8位，一个十六进制为4位，所以长度乘以2
    StringBuilder bfHex = new StringBuilder(bin.length * 2);
    int i;
    for (i = 0; i < bin.length; i++) {
        if (((int) bin[i] & 0xff) < 0x10)
            //如果小于10，那么转化为16进制需要往前面加0，凑足两位
            bfHex.append("0");
        //转为为16进制字符串
        bfHex.append(Integer.toString((int) bin[i] & 0xff, 16));
    }
    return bfHex.toString();
}
```

# 为什么byte转int需要与0XFF

> 0XFF = {[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0],[1,1,1,1,1,1,1,1]};

- 一个 `int` 类型的数字 `&0XFF` ，就是将这个数字的高24位全部置为0；

##  byte转化为一个int类型直接强转不行么？

- 一个`byte` 转化为32位的 `int` 类型，它的数值大小不会发生任何变化；
- 如果是正数那么高位自动补充 `0`；
- 如果是负数那么高位补充的是 `1` ；

我们先看一段代码：

```java
public static void main(String[] args) {
    System.out.println(Integer.toBinaryString(1));
    System.out.println(Integer.toBinaryString(-1));
}
```

输出结果：

```
1
11111111111111111111111111111111
```

- 学习计算机原理的时候我们知道计算机都是采用补码存储数据的；

```
1 = byte[00000001] = int {000000000000000000000000000000001} = 0x1
-1 = byte[11111111] = int {11111111111111111111111111111111} = 0xffffffff
```

我们在做二进制转16进制的时候，需要的是数据的正确性而不是数值的正确性。所以我们进行 `0XFF` 的时候抹掉了高24位，确保了数据二进制补码的完整新（同时也解释了转化的16进制如果小于10需要在前面加0的原因）。

![byte&0xff.png](https://upload-images.jianshu.io/upload_images/1319879-c6e40d69e7d6c673.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们从 `BufferedInputStream.java` 源代码中都可以找到获取下一个字节的方法 `int read()` ，最后的得到的字节也是需要 `&0xff` 转化为 `int` 类型进行返回的，这样才能保证数据的完整性。

# 总结

## 源码、反码、补码

### 源码

```
[+1]原 = 0000 0001
[-1]原 = 1000 0001
```

原码就是符号位加上真值的绝对值, 即用第一位表示符号, 其余位表示值.;

### 反码

```
[+1] = [00000001]原 = [00000001]反
[-1] = [10000001]原 = [11111110]反
```

正数的反码是其本身;
负数的反码是在其原码的基础上, 符号位不变，其余各个位取反;

### 补码

```
[+1] = [00000001]原 = [00000001]反 = [00000001]补
[-1] = [10000001]原 = [11111110]反 = [11111111]补
```

正数的补码就是其本身;
负数的补码是在其原码的基础上, 符号位不变, 其余各位取反, 最后+1. (即在反码的基础上+1);

补码的设计有意识的引用了模运算在数理上对符号位的自动处理，利用模的自动丢弃实现了符号位的自然处理，仅仅通过编码的改变就可以在不更改机器物理架构的基础上完成的预期的要求（将减法变为加法），所以补码沿用至今。

## 计算机巧妙地把符号位参与运算, 并且将减法变成了加法, 背后蕴含了怎样的数学原理呢?

将钟表想象成是一个1位的12进制数. 如果当前时间是6点, 我希望将时间设置成4点, 需要怎么做呢?我们可以:

> 1. 往回拨2个小时: 6 - 2 = 4
> 2. 往前拨10个小时: (6 + 10) mod 12 = 4
> 3. 往前拨10+12=22个小时: (6+22) mod 12 =4

2,3方法中的mod是指取模操作, 16 mod 12 =4 即用16除以12后的余数是4。 [同余定理](https://baike.baidu.com/item/%E5%90%8C%E4%BD%99%E5%AE%9A%E7%90%86/1212360?fromtitle=%E5%90%8C%E4%BD%99&fromid=1432545)
所以钟表往回拨(减法)的结果可以用往前拨(加法)替代!

首先要确定二进制中模到底是多少。

> 在二进制加法运算中膜有0~127，膜为128（128个数），即为2的n次方。
> 2的n次方减一 = A + A[反]
> 2的n次方 = A + A[反] + 1
> A[补] = A[反] + 1
> 2的n次方 = A + A[补]

所以可以将负数用补码方式进行变化进行 `加操作` ，符号为也可以参与运算。

文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！