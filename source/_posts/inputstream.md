---
title: 细说InputStream和OutputStream
date: 2019-6-14 20:45:00
tags: [inputstream,outputstream]
categories: Java
description: "我们进行Android开发的时候经常会遇到各种 `io` 操作， 比如网络请求，文件操作，数据传输等。Java中的 `InputStream` 和 `OutputStream` 都是 `io` 包中面向字节操作的顶级抽象类，关于java同步 `io`字节流的操作都是基于这两个的。 "
---

# 前言

我们进行Android开发的时候经常会遇到各种 `io` 操作， 比如网络请求，文件操作，数据传输等。

Java中的 `InputStream` 和 `OutputStream` 都是 `io` 包中面向字节操作的顶级抽象类，关于java同步 `io`字节流的操作都是基于这两个的。



>- 网络数据传输：`SocketInputStream` 和 `SocketOutputStream`
>- 文件操作：`FileInputStream` 和 `FileOutputStream`
>- 字节数据操作：`DataInputStream` 和 `DataOutputStream`



# InputStream

```java
package java.io;
public abstract class InputStream implements Closeable {

    //MAX_SKIP_BUFFER_SIZE用于确定最大缓冲区大小
    //在跳过时使用  
    private static final int MAX_SKIP_BUFFER_SIZE = 2048;

    //从输入流中读取下一个byte的数据，返回的是一个0~255之间的int类型数。
    //如果已经到了流的末尾没有byte数据那么返回-1。
    //此方法阻塞，直到输入数据可用、检测到流的末尾或抛出异常。
    public abstract int read() throws IOException;

    //从输入流中读取一些字节，并将其存储到缓冲区数组b。
    //实际读取的字节数是以整数形式返回。此方法将阻塞，知道输入数据为止可用，检测到文件结尾，或抛出异常。
    //如果b的长度为0，则不读取任何字节，返回0。如果没有可用的字节，因为流是在文件结尾，返回值-1.
    //读取的字节数是：最多读取b的长度。
    public int read(byte b[]) throws IOException {
        return read(b, 0, b.length);
    }

    //从输入流中读取长度len的byte数据到一个数组中。
    //尝试尽可能的读取len长度的byte，但是可能读取较小的长度内荣，返回的是实际获取到的数据长度。
    public int read(byte b[], int off, int len) throws IOException {
        if (b == null) {
            throw new NullPointerException();
        } else if (off < 0 || len < 0 || len > b.length - off) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return 0;
        }

        int c = read();
        if (c == -1) {
            return -1;
        }
        b[off] = (byte)c;

        int i = 1;
        try {
            for (; i < len ; i++) {
                c = read();
                if (c == -1) {
                    break;
                }
                b[off + i] = (byte)c;
            }
        } catch (IOException ee) {
        }
        return i;
    }

    //从输入流中跳过和丢弃n个byte数据。
    //返回实际丢弃的数据长度。
    public long skip(long n) throws IOException {

        long remaining = n;
        int nr;

        if (n <= 0) {
            return 0;
        }

        int size = (int)Math.min(MAX_SKIP_BUFFER_SIZE, remaining);
        byte[] skipBuffer = new byte[size];
        while (remaining > 0) {
            nr = read(skipBuffer, 0, (int)Math.min(size, remaining));
            if (nr < 0) {
                break;
            }
            remaining -= nr;
        }

        return n - remaining;
    }

    //估计可以读取（或跳过的）字节数,下一次调用可能是同一个线程或另一个线程。
    //单个读取或跳过此操作许多字节不会阻塞，但可能会读取或跳过更少的字节。
    public int available() throws IOException {
        return 0;
    }

    //关闭此输入流，并释放与这个流有关的任何系统资源
    public void close() throws IOException {}

    //标记输入流当前的位置，随后调用reset放在在最后的标记位置重新定位，以便后续读取相同的字节。
    //readlimit 参数标识此输入流允许在标记失效为止之前获取的字节数。
    public synchronized void mark(int readlimit) {}

    //将此输入流重新定位到上次调用mark方法标记的地方。
    //如果markSupported方法返回true，或者自从上次标记之后从该输入流中读取的数据大于标记的长度可能会抛出IOException。
    //如果markSupported方法返回false,那么调用改方法可能抛出IOException。
    public synchronized void reset() throws IOException {
        throw new IOException("mark/reset not supported");
    }

    // 测试这个输入流是否支持标记和充值。
    public boolean markSupported() {
        return false;
    }

}
````



# OutputStream


```java
package java.io;

public abstract class OutputStream implements Closeable, Flushable {
    //将指定的字节写入输出流中，一般来说要写入的这个字节是参数的低8位，高24位忽略。
    public abstract void write(int b) throws IOException;

    // 从指定的byte数组中写入到该输出流
    public void write(byte b[]) throws IOException {
        write(b, 0, b.length);
    }

    //从byte数组的off开始，想输出流中写入len长度的数据。可能空指针和数据越界异常。
    public void write(byte b[], int off, int len) throws IOException {
        if (b == null) {
            throw new NullPointerException();
        } else if ((off < 0) || (off > b.length) || (len < 0) ||
                   ((off + len) > b.length) || ((off + len) < 0)) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return;
        }
        for (int i = 0 ; i < len ; i++) {
            write(b[off + i]);
        }
    }

    // 清空输出流，强制将缓冲器的输出的数据被写入。
    public void flush() throws IOException {
    }

    //关闭此输出流并释放所有系统资源 
    public void close() throws IOException {
    }
}
```



# 使用

因为 `InputStream` 和 `OutputStream` 的 `io` 操作中的‘输入’和‘输出’是不可靠的，发生的异常是不受程序控制。都会有 `IOExcepiton` 异常抛出，所以我们在使用的时候需要进行 `try/catch` 。

```java
InputStream in = null;
try {
  in = new ByteArrayInputStream(bytes)；
  //部分代码省略
} catch(IOException e) {
  //部分代码省略
} finally {
  try {
    if (in != null) {
      in.close();
    }
  } catch(IOException e) {
     //部分代码省略
  }
}
```

java7支持`try-with-resources`方式关闭流，实现了`java.lang.AutoCloseable`接口的对象支持用下边的方式进行流的关闭：

```java
public interface AutoCloseable {
    void close() throws Exception;
}
```

```java
private static void printFileJava7() throws IOException {
    try(FileInputStream input = new FileInputStream("file.txt")) {

        int data = input.read();
        while(data != -1){
            System.out.print((char) data);
            data = input.read();
        }
    }
}
```

也支持一次性关闭多个：

```java
private static void printFileJava7() throws IOException {

    try(  FileInputStream     input         = new FileInputStream("file.txt");
          BufferedInputStream bufferedInput = new BufferedInputStream(input)
    ) {

        int data = bufferedInput.read();
        while(data != -1){
            System.out.print((char) data);
    data = bufferedInput.read();
        }
    }
}
```

# 相关实现类 

<center>![Closeable.png](https://upload-images.jianshu.io/upload_images/1319879-3dd7958bdc726ea8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)</center>



文章到这里就全部讲述完啦，若有其他需要交流的可以留言哦~！~！