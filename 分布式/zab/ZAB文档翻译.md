# ZAB 文档

> [文档地址](https://zookeeper.apache.org/doc/r3.5.0-alpha/zookeeperInternals.html#ch_Introduction)

---

[TOC]

---

## 序论

该文档包含 Zookeeper 内部工作的信息，包含以下内容：

1. 原子广播
2. 日志记录



## 原子广播

Zookeeper 的核心就是一个原子的消息系统，用来保持所有服务的同步（同步的内容如下：

- 可靠性传递

  如果消息 m，被一个服务传递，则它最终被所有的服务传递。（传递应该是过半之后的，类似于 Raft 的提交

- 总顺序

  如果在一个服务中消息 a 在消息 b 之前传递，那么在所有服务中 a 都是在 b 之前被传递的。



> 缺了一大段。

