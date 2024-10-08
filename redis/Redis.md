

# Redis

---

[TOC]



---

## Introduction

[Redis 知识结构脑图](https://www.processon.com/view/link/5f511977e0b34d6f59ddd751)



## 数据类型







## 持久化策略

Redis 的持久化策略分为 AOF（Append Only File）以及 RDB（Redis Database）。

AOF 就是追加写文件，该策略会记录所有的 Redis 所执行的更新命令，RDB 就是快照，会记录当前键空间所有的数据内容。

<br>

**AOF 是写后日志，区别于 MySQL 的 redolog，因此也会出现数据丢失的情况，并无法百分百保证不丢失。**、

RDB 包含了两种方式来生成当前数据快照 - SAVE 和 BGSAVE，SAVE 会在主线程执行快照生成逻辑，而 BGSAVE 会 fork 出子线程执行。

<br>两种方式各有优劣，AOF 随着时间会越来越大，而 RDB 数据丢失的情况更加严重（你没办法随时随刻都做快照，因此在 Redis4.0 之后又新增了两种方式混合的持久化策略。





## 网络模型

[Redis 和 I/O 多路复用](https://draveness.me/redis-io-multiplexing/)





## 分布式策略



Master - Slaver 模式解决的是读性能的横向扩展，但是只能保证最终一致性，并且因为单主结构的关系还存在单点故障的问题。

Sentinel 解决的是单点故障问题，采用第三方监控的方式，主节点宕机之后做故障转移。

Cluster 解决的是单机的内存限制，采用 Hash 槽的形式，将数据分布到 16384 个槽位，以 crc16 作为 Hash 函数，以 Gossip 交换当前的节点状态和槽位信息。





## 使用案例

### 分布式锁实现

Redis 的分布式锁实现非常简单并且高效，适用于大部分场景。

基本上就是基于 SET NX EX 命令，合理设置超时时间，对于如果时间不够而需要续期，续期问题可以在上锁之后添加守护线程定时执行命令。

<br>

集群导致的锁失效，因为异步扩散的时间差问题，上锁的主节点可能在扩散之前宕机，主备切换之后锁就失效了，会让另外的线程拿到锁。

此时，可以采用 Redis 官方提供的 RedLock 方案，会对过半的 Redis 实例分别上锁，全部成功才会成功。

<br>

Spring Inter 中对 RedisLock 也有实现，比较有特点的就是该实现会在本机创建 Lock 对象，并且上锁，由本地上锁减少 Redis 请求消耗。

但是对应的 Lock 需要



意向锁



线程池里的参数传递