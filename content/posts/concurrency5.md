---
title: Java并发编程问题汇总一：ThreadLocal
description: ThreadLocal
date: 2019-04-12
categories:
  - "开发设计"
tags:
  - "Java"
  - "Concurrency"
  - "ThreadLocal"
draft: true
---
ThreadLocal是比较常用的多线程工具，有效理解ThreadLocal的内部原理，对自信地使用它具有益处。

许多网友认为它是多线程环境下多线程数据共享的手段，其实这种想法是对ThreadLocal最大的误解。内存数据共享是并发编程中需要处理的一大难点，但是ThreadLocal却不是为了解决数据共享问题而生的。顾名思义，ThreadLocal为每个线程提供一个本地（本线程）副本，而这些副本之间，从出生到结束，都是没有丝毫联系的，它们仿佛是生活在平行宇宙中的同名个体，之间没有交际，互补影响，更何谈数据共享。

## Thread可以作为ThreadLocal内部Map的Key？
给同事介绍介绍ThreadLocal的时候，我经常说，你可以想象里面有一个HashMap，Key就是Thread本身，Value就是Thread对应的值，虽然对不懂的同事而言，这样的解释能让其更快的了解代码，使用ThreadLocal，但其实这是一种误人子弟的说法。

首先，简单思考一下，如果Key是Thread，Value是值的话，那这个HashMap实例本身存在什么地方？答案是，并没有这样一个超然于所有线程之外的地方用来存放这个map。就算是强行搞出一个这样的map，还是会出现并发问题，需要进行并发控制，脱离了ThreadLocal存在的意义。

更重要的是，在我们复杂的程序中，一个线程会有多个ThreadLocal变量，如果使用Thread作为key的话，只能存在一个Value，与实际使用场景严重不符。

那么，使用Thread当作Key的做法明显是不可行，既没有存放map的地方，也没法让一个Thread对应多个ThreadLocal变量，ThreadLocal中是怎么实现的呢？答案其实已经呼之欲出了：ThreadLocal对象本身作为key，ThreadLocalMap对象放到了每个Thread中。

## ThreadLocal与Thread共生

## 神奇的数字u0x61c88647

## WeakReference

## 使用场景