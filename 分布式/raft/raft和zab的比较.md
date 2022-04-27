# Raft 和 Zab 的异同比较



---

**协议都需要在同一个轮次以内达到过半投票，ZAB 私用 electionEpoch（logicClock） 来表示，而 Raft 使用 term 表示。**



> 选举层面的异同：

Raft 的选举是先到先得，先到节点只要它 term 和 index 都大于等于自身就可以得到选票，**并且此轮选举期间，投票不可修改。**

ZAB 则会根据节点间的信息对比选择更优节点，即在投票之后可以改变并再次广播当前投票信息，最终选取出数据最完整的节点。

ZAB 的选票主要就是三类数据：logicClock，myid，zxid（ZAB 

Raft 的选举主要是两类数据：term，logIndex（Raft 无法设置优先级，相同的 logIndex 就靠先来先服务



> 日志复制的异同：

Raft 和 ZAB 都是通过日志复制来实现一致性的，可以说都是基于 日志复制状态机 来实现的日志一致性。

并且日志都必须是连续且递增的，Raft 根据 index 来标识日志顺序，ZAB 根据 zxid 标识日志顺序。

ZAB 中实现的是顺序一致性，最终一致性，在复制的过程中只要过半节点复制成功就可以提交，提交之后就对客户端可读，因此在复制期间也可能出现节点数据不一致的情况。

Raft 更偏向于强一致性，一定要所有节点都复制成功才会选择提交，当然这个可以根据自身需求选择，和 ZAB 一样过半提交也满足组中一致性。

写入来说，Raft 和 ZAB 都只能通过 Leader 节点写入。





## reference

[MIT 大佬的解释](https://www.zhihu.com/question/28242561/answer/40075530)

相同点：

1.  两种协议本质上都是维护一个replicated log，都是基于一个日志复制状态机
2. 都是用 timeout 来触发选举，都是在和主节点的心跳超时的时候触发，而
3. 都由 Leader 来完成写操作
4. 都采用心跳来检测存活性
5. Leader 都要求拥有最全的（最新的）数据

不同点：

1. Raft 是 Strong Leader，日志只会存在 Leader 到 Follower 的覆盖（在分区恢复之后，可能会有 Follower 上的 Log 被覆盖，而 Zab 会有一个同步阶段，Leader 会接受 Follower 上的事务，然后在集群内同步。
2. Zab 使用 epoch 





[比较详细的比较 Raft 和 ZAB](https://my.oschina.net/pingpangkuangmo/blog/782702)

