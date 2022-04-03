# ZAB 协议
---

[TOC]

---

## 概述

ZAB（**Zookeeper Atomic Broadcast**  原子广播协议）包含**崩溃恢复和广播**两个子协议（或者说模式）。

ZAB 满足 CP 原则（不满足 A 高可用原则。

**Zookeeper 并不提供强一致性保证，只保证顺序一致性。**

<br>

## ZAB 的节点角色

ZAB 的节点分为以下几种角色：

- Leader - 领导节点，负责所有的写操作以及部分读操作
- Follower - 跟随节点，复制 Leader 节点数据，只用于处理读请求
- Observer - 类似 Follower 但是无投票权，用于横向扩展读能力

<br>

Leader 处于 Leading 状态，Follower 处于 Following 状态，在崩溃恢复（选主）阶段还存在 Looking 状态，表示节点正处于投票状态。

**Looking 状态下节点不对外提供读写服务。**

<br>

ZAB 属于强领导型协议，在每一个 epoch 中都会存在（仅存在）一个 Leader 节点。

<br>

## 广播（读写模型

ZAB 的所有**写请求都必须由 Leader 节点来处理**，如果 Follower 节点接收到写请求也会转发给 Leader 处理。

**Leader 节点接收到写请求后会转化为一个 Proposal，并带上 Zxid，然后将其分发给所有的 Follower 节点（包括 Observer 节点）。**

Leader 节点会为每个 Follower 准备一个队列，每个 Proposal 会按照 FIFO 的顺序发送给 Follower（保证全局的写入顺序。

Follower 接收到 Proposal 并本地写入之后会回复一个 ACK，只有在过半的 Follower 回复 ACK 之后 Leader 才会向客户端返回写入成功。

（因此 Zookeeper 并不能算是高性能写入，广播的过程强制要求过半接收。

<br>

ZAB 的事务模型基于 2PC 实现，Leader 作为事务协调器，发起事务并且等待从属事务提交。

<br>

### Zxid

Zxid 的由 Leader 生成保证其递增（应该也是连续的），这样在整个系统中可以保证事务处理的顺序性。

高 32 位是  epoch（纪元），代表着周期，每当选举产生一个新的 Leader 服务器时就会取出其本地日志中最大事务的 Zxid ，解析出 epoch（纪元）值操作加 1作为新的 epoch（相当于一个集群的逻辑时钟。

低 32 位是 counter（计数器），它是一个简单的单调递增的计数器，针对客户端的每个事务请求都会进行加 1 操作；

<br>



### Reference

- [漫谈ZAB协议——消息广播](https://www.modb.pro/db/73845)



## 奔溃恢复

ZAB 的奔溃恢复包括**选主和同步**两个流程。

### 选举流程

<br>

选主在以下两种场景会触发：

1. 集群启动初期，首次的 Leader 选举，此时所有节点都为 Looking 状态，并且 Zxid
2. 出现网络的中断或者 Leader 节点的宕机奔溃等和 Leader 节点连接断开的情况

**在 Leader 节点和超过一半的节点建立连接并且完成数据同步之后退出奔溃恢复的状态，转而进入消息广播阶段。**

奔溃恢复阶段的保证：
 1）确保**已经在 Leader 服务器上提交（Commit）的事务最终被所有的服务器提交**。
 2）确保**丢弃那些只在 Leader 上被提出而没有被提交的事务**。

<br>

奔溃恢复的模式下，节点先进入 Looking 状态下，并且先给自己投票，将票以 <MyID,Zxid> 的形式计入投票箱，之后广播自己的投票信息。

各节点收到投票信息之后和本地按照如下顺序进行对比：

1. EpochId 对比
2. Zxid 进行对比
3. Sid（启动时配置的 MyID） 对比

所有对比都是大者胜出，整理投票结构之后继续进行广播。

最后判断是否有半数节点选择出 Leader，选出 Leader 后进入数据同步阶段，需要半数以上节点完成同步才能退出奔溃回复状态。

#### Reference

- [漫谈ZAB协议——选举](https://www.modb.pro/db/73847)



### 同步流程

该流程是和 Raft 不同地方，在每次选举结束之后 Leader 会发起一次同步流程。

因为过半提交原则，且投票的判断依据也是 epoch 和 Zxid，所以称为 Leader 的节点肯定保存有较为完整的记录。

> 只有是被提交的 Proposal，新 Leader 肯定有，否则不可能得到提交该 Proposal 的那些节点的 Vote。

所以在同步的时候，主要做以下两件事情：

1. 补充 Follower 的 Proposal（ Follower 的日志记录小于当前 Leader
2. 截断 Follower 的 Proposal（ Follower 保留有上次 Leader 中未提交的 Proposal

在完成同步阶段之后，Leader 会发起一个 NEW_Leader 的 Proposal，待 quorum 的 Follower 接受该 Proposal。

在对 NEW_Leader 提交之前，Leader 不会接收任何请求。



#### Reference

- [漫谈ZAB协议——数据恢复与同步](https://www.modb.pro/db/73847)



### 异常流程处理



## ZAB 协议总结

下文节选自参考 3：

```
So there you go. Why does it work? Specifically, why does is set of proposals believed by a new Leader always contain any proposal that has actually been committed? First, all proposals have a unique Zxid, so unlike other protocols, we never have to worry about two different values being proposed for the same Zxid; Followers (a Leader is also a Follower) see and record proposals in order; proposals are committed in order; there is only one active Leader at a time since Followers only follow a single Leader at a time; a new Leader has seen all committed proposals from the previous epoch since it has seen the highest Zxid from a quorum of servers; any uncommited proposals from a previous epoch seen by a new Leader will be committed by that Leader before it becomes active.
```

1. 所有的 Proposal 都有一个唯一的 Zxid，所以和别的协议不同，ZAB 从来不需要关系不同的 Proposal 拥有同一个 Zxid。
2. 所有的节点都是以相同顺序处理 Proposal，并且以相同顺序提交 Proposal。
3. 同一时间只有一个活跃的 Leader，所有的 Follower 同一时间只跟随一个 Leader。
4. 一个 Leader 会提交之前的 epoch 的 Proposal，因为他的 Zxid 肯定是最大的，超过 quorum 的节点 Zxid 不如新的 Leader 节点。
5. 任何一个未提交的之前 opoch 的 Proposal，将会在 Leader 激活前被提交。（这点和 Raft 不一样，Raft 的日志提交是通过后续的日志来完成的，ZAB 使用一个同步阶段来推断当前未提交的 Proposal 是否应该被提交。



## 参考

1. [Zookeeper——一致性协议:Zab协议](https://www.jianshu.com/p/2bceacd60b8a)

2. [面试官问：ZooKeeper是强一致的吗？怎么实现的？](https://segmentfault.com/a/1190000039127403)

3. [ZooKeeper Internals](https://zookeeper.apache.org/doc/r3.5.0-alpha/zookeeperInternals.html)