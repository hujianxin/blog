---
title: Java并发编程问题汇总四：ArrayList和CopyOnWriteArrayList
description: COW
date: 2019-04-11
categories:
  - "开发设计"
tags:
  - "Java"
  - "Concurrency"
  - "COW"
draft: false
---
Java编程中，ArrayList是最常用的数据结构之一。

ArrayList使用了数组来实现List结构。读取和插入速度非常快，删除效率较低，因为设计到了数据拷贝。

ArrayList用户广泛，但是不能直接使用在多线程环境下。<!--more-->

## ArrayList的并发问题
ArrayList所有接口都不是为并发环境而设计的，所以很容易分析出，在并发环境中，ArrayList会出现什么问题，下面一add方法举例：
```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

在add方法中，使用了size++这个表达式，Java在处理自增运算时，内存操作设计到了load、change、store三个操作，很明显这不是原子操作。所以多线程情况下访问add方法，可能会出现其中某线程add没作用的现象。

当然，从各个方面分析ArrayList，它都不是线程安全的，这里就不一一赘述了。

## ArrayList支持并发的简单解决方案
早期JDK提供了Vector，它提供了与ArrayList相关同的接口，也是List的子类。实现如下
```java
public synchronized boolean add(E e) {
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = e;
    return true;
}
```
可以看到，add方法（所有方法）都是使用synchronized修饰的，所以这个类是线程安全的。

但是随着现代编程理念越来越清晰，使用装饰器模式更简洁明了，更统一，心智负担也更低。所以Java目前推荐使用Collections.synchronizedList方法来对list修饰，将ArrayList转换成线程安全的实例。Collections.synchronizedList返回一个SynchronizedList类，这个类对原始类进行了装饰，加上了synchronized。所以也是线程安全的。

但是上面这两种方式处理简单、性能要求不高的业务还可以，但是对性能要求高的业务不能使用他们。因为他们的每次方法调用都使用了锁，性能不及格。

## CopyOnWriteArrayList
CopyOnWriteArrayList使用了写时复制技术，是一个线程安全的类，在处理读多写少的使用场景中，有非常好的表现。

CopyOnWriteArrayList为了保证线程安全，做了一下工作：
1. set、add方法，直接使用ReentrantLock加锁
2. 里面的数组使用了volatile，保证可见行，所以不需要加锁。
3. 创建迭代器时，将当前array传入。之后的写操作都会将array换成新的数组，所以不会影响到迭代器。

CopyOnWriteArrayList如何保证效率的呢？
1. 读操作不会加所，而是使用volatile修饰
2. set操作根据传入的新值是否与旧值相等，决定是否copy