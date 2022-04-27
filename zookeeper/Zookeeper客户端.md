# Zookeeper

---

[TOC]

---



## Introduce

Zookeeper 客户端实例就是 Zookeeper 对象。

Zookeeper 对象中只有保存了 HostProvider （服务端的地址信息，以及 ClientCnxn 对象。

ClientCnxn 持有 WactherManager 以及各类消息队列和处理线程。



## 客户端初始化流程

客户端的初始化流程中最主要的还是创建 ClientCnxn，并且开启其中的执行线程。

创建的时候也可以设置默认的 Wacther。



## 客户端的网络处理

ClientCnxn 中包含两种逻辑线程：

1. SendThread - 数据和处理逻辑
2. EventThread - 事件处理线程（响应线程，包含了对 Watcher 的触发

同时包含两个消息队列：

1. pendingQueue - 已经发送等待响应的消息队列
2. outgoingQueue - 等待发送的消息队列



所有的网络请求都会被包装为 Packet，并且扔进消息队列（客户端网络请求都是异步的。

针对每一个请求，客户端都会生成一个 xid 并且添加到 Packet，在 Packet 发送之后就会被保存到 pendingQueue 等待对应 xid 的响应，并进行下一步处理。

xid 保存在 RequestHeader 里，一起保存的还有 type