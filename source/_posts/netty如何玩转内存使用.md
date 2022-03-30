---
title: netty如何玩转内存使用
date: 2022-03-14 00:01:00
tags:
  - Netty
categories: Netty
---

### Netty如何玩转内存使用

1、减少对象本身大小

2、对分配内存进行预估

3、Zero-Copy 零复制

4、堆外内存

优点：
1、破除对空间限制，减轻GC压力

2、避免复制

缺点：

1、创建速度稍慢

2、堆外内存受操作系统管理



5、内存池

为什么引入对象池:

1、创建对象开销大
2、对象高频率创建且复用
3、支持并发又能保护系统
4、维持、共享有限的资源

如何实现对象池？

1、开源实现：Apache Commons Pool
2、Netty轻量级对象池实现io.netty.util.Recycler

### 源码解读Netty内存使用

1、内存池/非内存池的默认选择及切换方式
io.netty.channel.DefaultChannelConfig#allocator

2、内存池的实现
io.netty.buffer.PooledDireByteBuf
3、堆外内存/堆内内存的默认选择及切换方式
4、堆外内存的分配本质




