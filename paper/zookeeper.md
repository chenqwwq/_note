# Zookeeper 论文





## 3. 原子广播协议（Atomic broadcast protocol）

在 Zab 中，每个节点包含三种可能的状态：follower，leading，election，无论节点处于 follower 还是 leader 状态，都将顺序执行以下三个 Zab 的流程，（1）discovery （2）synchronization 以及（3）广播，