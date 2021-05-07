# Netty 中写数据的逻辑

> 以 4.1.53.Final-SNAPSHOT 中 Nio 相关封装为例。



---

[TOC]

---

## 概述

Netty 中的写数据分为两步：

1. Write 
2. Flush



Write 只是把数据添加到缓冲区，Flush 才是真实的发送方法。





## 写缓冲区

写缓冲区的具体实现还是在 AbstractChannel 里。

1. 过滤消息，例如 Nio 只允许 ByteBuf 进行写入
2. 推断写入消息的大小
3. 将 ByteBuf 添加到 tailEntry 后面，添加到链表
4. 增加等待发送的字节计数
5. 到达高水位设置 Channel 不可写状态





### Netty 的缓冲区

这里的缓冲区是指



