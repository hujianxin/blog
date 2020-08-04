---
title: ZooKeeper选举
description: Zookeeper如何正确选主、如何正确帮其他系统选主。
date: 2020-03-01
categories:
  - "开发设计"
tags:
  - "Zookeeper"
  - "ZAB"
draft: false
---
![20200727110914](https://raw.githubusercontent.com/hujianxin/pico/master/img/20200727110914.png)
<!--more-->

## Zk的快速选举以及一致性保证
### 概念
1. myid，每个节点的唯一标识，是存在服务器上特定位置的一个文件，里面有唯一标识的数字
2. zxid，64位的数字，前32位表示Leader的epoch;后32位表示该epoch内的序号，也就是第几次原子写入。

### 服务器状态
1. LOOKING
2. FOLLOWING
3. LEADING
4. OBSERVING

### 选票结构
1. logicClock，表示这是该服务器发起的地几轮投票
2. state
3. self_id
4. self_zxid
5. vote_id
6. vote_zxid

### 投票过程
1. 先比较logicClock
2. 再比较zxid
3. 再比较myid

### Commit过的数据不丢失
快速选举与一致性保证都属于ZAB算法，zk通过快速选举中的zxid，保证在leader宕机之后，选出来的新leader包含所有commit过的数据。

### 未Commit过的消息对客户端不可见
新Leader会通过zxid判断，将Follower中未Commit，也未过半的消息删除。

## 如何利用ZK选主（分布式锁原理等同）
### 概念
zk作为选主工具，主要依赖与这两种概念：
1. Persist节点和Ephemeral节点分类，其中Ephemeral节点会在客户端与zk断开session连接时，自动删除。这是选主的必备特性，用于广播leader宕机事件。
2. Sequence节点和Non-Sequence节点分类，这两者用于决定使用公平选主、非公平选主。
    其中，Sequence是公平选主， 因为Leader宕机之后，会按顺序通知下一个节点自动成为Leader。而Non-Sequence是非公平选主，Leader宕机之后，会通知所有节点，进行竞争，重新竞选Leader。


![20200731151706](https://raw.githubusercontent.com/hujianxin/pico/master/img/20200731151706.png)