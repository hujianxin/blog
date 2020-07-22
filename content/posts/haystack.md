---
title: Finding a needle in Haystack/ facebook’s photo storage
description: Finding a needle in Haystack/ facebook’s photo storage
date: 2020-05-18
categories:
  - "经典论文"
tags:
  - "Haystack"
  - "S3"
  - "经典论文"
draft: true
---

Facebook的Haystack系统比较有名气。虽然目前它已经被F4所替代，但其简洁的设计还是值得学习和思考。而且，学习F4之前，对Haystack进行细致的了解也是必须的过程。

Haystack并不是一个标准的对象存储系统，它仅仅围绕这Facebook公司对图片存储的需求设计和实现。
图片存储和普适对象存储的区别：
1. 图片存储对延迟和吞吐要求更为严格，虽然说普通的对象存储系统也需要做到比较好的延迟，但是很多应用场景是不需要考虑延迟的，所以专业的图片存储延迟要求肯定要高一些。
2. 数据删除少。普通的对象存储系统，用户删除数据会多一些，而且用户可以配置TTL，配置TTL之后，系统会大量删除数据。

不过总体来说，Haystack具备了对象存储的基本功能要求。


论文中提到了Haystack在下面几个方面表现卓越：
1. 高吞吐、低延迟，这个主要的易于blob meta都存放到内存中，有点类似与hdfs的namenode。
2. 容错性，haystack数据三备份，分布在不同的机房中。这个比基于hdfs的对象存储要高一些，基于hdfs的对象存储只能做到跨机器备份，而没法做到跨机房备份，小大小闹还可以。
3. 节省费用，可能对比之前的NAS方案，Haystack是节省的，但是对比后面的F4以及其他公司的对象存储，Haystack在费用方面肯定做的还不到位。
4. 简单

Haystack由三大部分组成：


## 从Haystack中学到的经验
1. 底层File中直接存储Blob的删除信息，方便GC小规模的优化
2. IndexFile设计