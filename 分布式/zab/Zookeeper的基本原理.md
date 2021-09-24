# ZAB 协议
---

[TOC]

---





## ZAB

ZAB（**Zookeeper Atomic Broadcast**  原子广播协议）包含**崩溃恢复和广播**两个阶段。

**Zab 协议的特性**：
 1）Zab 协议需要确保那些**已经在 Leader 服务器上提交（Commit）的事务最终被所有的服务器提交**。
 2）Zab 协议需要确保**丢弃那些只在 Leader 上被提出而没有被提交的事务**。

## ZAB 的节点角色

在广播阶段，ZAB 的节点分为以下几种角色：

- LEADER - 领导节点，负责所有的写入和 Proposal 的发起
- FOLLOWER - 跟随者节点，负责 Proposal 的写入和ACK，可以处理读请求
- OBSERVER - 类似 FOLLOWER 但是无投票权

<br>

LEADER 处于 LEADING 状态，FOLLOWER 处于 FOLLOWING 状态，而在崩溃恢复阶段还存在 LOOKING 状态，表示节点正处于投票状态。

> **LOOKING 状态下节点不对外提供读写服务。**

## ZAB 的读模型

ZAB 的所有读请求都必须由 LEADER 节点来处理，如果 FOLLOWER 节点接收到写请求也会转发给 LEADER 处理。

LEADER 节点接收到写请求后将其转化为一个 Proposal，并将其分发给所有的 FOLLOWER 节点（包括 OBSERVER 节点）。

只有在**过半**的 FOLLOWER 节点成功复制并回复 ACK 信息之后，会再次广播 COMMIT 信息表示写请求成功，最后返回写请求响应，对外可读。

> 该协议类似于 2PC，但是并不要求全部的节点都必须要写入成功。

<br>

ZAB 为每个写入的 Proposal 都提供一个 ZXID，ZXID 是一个64位长整形，高32位表示的是当前的 epoch，而低32位表示的是 epoch 之内的递增值。

根据递增的 ZXID，ZAB 可以实现顺序一致性。（我的理解就是写入的顺序是一致的，当然也实现了最终一致性



## ZAB 的选主流程







## 参考

[Zookeeper——一致性协议:Zab协议](https://www.jianshu.com/p/2bceacd60b8a)

[面试官问：ZooKeeper是强一致的吗？怎么实现的？](https://segmentfault.com/a/1190000039127403)