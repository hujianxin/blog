---
title: Raft的原理与应用
description: Raft的原理与应用
date: 2019-06-19
categories:
  - "开发设计"
tags:
  - "Raft"
draft: false
---

Raft算法在分布式领域可谓是大名鼎鼎，最近通过6.824课程，更细致的学习了一下Raft算法，在这里做一下总结工作。<!--more-->

## Raft算法在解决什么问题？
众所周知，Raft算法是一致性协议。于此同时呢，有很多人也挺说过两阶段提交，表示这也是一致性协议。难道他们是一种类型的算法吗？

其实不然，此一致性非彼一致性，这里用英文单词来区分可能更合适：Raft的一致性是Consensus，指的是一致性提交，2PC的一致性是Consistency，指的是事务的外部一致性。

也就是说，Raft是为了保证多个副本数据保持一样。而2PC是指，在事物提交中，保证原子性和外部一致性。完全是两个领域的概念。

## Raft应用场景举例

## Raft算法中的基本概念
1. Leader、Candidate、Follower，这三个概念是Raft集群中，机器的角色。一个机器可以是Leader，可能因为故障下线又上线之后又会变成Follower。任何写请求，都是由Leader来处理，然后同步到其他Follower。
2. Leader 选举，是一个过程，指的是系统中目前没有一个Leader角色，需要经过Leader选举选出唯一的Leader。
3. Term，任期。每换一个Leader，都会增加一个任期，任期是递增的数字。
4. Log，指的是需要同步的指令日志
5. State Machine，状态机，Log最后需要在状态机中得以实例化、现实化。

## 三种角色以及转换过程
![20200729130940](https://raw.githubusercontent.com/hujianxin/pico/master/img/20200729130940.png)

由上图可以看出来，三个角色是可以互相转换的。

1. Follower可以转换成Candidate，这个一般是发生在系统中开始、或者Leader下线之后，Follower发起新一轮选举，自己自动转为candidate角色。
2. Candidate可以转换成Candidate，一般是发生在选举超时（长时间没有选举出新的Leader来），重新选举。
3. Candidate转换成Leader，则是这个节点被选举成为唯一的Leader。
4. Candidate转换成Follower，则是发生在其他的某个Candidate选举成功，自身选举失败之后。Candidate重新变成Follower，做一名小弟。
5. Leader变成Follower，一般是发生在，某个Leader发生了网络隔离，系统中以为他下线了，就选出了新的王，他上线之后，发现自己的王朝已经结束了，自动变为小弟。

## 节点维护的状态
![20200729133352](https://raw.githubusercontent.com/hujianxin/pico/master/img/20200729133352.png)
下面三个是所有节点共有的，需要持久化的信息：
1. concurrentTerm，本届点所在的当前任期
2. votedFor，要投票给谁
3. log[]，本节点维护的log entries

下面是所有节点共有的，不需要持久化的信息：
1. commitIndex，在log entries里面，需要commit的最大的index
2. lastApplied，在log entires里面，已经apply到状态机的最大index

下面是只需要维护到leader节点，而且不需要持久化的信息：
1. 

## 两个重要的远程调用
Raft中，存在两个重要的RPC过程，也是构建Raft算法唯二必须的远程调用。
1. RequestVote RPC
2. AppendEntries RPC