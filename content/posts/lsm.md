---
title: LSM Tree
description: LSM Tree
date: 2020-06-18
categories:
  - "开发设计"
tags:
  - "LSM"
  - "HBase"
draft: false
---

![lsm_tree](https://raw.githubusercontent.com/hujianxin/pico/master/img/lsm_tree.png)
<!--more-->

LSM Tree是HBase使用的数据存储结构。主要原因是，HBase基于HDFS作为底层文件存储，而HDFS不支持随机写，只支持顺序写。而LSM Tree可以轻松应对这种使用场景。

随机写磁盘会导致大量的磁盘寻道，影响写入效率，LSM本身是顺序写入，在磁盘写入方面具有先天优势。另外，有些写多读少的应用场景也非常适合使用LSM Tree。

## HBase的LSM Tree实现
在HBase设计中，数据分为两部分：MemStore、DiskStore。

新写入的数据直接写入MemStore，但是为了防止数据丢失，会写一条WAL。当MemStore达到一定的阈值，将其设置只读，并新建一个新的MemStore用来写入，只读的MemStore会被异步的刷成一个DiskStore。

## HBase的compaction
LSM Tree实现的系统，因为是append only的设计，所以compact是必须的过程。HBase中，分为两种compact：minor compact，major compact

### minor compact
minor compact是选取部分小的、相邻的HFile，合并成一个大的HFile

### major compact
major compact，是将整个DiskSotre合并成一个HFile，合并过程中，会删除：过期数据、已经打了删除标记的数据、版本号超额的数据。

## 分层压缩：LevelDB的LSM Tree实现
LevelDB也是使用了LSM Tree的设计，但是他的compact过程与HBase不同，采用了分层压缩的设计。

![LSM_LEVEL](https://raw.githubusercontent.com/hujianxin/pico/master/img/LSM_LEVEL.png)

如上图所示：
1. Level0层，是由ImmutableMem Table dump出来的。所以Level0层不同文件可能会有重叠的。
2. Level n是由level n-1和level n层compact出来的。
3. 读数据时，顺序是内存->Level0->Level1->Level...，在查询Level0时，需要查询多个文件，查询其他层时，只需要查询特定文件。
