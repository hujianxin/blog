---
title: Java并发编程问题汇总五：ThreadLocal
description: ThreadLocal
date: 2020-08-04
categories:
  - "开发设计"
tags:
  - "Java"
  - "Concurrency"
  - "ThreadLocal"
draft: false
---
ThreadLocal是比较常用的多线程工具，有效理解ThreadLocal的内部原理，对自信地使用它具有益处。

许多朋友认为它是多线程环境下处理**数据共享**的手段，其实这种想法是对ThreadLocal最大的误解。

内存数据共享是并发编程中需要处理的一大难点，但是ThreadLocal却不是为了解决数据共享问题而生的。顾名思义，ThreadLocal为每个线程提供一个本地（本线程）副本，而这些副本之间，从出生到结束，都是没有丝毫联系的，它们仿佛是生活在平行宇宙中的同名个体，之间没有交际，互补影响，更何谈数据共享。<!--more-->

## 使用场景
如上所说，ThreadLocal是为每个线程提供一个变量的本地副本，所以简单demo如下：
```java
private static final ThreadLocal<List<String>> buffer = new ThreadLocal<>(); // 每个线程的buffer

public static void main(String[] args) throws Exception{
  ThreadPool pool = new ThreadPool();
  pool.submit(()->{
    // 获取本线程的buffer
    List<String> list = buffer.get();
    if (list == null) {
      buffer.set(list);
    }
    // 使用buffer做一些工作
  });
}
```

## ThreadLocal与Thread共生
给同事介绍介绍ThreadLocal的时候，我经常说，你可以想象里面有一个HashMap，Key就是Thread本身，Value就是Thread对应的值，虽然对不懂的同事而言，这样的解释能让其更快的了解代码，使用ThreadLocal，但其实这是一种误人子弟的说法。

首先，简单思考一下，如果Key是Thread，Value是值的话，那这个HashMap实例本身存在什么地方？答案是，~~并没有这样一个超然于所有线程之外的地方用来存放这个map。~~，此处理解有误，ThreadLocalMap存放到一个线程里面，实际上就不需要考虑线程安全的问题了。

那么Java中是如何解决的呢？

答案很简单，也很精妙！在每个Thread中存一个Map，用ThreadLocal作为key。

仔细一想，这即精妙，又是唯一可能的实现方式。这就是ThreadLocal和Thread类的共生和联动了。

## ThreadLocalMap和HashMap
那既然在Thread中存放一个map，那是HashMap可不可以？

可以通过ThreadLocalMap的源码看到，Entry是一个WeakReference。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。

这是因为，一旦Thread结束，threadLocal变量也就变成了一个不可达的值，如果是强引用的话，会因为Thread中的map存在，导致gc无法回收它。如果是弱引用，则会被自动回收。

但是虽然Key被gc了，但是Value还是不会被gc，导致整个entry不会被gc，最后还是会出现内存泄漏，所以最好是调用remove方法。

## 参考
[透视ThreadLocal](https://zhuanlan.zhihu.com/p/37733237)