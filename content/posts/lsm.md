---
title: LSM
description: LSM Tree
date: 2020-05-18
categories:
  - "开发设计"
tags:
  - "LSM"
  - "HBase"
draft: true
---

LSM Tree是HBase使用的数据存储结构。主要原因是，HBase基于HDFS作为底层文件存储，而HDFS不支持随机写，只支持顺序写。而LSM Tree可以轻松应对这种使用场景。

随机写磁盘会导致大量的磁盘寻道，影响写入效率，LSM本身是顺序写入，在磁盘写入方面具有先天优势。另外，有些写多读少的应用场景也非常适合使用LSM Tree。

## 大小分级：HBase的LSM Tree实现

## 分层压缩：LevelDB的LSM Tree实现

## LSM Tree的提速手段
### LRU Cache
### BloomFilter
### 使用二分搜索加速KeyValue的查找
### 如何提高WAL的写入性能