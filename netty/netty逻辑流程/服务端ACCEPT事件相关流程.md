# 服务端 OP_ACCEPT 事件相关流程

> 客户端连接之后的 Channel 对象创建和初始化过程。

[TOC]

---

## 概述

**Netty 在创建服务端 Bootstrap（ServerBootstrap）并绑定端口的时候，就会创建服务端的 Channel（ServerSocketChannel ）绑定上 EventLoop，并监听 Channel 上的 OP_ACCEPT 事件。**

> ServerBootstrap 在初始化 ServerSocketChannel 的时候还会添加一个特殊的 ChannelInboundHandler，也就是 **ServerBootstrapAcceptor**，它是处理 accpet 事件的主要对象。

<br>

通过 accpet 方法返回的是 JDK 原生的 Channel，因此 Netty 还需要对其进行封装（封装为  Netty 自定义的 Channel 类型）并绑定到某个 EventLoop 中。

![](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122230741489.png)

以上就是 NioEventLoop#run 方法也就是轮询方法中的一段，这里就包含了 SelectionKey.OP_ACCEPT 事件的处理。

**创建 SocketChannel 的起点就在 unsafe.read 方法。**



## 一、获取并包装原生的 Channel

Netty 中的底层读写基本是由 Unsafe 类完成的，Unsafe 的实现分为以下两种：

- **NioMessageUnsafe  -  以 Object 列表作为缓存读取**
- **NioByteUnsafe  -  以 ByteBuf 作为缓存读取**

Netty 中 accpet 和 read 在同一个方法签名中实现，通过不同  unsafe 实现不同的逻辑。

以下是 AbstractNioMessageChannel$NioMessageUnsafe#read 方法的部分源码:

![](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122231328166.png)

首先是通过 doReadMessages 方法，读取所有数据到 readBuf 中，readBuf是一个  List<Object> 对象，作为缓存，如下：

![image-20201122231444241](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122231444241.png)

AbstractNioMessageChannel#doReadMessages 是一个模板方法，以下是 NioServerSocketChannel 的具体实现：

![image-20201122231704714](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122231704714.png)

**SocketUtils.accept(javaChannel())  就是处理原生 Channel 的 accept 事件的方法，最终会返回一个 SocketChannel**，获取到之后直接将其包装为 Netty 扩展的 NioSocketChannel，并添加到缓冲链表里面。

读取完毕之后，会遍历该 List，**通过 channelRead 事件将 NioSocketChannel 传递出去**，读取结束在触发 channelReadComplete。

<br>

<br>

显而易见的，对 NioSocketChannel 进一步初始化以及绑定 EventLoop 的逻辑都在 channelRead 事件中。

在服务端初始化的时候，**添加的 ServerBootstrapAcceptor 也在这个时候发挥作用。**

<br>

## 二、 传播 channelRead 事件

 以下就是 ServerBootstrap$ServerBootstrapAcceptor#channelRead 的逻辑：

![image-20201122233715818](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122233715818.png)

上来就是连续的三种配置:

- 配置 ChannelPipeline

- 配置 ChannelOptions 

- 配置 Attributes

**以上三种都是在创建 ServerBootstrap 时就指定的。**

最后是走的注册方法，将初始化完毕的 NioSocketChannel 注册到 childGroup，注册的逻辑可以参考 ServerSocketChannel 的注册。

[Nio 服务端启动流程](./Nio服务端启动流程.md)



> ChannelPipeline 此时只是定义并未初始化，在注册到 EventLoop 之后才会正式的初始化。



## 总结

> SocketChannel 创建的整体流程

整个添加的流程如下:

1. Unsafe 通过 accpet 方法获取 JDK 原生 Channel
2. Accpeter 初始化 Channel，包括 ChannelPipeline，ChannelOptions 以及 Attribute
3. 注册到 workerGroup，开启读取。

<br>

因为 Netty 对 Channel 的重新封装比较好，NioServerSocketChannel（服务端 Channel） 和 NioSocketChannel（客户端 Channel） 的方法和表现基本一致，包括注册的方法。

不同的是，NioServerSocketChannel 在创建的时候指定的 readInterectOp 是 OP_ACCEPT 类型，而 NioSocketChannel 的是 OP_READ 类型。

响应 OP_ACCEPT 方法返回的是 SocketChannel，而处理 OP_READ 处理的是 ByteBuf。

![image-20201122235158546](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122235158546.png)

AbstractNioByteChannel 是 NioSocketChannel的直接父类。

![image-20201122235227811](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122235227811.png)



> 建立客户端 SocketChannel 是由 ServerBootstrapAcceptor 负责初始化和绑定的。

**ServerBootstrapAcceptor 继承自 ChannelInboundHandlerAdapter 负责响应 Accept 事件，生成 SocketChannel，并注册到 EventLoop。**

ServerBootstrapAcceptor 是在服务端 ServerSocketChannel 创建并初始化的时候 addLast 添加的（initAndRegister）。





> ChannelPipeline 的初始化

ChannelPipeline 在一开始都会被包装为 ChannelInitilalizer，然后在首次注册的时候调用 handlerAdd 方法。

