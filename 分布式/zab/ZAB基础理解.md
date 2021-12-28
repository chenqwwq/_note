# ZAB 协议
---

[TOC]

---





## ZAB

ZAB（**Zookeeper Atomic Broadcast**  原子广播协议）包含**崩溃恢复和广播**两个子协议（或者说模式）。

广播协议保证数据的只从单节点写入而后广播到集群内所有节点，进而保证数据的顺序一致性（以及最终一致性）以及持久性（数据备份），而奔溃恢复，则是在 Leader 宕机等情况下完成新 Leader 的选举。

<br>

## ZAB 的节点角色

在广播阶段，ZAB 的节点分为以下几种角色：

- LEADER - 领导节点，负责所有的写入和 Proposal 的发起
- FOLLOWER - 跟随者节点，负责 Proposal 的写入和 ACK，可以处理读请求
- OBSERVER - 类似 FOLLOWER 但是无投票权，也不需要回复 ACK，用于增强读能力

<br>

LEADER 处于 LEADING 状态，FOLLOWER 处于 FOLLOWING 状态，而在崩溃恢复（选主）阶段还存在 LOOKING 状态，表示节点正处于投票状态。

> **LOOKING 状态下节点不对外提供读写服务。**

<br>

## ZAB 的广播（读写）模型

ZAB 的所有**写请求都必须由 LEADER 节点来处理**，如果 FOLLOWER 节点接收到写请求也会转发给 LEADER 处理。

**LEADER 节点接收到写请求后会带上 ZXID，并转化为一个 Proposal，然后将其分发给所有的 FOLLOWER 节点（包括 OBSERVER 节点）。**

ZXID 的由 Leader 生成保证其递增（应该也是连续的），这样在整个系统中可以保证事务处理的顺序性（Zookeeper 并不提供强一致性保证，只保证顺序一致性。

> ZXID 的生成：
>
> - 高 32 位是： epoch（纪元），代表着周期，每当选举产生一个新的 Leader 服务器时就会取出其本地日志中最大事务的 ZXID ，解析出 epoch（纪元）值操作加 1作为新的 epoch（相当于一个集群的逻辑时钟。
> - 低 32 位是： counter（计数器），它是一个简单的单调递增的计数器，针对客户端的每个事务请求都会进行加 1 操作；

只有在**过半**的 FOLLOWER 节点成功复制并回复 ACK 信息之后，会再次广播 COMMIT 信息表示写请求成功，最后返回写请求响应，对外可读。

> 该协议类似于 2PC，但是并不要求全部的节点都必须要写入成功。

<br>

## ZAB 的奔溃恢复流程

ZAB 的奔溃恢复包括**选主和同步**问题。

选主在以下两种场景会触发：

1. 集群启动初期，首次的 Leader 选举，此时所有节点都为 LOOKING 状态。
2. 出现网络的中断或者 Leader 节点的宕机奔溃等和 Leader 节点连接断开的情况

**在 Leader 节点和超过一半的节点建立连接并且完成数据同步之后退出奔溃恢复的状态，转而进入消息广播阶段。**

奔溃恢复阶段的保证：
 1）确保**已经在 Leader 服务器上提交（Commit）的事务最终被所有的服务器提交**。
 2）确保**丢弃那些只在 Leader 上被提出而没有被提交的事务**。

<br>

奔溃恢复的模式下，节点先进入 LOOKING 状态下，并且先给自己投票，将票以 <MyID,ZXID> 的形式计入投票箱，之后广播自己的投票信息。

各节点收到投票信息之后和本地按照如下顺序进行对比：

1. EpochId 对比
2. ZXID 进行对比
3. Sid（启动时配置的 MyID） 对比

所有对比都是大者胜出，整理投票结构之后继续进行广播。

最后判断是否有半数节点选择出 Leader，选出 Leader 后进入数据同步阶段，需要半数以上节点完成同步才能退出奔溃回复状态。



## ZAB 的同步流程

该流程是和 Raft 不同地方，在每次选举结束之后 Leader 会发起一次同步流程。

因为过半提交原则，且投票的判断依据也是 epoch 和 zxid，所以称为 Leader 的节点肯定保存有较为完整的记录。

> 只有是被提交的 Proposal，新 Leader 肯定有，否则不可能得到提交该 Proposal 的那些节点的 Vote。

所以在同步的时候，主要做以下两件事情：

1. 补充 Follower 的 Proposal（ Follower 的日志记录小于当前 Leader
2. 截断 Follower 的 Proposal（ Follower 保留有上次 Leader 中未提交的 Proposal

在完成同步阶段之后，Leader 会发起一个 NEW_LEADER 的 Proposal，待 quorum 的 Follower 接受该 Proposal。

在对 NEW_LEADER 提交之前，Leader 不会接收任何请求。



## ZAB 协议总结

下文节选自参考 3：

```
So there you go. Why does it work? Specifically, why does is set of proposals believed by a new leader always contain any proposal that has actually been committed? First, all proposals have a unique zxid, so unlike other protocols, we never have to worry about two different values being proposed for the same zxid; followers (a leader is also a follower) see and record proposals in order; proposals are committed in order; there is only one active leader at a time since followers only follow a single leader at a time; a new leader has seen all committed proposals from the previous epoch since it has seen the highest zxid from a quorum of servers; any uncommited proposals from a previous epoch seen by a new leader will be committed by that leader before it becomes active.
```

1. 所有的 Proposal 都有一个唯一的 ZXID，所以和别的协议不同，ZAB 从来不需要关系不同的 Proposal 拥有同一个 ZXID。
2. 所有的节点都是以相同顺序处理 Proposal，并且以相同顺序提交 Proposal。
3. 同一时间只有一个活跃的 Leader，所有的 Follower 同一时间只跟随一个 Leader。
4. 一个 Leader 会提交之前的 epoch 的 Proposal，因为他的 zxid 肯定是最大的，超过 quorum 的节点 zxid 不如新的 Leader 节点。
5. 任何一个未提交的之前 opoch 的 Proposal，将会在 Leader 激活前被提交。（这点和 Raft 不一样，Raft 的日志提交是通过后续的日志来完成的，ZAB 使用一个同步阶段来推断当前未提交的 Proposal 是否应该被提交。



## 参考

1. [1Zookeeper——一致性协议:Zab协议](https://www.jianshu.com/p/2bceacd60b8a)

2. [面试官问：ZooKeeper是强一致的吗？怎么实现的？](https://segmentfault.com/a/1190000039127403)

3. [ZooKeeper Internals](https://zookeeper.apache.org/doc/r3.5.0-alpha/zookeeperInternals.html)