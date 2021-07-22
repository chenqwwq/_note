# 客户端 Channel 的建立过程

> 客户端连接之后的Channel对象创建和初始化过程。

[TOC]

## 概述

根据Reactor模型论述，接受连接创建 Channel 的工作就是由前端的 BossGroup 来完成的。

> 从 NioServerSocketChannel#accpet 返回的原生 Channel 进一步的包装成为 Netty 中的 NioSocketChannel。

以原生的Nio网络模型来看，就是处理ServerSocketChannel的`OP_ACCPE`事件。

经过之前的流程，NioServerSocketChannel已经绑定到BossGroup的EventLoop中，并且EventLoop也开启了轮询。

注册到本地的端口之后就开始了正常接收并响应客户端的请求。



![](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122230741489.png)



以上就是NioEventLoop#run方法也就是轮询方法中的一段，这里就包含了SelectionKey.OP_ACCEPT事件的处理。

**所以创建SocketChannel的起点就在`unsafe.read()`方法。**



## 一、获取并包装原生的 Channel

以下就是AbstractNioMessageChannel$NioMessageUnsafe#read方法的部分源码:

Unsafe的实现分为NioMessageUnsafe以及NioByteUnsafe，不同的是NioByteUnsafe只处理以ByteBuf为目标的读取，而NioMessageUnsafe读取所有对象。

![](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122231328166.png)

首先是通过doReadMessages方法，读取所有数据到readBuf中，readBuf是一个`List<Object>`对象，作为缓存,如下：

![image-20201122231444241](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122231444241.png)

AbstractNioMessageChannel#doReadMessages是一个模板方法，具体的实现在NioServerSocketChannel中。

![image-20201122231704714](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122231704714.png)

`SocketUtils.accept(javaChannel())`就是处理原生Channel的accept事件的方法，最终会返回一个SocketChannel。

这是原生的 SockChannel，获取到之后直接将其包装为NioSocketChannel，并添加到缓冲链表里面。

<font size=2>暂时忽略退出的逻辑</font>

接下来会对链表中所有的 NioSocketChannel 出发一遍`ChannelRead`事件，并最终触发`ChannelReadComplete`事件，也就是到了事件传播的流程。

触发的事件会在 ServerSocketChannel 绑定的 ChannelPipeline 中传播，并处理。



## 二、 传播channelRead事件

在第一步中，原生的Channel通过事件传播机制，来到了ChannelPipeline中，首先是`ChannelRead`事件:

![image-20201122233303967](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122233303967.png)

因为是`ChannelRead`事件，首先就到了HeadConetxt中。

HeadContext并没有处理，而是直接向后传播，最终到`ServerBootstrapAcceptor`中进行进一步处理。

`ServerBootstrapAcceptor`这个类就是在最开始创建服务端Channel并初始化的过程中指定的，以下是ServerBootstrap#init方法:

![image-20201122233545450](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122233545450.png)

可以看上图稍微回顾一下之前的流程，再来分析ServerBootstrapAcceptor的逻辑流程。

以下就是ServerBootstrap$ServerBootstrapAcceptor#channelRead的源码:

![image-20201122233715818](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122233715818.png)

上来就是连续的三种配置:

1. 配置`ChannelPipeline`
2. 配置`ChannelOptions`
3. 配置`Attributes`

以上三种都是在创建ServerBootstrap时就指定的。

最后是走的注册方法，将初始化完毕的NioSocketChannel注册到workerGroup中(图片中的childGroup)。



注册的逻辑可以参考ServerSocketChannel的注册。

到这里如果EventLoop是第一次调用注册方法也会开启自身的线程开始轮询并处理SocketChannel上的事件。







## 总结

> SocketChannel创建的整体流程

整个添加的流程如下:

1. 通过accpet方法获取JDK原生Channel
2. 初始化Channel，包括ChannelPipeline，ChannelOptions以及Attribute
3. 注册到workerGroup，开启读取。



因为Netty对Channel的重新封装比较好，NioServerSocketChannel和NioSocketChannel的方法和表现基本一致，包括注册的方法。

不同的是，NioServerSocketChannel在创建的时候指定的readInterectOp是OP_ACCEPT类型，而NioSocketChannel的是OP_READ类型。

![image-20201122235158546](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122235158546.png)

AbstractNioByteChannel是NioSocketChannel的直接父类。

![image-20201122235227811](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122235227811.png)



> 建立客户端SocketChannel是由`ServerBootstrapAcceptor`完成的

ServerBootstrapAcceptor继承自ChannelInboundHandlerAdapter负责响应`Accept`事件，生成SocketChannel，并注册到EventLoop。

ServerBootstrapAcceptor是在服务端ServerSocketChannel创建并初始化的时候addLast添加的(initAndRegister)。

