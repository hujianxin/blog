---
title: Java并发编程问题汇总一：AQS
description: AQS
date: 2019-06-08
categories:
  - "开发设计"
tags:
  - "Java"
  - "Concurrency"
  - "AQS"
draft: false
---
上一篇，我们了解了Java自带synchronized锁的一些相关知识，本篇文章将会聚焦在concurrent中，所有锁类型都会用到的工具类：AQS（AbstractQueuedSynchronizer），看看concurrent包中的ReentrantLock、CountDownLatch等工具都是如何实现的。<!--more-->

## 引入：ReentrantLock、NonretrantLock、ReentrantReadWriteLock
### 公平锁与非公平锁
为了方面后面介绍AQS，我们先看一下ReentrantLock是如何使用AQS来完成自己功能的。先了解AQS的使用，再了解AQS的原理，往往学习效率更高。

ReentrantLock是concurrent包里提供的可重入锁。它提供了公平模式和非公平模式：
1. 公平模式：根据申请顺序，线程会进入一个队列。队首的线程出队（释放锁）之后，下一个线程会获取到锁。这种调度方式的优点是，不会出现饥饿现象。缺点是吞吐低。
2. 非公平模式：线程获取锁的时候，会直接尝试获取锁（插队），获取不到才需要去排队。

![20200727144207](https://raw.githubusercontent.com/hujianxin/pico/master/img/20200727144207.png)

用户可以在构造函数中，指定是否使用公平模式：
```java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```
通过上面可知，公平模式和非公平模式的不同点在与一个叫sync的东西，那这个sync是个什么东西呢？通过下面的代码可以看出来，Sync就是一个抽象类，他继承了AQS，在AQS提供的基础设施的基础上，完成了ReentrantLock另外的基础设施。
```java
abstract static class Sync extends AbstractQueuedSynchronizer {
  // ...省略
}
```
Sync类的内容后面分析，目前的当务之急是分析一下FiirSync和NonfairSync的区别，循序渐进。
![20200727145954](https://raw.githubusercontent.com/hujianxin/pico/master/img/20200727145954.png)

通过上面的对比图可知，公平模式比非公平模式多了一个判断语句：hasQueuedPredecessors()
```java
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```
通过方法名和实现可以看出来，这是判断thread等待队列是否有后继节点。这也与公平锁的定义一致。

### 可重入锁与不可重入锁
之前有提到过可重入锁，但是没有说过它具体是什么含义。

可重入锁指的是，锁以**线程**为单位，同一个线程内，多次访问同一个加锁块，是不会被阻塞的。而不可重入锁，是以**调用**为单位，无论是不是在同一个线程，只要有另一个调用在访问这个加锁的块，当前调用就需要阻塞。

JDK中，只实现了可重入锁，没有实现不可重入锁，应该是因为不可重入锁适用范围窄，而且容易死锁，难以使用。

但是netty中实现了不可重入锁，不可重入锁的实现更简单，只是简单的继承了AQS，然后做了一点工作。我们先来对比一下这两者的差异：
![20200727151932](https://raw.githubusercontent.com/hujianxin/pico/master/img/20200727151932.png)
通过上图可以看出来，两者的差距在与一个叫state的东西，在可重入锁中，state是一个计数器，当前同一个线程访问这个代码片段时，state+1。而在不可重入锁中，state只是一个标志位。

### 独占锁与共享锁
JDK中提供了ReentrantReadWriteLock，也叫读写锁。

读写锁存在的意义是，如果一个系统，写请求少，读请求多，那么如果每次读都加锁的话，严重影响性能了。因为写请求少，本来只需要对写写、写读进行隔离就行，读读是不受影响的。所以出现了读写锁。

ReentrantReadWriteLock的构造函数如下：
```java
public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}
```
可以看到，内部有两个锁，一个是读锁，一个是写锁。而这两个锁，共用了同一个sync，共用sync，是因为读和写是排斥的，需要了解对方的信息。

那到底是如何使用同一个sync来实现读写分离的呢？关键还在与state这个字段，前面在将可重入、不可重入的时候已经说过了，在可重入锁中，state是一个计数器，不可重入锁中，state是一个标志位。在读写锁中，state的作用又变了，跟之前完全是两码事了！

state是一个int类型，32位。在读写锁中，高16位表示读状态（计数器），低16位表示写状态。将到这里，有些读者应该有所感觉了。

在ReentrantReadWriteLock中，实现了tryAcquire和tryAcquireShared两个方法，分别是用在写锁和读锁的获取上。

```java
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 && // 获取独占锁的个数，如果独占锁的个数是0，或者是本线程（可重入）在操作，那就可以继续获取锁了。
        getExclusiveOwnerThread() != current)
        return -1;
    int r = sharedCount(c);
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) { // 如果没有其他读写，则继续获取锁
        // (Note: if c != 0 and w == 0 then shared count != 0)
        if (w == 0 || current != getExclusiveOwnerThread())// 如果有其他读操作或者有其他线程在操作，则无法获取锁。
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

## AQS
经过分析ReentrantLock等的实现，我们已经对AQS是什么有了一个隐约的轮廓：
1. AQS里面有一个队列，用来存排队的线程，可以在这个队列中胜出的线程，就可以获取到锁。
2. AQS里面有一个state字段，这个字段在可重入锁中充当的是计数器角色，表示当前线程重复获取了多少次锁。在不可重入锁中充当的是标志位，表示是否有其他操作获取过锁。在读写锁中，高16位是读计数器，低16位是写计数器。
3. AQS里面有一个exclusiveOwnerThread字段，表示当前是哪个线程再获取独占锁。

除此之外，这里还要总结一下：
1. 队列的入队和出队都用了CAS工具unsafe.compareAndSwapObject，也就是自选锁。
2. state的设置也是使用CAS实现的。
3. 线程的阻塞和释放使用了LockSupport.park和LockSupport.unpark

## Condition
除了有一个同步队列之外，AQS里面还有一个ConditionObject类，这个是配合Condition使用的。ConditionObject里面也有一个队列。Condition用来两个线程之间通讯用的工具，具体使用方式如下：
```java
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();
...
lock.lock();
try {
  while ( 某条件A ) {
    condition.await();
  }
  // 条件A满足时应执行的代码
} finally {
   lock.unlock();
}
```

ConditionObject实现了java.util.concurrent.locks.Condition接口，表示自己是一个条件队列。作为内部类，它没有被static关键字修饰，表示它不能脱离外部的AQS类独立存在，必须与外部类AQS实例建立关联。这与我们实际使用可重入锁也是一致的。Condition属于某个锁。

Condition有两个核心方法:await(), signal()，这里提一嘴在Java原生体系中ReentrantLock对应的是synchronized，而Condition的await、signal对应的是Object的wait、notify。
```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

这里分析一下await步骤：
1. 相应中断
2. 将当前线程放入条件队列（ConditionObject维持的一个队列，与AQS中的同步队列不是一个）
3. 释放当前线程
4. 然后睡觉-被唤醒-睡觉-被唤醒-睡觉-被唤醒如此循环，直到发现当前线程已经处于同步队列了，说明自己已经获取到锁了。
5. 后面就是清理所状态

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
              (first = firstWaiter) != null);
}
```
下面分析signal步骤：
1. 从条件队列头部开始，找到第一个未取消等待的线程，并把线程由条件等待状态改为就绪状态
2. 条件队列取下来放入锁队列
3. 把这个线程用CAS操作设为等待锁状态

## CountDownLatch
CountDownLatch的代码比较简单，与其他动辄三四千行的大类来说，300多行代码可谓是小巧玲珑了，所以在大体了解了AQS的原理之后，了解CountDownLatch的原理也比较简单。
```java
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }

    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```
通过上面的代码可知CountDownLatch中，state也是一个倒计数的工具。

CountDownLatch有两个核心方法：await、countDown

在调用await之后：
1. sync的acquireSharedInterruptibly会被调用
2. 在acquireSharedInterruptibly中，把当前线程加入同步队列。
3. 进入循环：判断是否state已经被减到0,如果被减到0，则返回，否则睡觉
4. 这里是被谁唤醒的呢？

是被countdown唤醒的。具体不再分析了，可以参考后面的参考文章。

## 其他
至此，我们学习了Java中，如何使用AQS实现可重入锁、不可重入锁、公平锁、非公平锁、读写锁、CountDownLatch。

除此之外，AQS还是实现线程池Worker、Semaphore等的核心工具。关于线程池，后面会单独写一篇文章。

## 参考
1. [不可不说的Java“锁”事](https://tech.meituan.com/2018/11/15/java-lock.html)
2. [AQS条件队列Condition实现详解](http://byteliu.com/2018/09/01/AQS%E6%9D%A1%E4%BB%B6%E9%98%9F%E5%88%97Condition%E5%AE%9E%E7%8E%B0%E8%AF%A6%E8%A7%A3/)
3. [【JUC】JDK1.8源码分析之CountDownLatch（五）](https://www.cnblogs.com/leesf456/p/5406191.html)