# Java 的 IO 模块



---

[TOC]

---



## 概述

[IO（Input/Output）的概念](../../系统/Linux的IO模型.md)

<br>

Java 的 IO 可以分为 BIO，NIO，AIO 三类（以网络 IO 介绍，文件 IO 阻塞相对较短。

<br>

<br>

## BIO

BIO 是 Java 中最基础的 IO 模式，对应来说是 Linux 五种 IO 模型中的**同步阻塞 IO**。



## NIO

NIO 在 Java 中主要包含三类对象：

- **Channel** - 对 fd 的抽象
- **Buffer** - IO 的缓冲池
- **Selector** - 多路复用器



<br>

NIO 不再和 BIO 一样直接操作 Socket 对象，而是包装为了 Channel，并且数据的 IO 也不再进过流，而是进一步封装的 Buffer。

> 相对于流来说，Buffer 的效率明显更高。

对于文件 IO，Java 提供了 FileChannel，其中可以定义了几种零拷贝的形式。

对于网络 IO（Java 中的网络 IO 基本都是传输层以上的抽象，所以对于 UDP，提供了 DatagrqamChannel，而对于 TCP 则是另外提供了 SocketChannel 以及 ServerSocketChannel。

ServerSocketChannel 是服务端 Channel，绑定服务端某接口并监听 accpet 事件（而 SocketChannel 不包含该事件。

Buffer 一定程度上就是一整块的可写区域或者说数据，以 Buffer 为单位而不是以字节（流就是有字节为单位操作）操作，更加的方便。

对于 Buffer，Java 中根据不同的基础类型提供了不同的 Buffer：

- ByteBuffer
- CharBuffer
- DoubleBuffer
- FloatBuffer

...

但是 Buffer 也具有一定程度上的复杂度（ Netty 对此又进行了一波优化，Buffer 主要包含了一下三个属性：

- capacity - 表示 Buffer 的容量
- position - 下一个读写的位置
- limit - 剩余可读/可写的大小

读写使用 flip() 方法切换。

<br>

Selector 是对 IO 多路复用的支持，对于不同的底层系统会提供不同的实现。

可以使用 **-Djava.nio.channels.spi.SelectorProvider=sun.nio.ch.EPollSelectorProvider** 指定具体的实现。

包含的就是常说的 poll，epoll，select，kqueue 等底层系统调用。







## NIO 和 BIO 的对比

1. BIO 是基于流（Stream）的操作，而

