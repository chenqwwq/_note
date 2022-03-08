# Reactor 模式

## 概述

Reactor 模式是基于 I/O 多路复用技术实现的线程模型。

Reactor 源于 Doug Lea 的经典论文 Scalable I/O In Java。





## 单线程模型

单线程模型是最基础的 I/O 多路复用的实现方式，包括所有的 I/O 操作（accept、read、write）等，都是由单一线程完成的。

（Selector 中的 read/write 等事件，甚至包括对数据的业务逻辑都是由 Selector 的持有线程处理。

单线程 Reactor 模型中只有单个线程就是 Reactor 线程。

<br>

单线程处理的连接数非常有限，读写操作很快就会打满，无法充分发挥多核优势。

<br>

使用单线程 Reactor 模型的实现有：

1. Redis

## 多线程模型

多线程 Reactor 模型在单线程的基础上增加了一个线程池，业务逻辑都转交由线程池完成。

（但此时 Read/Write 等 I/O 事件仍然是由 Reactor 线程完成的。



## 主从多线程模型



主从多线程模型将 I/O 事件进一步细分，由 mainReactor 事件处理 accpet 事件，然后将创建的 Socket 对象注册到 subReactor，而 subReactor  只用来处理 read/write 事件。

mainReactor 持有 Selector 并监听服务端 Socket，而 subReactor 持有的 Selector 则是监听客户端 Socket。

另外的对网络数据的业务逻辑全部在线程池中处理，有 subReactor 线程调度。

![4.png](../系统/io/assets/CgqCHl-ZNEGAMU-zAAEsYdWKArA085.png)





## TOMCAT 的线程模型

TOMCAT 的线程模型中，Acceptor 负责处理所有的 accpet 事件，而 Poller 处理读写事件，而 Handlers 对应的就是线程池用于处理业务逻辑。

TOMCAT 的线程模型基本基于主从多线程模型，但是又有不同，主从多线程模型中会由 subReactor 读取所有的请求，而 Poller 线程只会读取 Request Line 以及 Headers 信息，Body 的读取在另外的 BlockPoller。



## Reference

- [Reactor 模式](https://zzcoder.cn/2020/08/23/Reactor%E6%A8%A1%E5%BC%8F/#more)

