---
title: HBase Snapshot
description: Snapshot
date: 2020-05-18
categories:
  - "开发设计"
tags:
  - "HBase"
  - "Snapshot"
draft: false
---
![20200727110948](https://raw.githubusercontent.com/hujianxin/pico/master/img/20200727110948.png)

之前将过BulkLoad可以离线的写入HBase，并且不会出发HBase的写流程，从而减轻HBase的压力、提高写入速度。在读取HBase的时候，也有一个机制，可以直接绕过HBase的读取流程，直接读取HFile，也就是Snapshot机制。

Snapshot原本是为了快速备份HBase设计的。但是我们使用这个功能来快速扫表，效果惊艳。

## Snapshot技术基础原理

为什么Snapshot可以做到这么快呢？主要是因为HBase给表打Snapshot的时候，并没有真正的备份数据，生成了原始数据的一个指针。

能这样做得益与HBase落盘的数据都不会再修改，直到compact。

所以Snapshot的总体流程是这样的：
1. Memstore flush生成新文件
2. 对当前所有的Hfile建立引用指针。

## Snapshot具体流程
Snapshot是由客户端发起的，但是真正执行这一动作的是RegionServer，RegionServer非常多，这会导致Snapshot的事务问题，HBase通过两阶段提交来解决这个问题：

我们都知道两阶段提交的过程包括：
1. 协调者发起prepare，等待ack
2. 执行者准备环境，发送ack
3. 协调者收到所有的ack之后，发起commit

具体到Snapshot的实现中，两阶段提交主要是通过Zk为媒介进行的。

## Snapshot具体实现
前面将了Snapshot的流程，其中重要概念就是这个引用文件。

引用文件主要包含了region的元数据，HFile文件名等。

## Snapshot之后遇到Compaction
前面说到，snapshot文件只是真实HFile的引用文件，如果发生compaction之后，原文件不在了，snapshot如何保证不失效的呢？

HBase 的做法很简单，就是在compact之前，将原始数据转移到archive目录里面。