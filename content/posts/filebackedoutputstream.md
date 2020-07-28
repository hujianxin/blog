---
title: 慎用FileBackedOutputStream
description: FileBackedOutputStream
date: 2019-08-18
categories:
  - "开发设计"
tags:
  - "FileBackedOutputStream"
  - "finalized"
draft: false
---

在项目中遇到一个FileBackedOutputStream的坑，在此记录一下。

因为某些原因，需要实现一个ResettableInputStream，大家都知道，JDK中，ByteArrayInputStream是可以mark、reset的，但是代价就是将所有数据都load进内存，在数据量较大时，这是无法接受的。因为我们需要的是任意的在流中穿梭的能力，这时候，可以借助磁盘来完成这一功能。索性，Guava中提供了一个类，FileBackedOutputStream，将数据写入其中之后，如果数据超过一定限度，它就会将数据写入磁盘。


FileBackedOutputStream的构造函数如下所示，里面提供了一个叫supplier的类，这个类作为OutputStream的出口。这里应该注意一下，如果resetOnFinalize这一个参数如果设为true的话，supplier会重写finalize方法，里面调用reset操作。这也是本篇文章出现的原因。
```java
  public FileBackedOutputStream(int fileThreshold, boolean resetOnFinalize) {
    this.fileThreshold = fileThreshold;
    this.resetOnFinalize = resetOnFinalize;
    memory = new MemoryOutput();
    out = memory;

    if (resetOnFinalize) {
      supplier = new InputSupplier<InputStream>() {
        @Override
        public InputStream getInput() throws IOException {
          return openStream();
        }

        @Override protected void finalize() {
          try {
            reset();
          } catch (Throwable t) {
            t.printStackTrace(System.err);
          }
        }
      };
    } else {
      supplier = new InputSupplier<InputStream>() {
        @Override
        public InputStream getInput() throws IOException {
          return openStream();
        }
      };
    }
  }

  public InputSupplier<InputStream> getSupplier() {
    return supplier;
  }
```
<!--more-->

## 通过FileBackedOutputStream实现ResettableInputStream
其实有了FileBackedOutputStream，实现ResettableInputStream就非常简单这里就给出一个简单的例子：
```java
public class ResettableFileBackedInputStream extends InputStream {
    protected long pos;
    protected long mark;
    private final FileBackedOutputStream outputStream;
    private InputStream inputStream;

    public ResettableFileBackedInputStream(FileBackedOutputStream fileBackedOutputStream) throws IOException {
        outputStream = fileBackedOutputStream;
        inputStream = fileBackedOutputStream.getSupplier().getInput();
        pos = 0;
        mark = 0;
    }

    @Override
    public boolean markSupported() {
        return true;
    }

    @Override
    public synchronized void mark(int readlimit) {
        mark = pos;
    }

    @Override
    public synchronized void reset() throws IOException {
        close();
        inputStream = outputStream.getSupplier().getInput();
        pos = 0;
        long skipped = skip(mark);
        if (skipped != mark) {
            throw new IOException("Error when skip inputStream: skip " + mark + ", return " + skipped);
        }
    }

    @Override
    public long skip(long n) throws IOException {
        long skipped = inputStream.skip(n);
        pos += skipped;
        return skipped;
    }

    @Override
    public int read() throws IOException {
        int r = inputStream.read();
        ....
    }

    @Override
    public int read(byte b[], int off, int len) throws IOException {
        int readBytes = inputStream.read(b, off, len);
        ....
    }

    @Override
    public void close() throws IOException {
        inputStream.close();
    }
}
```

很简单，因为一切内容都是在FileBackedOutputStream的supplier里面完成的。

## FileBackedOutputStream原理
### 1. 新写的内容都是往MemoryOutput里面写
FileBackedOutputStream里面，维护这一个ByteArrayOutputStream，用来写入新的数据。
```java
private static class MemoryOutput extends ByteArrayOutputStream {
    byte[] getBuffer() {
        return buf;
    }

    int getCount() {
        return count;
    }
}
```

在写入数据时，都是往这个OutputStream里面写：
```java
@Override public synchronized void write(int b) throws IOException {
    update(1); // 暂时忽略
    out.write(b);
}
```

在读取数据时，返回的是：
```java
return new ByteArrayInputStream(
          memory.getBuffer(), 0, memory.getCount());
```

所以上面的还是非常简单的。

### 2. 每次写时，都检查是否超过阈值
因为不能将全部内容都写进内存，所以在write的开始有一个update方法，update方法中步骤如下：
1. 如果超出了内存阈值，并且file不存在
2. 创建一个临时文件
3. 将ByteArrayOutputStream里面的数据写入到这个文件
4. 将out从ByteArrayOutputStrema变成这个FileOutputStream
5. 以后调用write的时候，都是往这个文件里写了。
6. 从这个OutputStream里面获取InputStream的时候，直接在这个文件中获取InputStream就可以。

## 资源的销毁
FileBackedOutputStream的大体原理已经介绍完了，但是重要的是资源还没有删除。在这个类里面，提供了一个reset方法，用来将OutputStream恢复到初始状态，顺便在里面完成了文件资源的删除工作。

可能是害怕调用者忘记调用reset来删除资源，所以在构造函数中，如果指定resetOnFinalize的时候，会重写finalized方法。在finalized中来删除资源。

但是，finalized本身是不能够保证被立刻执行的，可能会因为某些原因导致长达几分钟甚至几十分钟不被调用，最后导致reset不能够被及时调用。

## 不建议使用finalized
finalized是Object类的方法。finalized被调用的过程如下：

1. JVM在垃圾回收会检查对象是否重写过finalized方法
2. 如果重写过，则把对象放入到一个叫F-Queue里面
3. 一个低优先级的线程Finalizer线程去执行这些方法
4. 如果在执行过程中，这个对象重新引用了自己，则会被垃圾回收的清单中清楚掉，复活了

由此可见，这个finalize方法什么时候被调用，取决与其他的finalize方法什么时候被调用，甚至取决与线程什么时候被调度。是完全不可控的，如果指望这在这里面释放资源，可能会造成大问题。而且这个对垃圾回收造成困难。