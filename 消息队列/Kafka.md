# Kafka



## Kafka 的 Producer 结构

生产者的消息经过发送方法后，会经由以下几个组件发送：

1. 拦截器
2. 序列化器
3. 分区器
4. 累加器



## Kafka 中的 Election

1. Controller 的选举
2. Leader Partition 的选举



> Leader Election算法非常多，比如Zookeeper的[Zab](http://web.stanford.edu/class/cs347/reading/zab.pdf), [Raft](https://ramcloud.stanford.edu/wiki/download/attachments/11370504/raft.pdf)和[Viewstamped Replication](http://pmg.csail.mit.edu/papers/vr-revisited.pdf)。而Kafka所使用的Leader Election算法更像微软的[PacificA](http://research.microsoft.com/apps/pubs/default.aspx?id=66814)算法。

## Kafka 的语义保证

- At Least Once
- At Most Once
- Exactly Once





## Kafka 的幂等性保证

幂等性保证是指一条消息多次发送在 Partition 中只保存一条，从而在 Broker 端保证消息不会重复。

另外借由幂等性，**也能进一步保证消息的有序性**，虽然消息在 Partition 中本身就是有序的。

开启幂等性之后，Producer 发出的消息会**带有 PID（Producer ID）和 Seq（Sequence Number）两个属性**，保证在单个生产者一条消息多次发送的幂等性。

> **通过 enable.idempotence = true 开启幂等性保证。**
>
> PID 和 Seq 都由 Kafka 客户端提供，对用户完全透明。

如果中间出现缺漏，则直接抛出异常。

> 该种实现可以参考 TCP，首先生成 ISN，在后续发送过程中递增，再根据端口来区分报文归属，以窗口的形式保证消息的有序并且不重复。



## Kafka 的事务性保证

Kafka 的事务性保证，保证的最终目标是**多个读写操作的原子性**。

Kafka Client 引入了一个 Transaction Id，在 Client 启动时由用户指定。

Kafka Broker 引入了一个 Transaction Coordinator （TC）端，负责维护整个集群的 Transaction Log，该 Log 就近保存在 Topic 中，借由 Topic 实现日志的持久性。

另外，Kafka 的事务需要开启幂等性做前提。

相关流程如下：

![Kakfa事务流程](assets/Kafka%E4%BA%8B%E5%8A%A1%E6%B5%81%E7%A8%8B.png)

1. 找到 Transaction Coordinator，通过向集群中的任意节点发送 FindCoordinate 请求。
2. 获取 PID，通过向 TC 发送 InitPidRequest 的请求，TC 同时保存了 Transaction ID 和 PID 的对应关系，并且对应的 epoch 增加，如果有还会继续之前未完成的操作。

> epoch 的作用是 PID 的有效性，Broker 收到 PID 相同但是 epoch 不同的消息时，会直接丢弃掉较小的。

1. 



### 相关参考

[Kafka 设计解析（八）：Kafka 事务机制与 Exactly Once 语义实现原理](https://www.infoq.cn/article/kafka-analysis-part-8)





## Kafka Comsumer Group

