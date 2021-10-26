# Netty 的内存管理

---

[TOC]

---



## 概述

Netty 基于 jemalloc 实现了自身的内存管理。

早些时候的 jemalloc3 基于伙伴算法（基于完全二叉树）以及 Slab 算法实现，但是还会存在内存碎片的问题（或者说内存浪费。

因为基于完全二叉树实现的伙伴算法，对于 1025b 的内存申请会直接扩大到 2048 的 subpage，中间可能出现 1023b 的内存浪费。

而最新的版本中，Netty 升级为 jemalloc4 的分配方式可以减少这种内存的浪费。





## 笔记

最上层的对象是 ByteBufAllocator，具体的实现有PooledByteBufAllocator 和 UnpooledByteBufAllocator。

PooledByteBufAllocator 会管理所有申请的内存，默认会创建 PoolArena，直接内存（DirectArena）和堆内存（HeapArena），默认各32个。

使用 PooledByteBufAllocator 分配内存的时候，默认使用直接内存。

> JVM 的堆属于逻辑内存，物理地址可能随着 GC 的变化而变化，但是 C 中一些处理函数需要一个固定的内存地址，所以如果使用堆内存的时候，会默认先复制到堆外内存在进行具体的系统调用，因此相对于使用直接内存来说，使用堆内存多一次 IO 复制。

分配的时候会先获取本地线程的缓存，Netty 中使用 FastThreadLocal 为每个线程创建内存的缓存，使用的 PoolThreadLocalCache（继承于 PoolThreadCache）封装保存。

创建 PoolThreadLocalCache 的时候会从 Allocator 中选择一个最少使用的 Arena（直接内存和堆内存都会选择一个）。

从缓存中取出 Arena，然后继续分配。

> PoolThreadLocalCache 只是标记 Arena 的归属，其实最终使用的还是 Allocator 的 Arena。





## 参考

- [Netty源码解析 -- PoolChunk实现原理](https://www.cnblogs.com/binecy/p/14092637.html)

- [Netty源码解析 -- PoolChunk实现原理(jemalloc 3的算法)](https://www.cnblogs.com/binecy/p/14191601.html)