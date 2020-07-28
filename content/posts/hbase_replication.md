---
title: HBase 集群复制
description: HBase 集群复制
date: 2020-05-18
categories:
  - "开发设计"
tags:
  - "HBase"
draft: false
---

HBase本身支持集群复制，也就是说，制定一个备集群，HBase的RegionServer会异步（或同步）的将数据同步到备集群中。

这个功能看似简单，其实非常有用。首先，备集群本身是非常有用的，在某些系统中，需要跑一些离线任务半实时的统计，直接在线上HBase跑的话，容易占据太多资源，造成抖动。所以通过备集群来做这件事情是一个不错的选择。当然，我认为SnapShot应该是比这个更靠谱一点。

另外，这里的备集群并不一定是HBase集群，HBase提供了一个抽象，可以让用户自定义BaseReplicationEndpoint，可以将需要备份的数据自定义得写到某个地方，这就提供了无限的想象力。<!--more-->

本文主要基于小米内部的某个HBase版本讲的，其中串行复制功能，在社区版本的HBase2.x里面才出现，但是社区的同步复制，却不在我这个版本里面。

## 非串行复制
在HBase中，创建Peer的大致流程如下：
1. 客户端在ZooKeeper上创建一个Znode，创建之后客户端给用户返回OK
2. 每个RegionServer会收到一个Zk的Watch Event，收到之后创建一个ReplicationSource的线程，这个线程首先将HLog保存到walGroup-Queue里面
3. 每个WallGroup-Queue有一个ReplicationSourceWALReader线程，将HLog按entry方式读出来，写入到entryBatchQueue里面。
4. 每个EntryBatchQueue后面有一个 ReplicationSourceShipper线程，不断从Queue里面除去Log Entry，通过ReplicationEnpoint的replicate方法传输到远方。

## 串行复制
非串行复制过程中，假如Region发生了漂移，从RS0迁到RS1上，则RS0和RS1同时含有这个Region的HLog，那么Replication可能就发生错乱了。

为解决这个问题，首先要做到RS0上的所有数据同步完成之后，再同步RS1上的数据。

为了解决上面的问题小米的工程师实现了Barrier的概念，每次产生一个新的Region，都会将为这个Region能够读到的最大的SequenceID设为Berrier。同时，整个集群都有一个LastPushedSequenceId，当然Region有一个PendingSequenceId。如果PendingSequenceId小于LastPushedSequenceId，则不推送。
