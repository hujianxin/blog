---
title: 《深入理解Java虚拟机》
description: 读书笔记
date: 2020-04-18
categories:
  - "开发设计"
tags:
  - "Reading"
  -  "JVM"
draft: true
---

## JVM的内存布局和结构
### JVM布局
1. 共享区域
    1. 堆：对象建立的主要地方，垃圾回收的主要发生地
    2. 方法区：永久代，存放：类信息、常量、静态变量、JIT编译后的代码
2. 线程独占区域
    1. 虚拟机栈
    2. 本地方法栈
    3. 程序计数器
### 对象创建过程
1. 类加载检查：去方法区找到类的符号引用，并检查是否经过加载、解析、初始化，如果没有的话，先加载
2. 在空闲列表中找到一个块，去堆中分配内存
3. 初始化内存为0
4. 设置对象信息：类、hashCode，GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。
### 对象的内存布局
![对象的内存布局](https://github.com/hujianxin/blog/blob/master/static/img/java_object.png)
数组对象与普通对象相比，多了Length信息。

其中，Mark Word中包含了对象的大量信息。
包含了：
1. HashCode
2. GC分代年龄
3. 锁状态标志
4. 线程持有的锁
5. 偏向线程ID
6. 偏向时间戳
![Mark Word](https://github.com/hujianxin/blog/blob/master/static/img/mark_word.png)
### 对象的访问定位
![对象的访问定位](https://github.com/hujianxin/blog/blob/master/static/img/对象访问定位.png)

## GC和内存分配策略
### 引用计数法的思考
众所周知，引用计数法没法用在虚拟机实现中用以判断对象是否存活，但是引用计数法在其他领域却可能有发挥重要作用的用途。

例如，在某些不具备事务能力的数据库中，某个访问设计了几个数据库操作无法做到原子性，如果删除成功了一个表中的字段，而另一个表中却因为exception存留了下来，一般会造成不一致甚至丢数据的问题。

### 可达性分析
可达型分析主要是通过GC Roots遍历所有对象，所有遍历不到的对象都是需要被清理的对象。

可用的GC Roots：
1. 虚拟机栈（栈帧中的本地变量表）中的引用对象
2. 方法区中的 类静态属性引用的对象
3. 方法区中常量引用的对象
4. 本地方法栈中JNI引用的对象

另外，JVM中，Java对引用的概念进行了扩充：强引用，软引用，弱引用，虚引用
1. 强引用，最常见的引用
2. 软引用，在内存不足时，会被回收
3. 弱引用，无论如何都会被回收
4. 虚引用，无法获取到对象

### 垃圾回收算法
1. 标记清理：老年代uu
2. 复制：用在新生代
3. 标记整理：用在老年代

### 复制算法实际布局情况
JVM中的新生代算法都是使用复制算法，一个重要的原因来自，98%的对象都是朝生夕死，所以布局是这样的。
![gc布局](https://github.com/hujianxin/blog/blob/master/static/img/gc.png)

新生代分为Eden、Survivor From，Survivor To，默认，Eden占了80,Survivor分别占用了10%，这个比例可以通过-XX:SurvivorRatio来设置。
其中，Eden区是新对象创建的地方，Survivor分为两个区域的原因来自复制算法的要求，总需要有一个空闲的区域用来整理空间，虽然说是一种浪费，但是属于必须浪费的空间。

### 如何枚举GC Roots
一个一个枚举肯定不现实，HotSpot中，使用一组称为OopMap的数据结构达到这个目的。

### 安全点
安全点一般设置在：方法调用、循环跳转、异常等地方。

虚拟机让所有线程都跑到安全点之后，主动停下来。

### 各种垃圾回收器的特点和比较
1. Serial，新生代，串行垃圾回收，在client模式下默认的垃圾回收器。
    1. 简单高效
    2. 单核环境下效率很高
2. ParNew，新生代，并行垃圾回收，在Serial基础上，多线程垃圾回收。
    1. 多核的Server模式很有用
    2. CMS默认的新生代垃圾回收器
3. Parallel Scavenge，新生代
    1. 关注吞吐量，在用户延迟要求不高，但是对程序计算效率要求高的地方很有用
4. Serial Old，老年代，Serial的老年版
5. Parallel Old，老年代
    1. 为配合Parallel Scavenge而生
    2. 关注吞吐
6. CMS，老年代
    1. 使用最广泛的之一
    2. 延迟低
    3. 对CPU资源敏感，资源较少的话，吞吐会受到影响
    4. 无法收集浮动垃圾，如果浮动垃圾多的话，容易引发full gc
7. G1，包含了新生代+老年代的全功能垃圾回收器
    1. 并发并行
    2. 分代收集
    3. 空间整合
    4. 可预测停顿

### 内存分配和回收策略
1. 对象优先在Eden分配
    1. 当Eden没有足够空间时，JVM发起一次minor gc
    2. -Xms初始堆大小
    3. -Xmx最大堆大小
    4. -Xmn年轻带大小，-XX:SurvivorRatio设置Eden和Survisor的比例。
    5. -Xss每个线程的虚拟机栈大小
2. 大对象直接进入老年代
    1. -XX:PretenureSizeThreshold
3. 长期存活的对象进入老年代
    1. 默认-XX:MaxTenuringThreshold=15
4. 动态对象的年龄判定：同年对象达到Survisor空间的一半，会晋升到老年代
5. 空间分配担保：Minor GC之前，如果老年代最大的连续空间不足以放下新生代所有的对象，则Full GC

## 类文件结构
![类文件结构](https://github.com/hujianxin/blog/blob/master/static/img/class.png)

## 类加载机制
### 类加载时机
类加载到卸载，经历了下面过程：加载-》验证-》准备-》解析-》初始化-》使用-》卸载
![类加载过程](https://github.com/hujianxin/blog/blob/master/static/img/loading.png)

下面情况下开始类加载：
1. 遇到new，getstatic，putstatic，invokestatic这4个指令的时候
2. 使用java.lang.reflect包的方法对类进行反射调用的时候
3. 子类被初始化时
4. 包含main方法的类
5. 如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getStatic, REF_putStatic, REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行初始化，需要触发其初始化

### 类加载过程
1. 加载
    1. 通过类的全名，获取二进制字节流
    2. 转换成方法区运行时数据结构
    3. 在内存中，生成一个代表这个类的java.lang.Class对象，作为方法区这个类各种数据的访问入口
2. 验证
    1. 文件格式
    2. 元数据愈发验证
    3. 字节码验证
    4. 符号引用验证
3. 准备： 类变量的初始化
4. 解析
5. 初始化：cinit方法调用，各种filed的初始化
