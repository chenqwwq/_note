# Zookeeper





## Introduce

```
ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. 
```

Zookeeper 是一个中心化服务，用来作为配置中心，命名服务，提供分布式同步以及集群服务。

Zookeeper 的特性：

1. 顺序一致性（Sequential Consistency，来自客户端的请求在全局范围内保证以相同的顺序执行
2. 原子性（Atomicity，请求要不成功要不失败，不会有中间状态
3. 统一视图（Single System Image，客户端链接哪个服务器都可以看到相同的服务视图（？？不是很理解，最终一致性为什么会有相同视图
4. 可靠性（Reliability，数据更新成功则不会丢失。
5. 及时性（Timeliness，确实视图在特定时间范围内是最新的（？？？这个特定范围有意思了



## 数据结构

Zookeeper 以类似文件系统的形式保存所有的数据（数据存在分明的层次空间），间隔使用 / 表示（根目录也是 / 。

每个节点都被称为 ZNode，每个 ZNode 表示一层目录，也表示一份数据（建议存储不超过 1MB。

ZNode 有以下几种类型：

1. 持久节点（Persistent），节点长久保存
2. 临时节点（ephemeral），在连接断开后，该连接创建的临时节点就会被删除
3. 顺序节点（Sequential），创建相同名字的节点，节点名称后面自动递增数字



节点状态：

| 状态属性       | 状态说明                                       |
| -------------- | ---------------------------------------------- |
| czxid          | create zxid（节点被创建时的 zxid               |
| mzxid          | modify zxid（节点创建时的 zxid                 |
| pzxid          | 子节点最后一次修改的事务Id                     |
| ctime          | create time（节点创建时间                      |
| mtime          | modify time（接待你更新时间                    |
| version        | 当前节点的版本号（ZK 的更新使用 version 做 cas |
| cversion       | child version （子节点的版本号                 |
| aversion       | acl version（节点权限的版本号                  |
| ephemeralOwner | 临时节点拥有者 Id（如果是持久节点，该值为0     |
| dataLength     | 数据内容长度（就是二进制数组的长度             |
| numChildren    | 当前节点的子节点个数                           |





## ZAB 协议



## 源码相关

ClientCnxn 包含了所有客户端相关逻辑，以及一些命令的处理。

ClientCnxn 中包含 SendThread 和 EventThread 两个线程，分别用于处理待发送 Packet 和相关事件处理。

ClientCnxn 中使用 outgoingQueue 保存需要发送的数据，SendThread 直接轮询该队列。





## Reference

- [huzb - Zookeeper 学习笔记](https://huzb.me/2019/07/14/zookeeper%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/)
