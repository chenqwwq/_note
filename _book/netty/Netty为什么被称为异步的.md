## 为什么说Netty是异步的

首先，Netty主要还是基于NIO的，NIO就是一个同步非阻塞的网络模型，那么为什么还说Netty是异步的呢？

其实这里的异步准确说是异步事件驱动，异步的不是底层的NIO这个网络模型，而是Reactor这种编程模型，Netty底层的IO操作还是同步的，比如read和write等还是在EventLoop的主线程中执行的。

Reactor将底层的同步IO抽象成事件，然后通过事件的分发器发给处理器进行处理，可以有效提高执行效率，除了Netty，Redis也使用的该类模型。



Netty中分为前端的bossGroup，以及后端的workerGroup，实现上都是EventLoopGroup。

通常来说bossGroup只包含一个线程，而workerGroup包含多个，每个线程就对应的一个EventLoop也就是事件轮询器。

上述的EventLoop每个还会持有一个Selector对象。

将NioServerSocketChannel注册到bossGroup的EventLoop就相当于注册到其上的Selector对象，负责监听OP_ACCEPT事件。

EventLoop响应OP_ACCEPT事件生成对应的SocketChannel并注册到IO线程池，并各自处理IO事件。



整理来说Netty中对应的三种线程有

1. Accept线程 - bossGroup中的线程，负责接受请求，创建SocketChannel
2. IO线程 - worker中的线程，负责处理read，connec，write等IO事件
3. 业务线程 - 负责处理read等IO事件中读取出来的数据



需要注意的是，Netty自身默认并不带有业务线程的概念，按照Reactor的规范，read操作读取出来的数据会被封装为事件然后交给业务线程处理。

但在Netty中，如果在创建ServerBootstrap的时候childHandler或者handler没有指定工作线程池的话，相应的逻辑还是在EventLoop的主线程执行的，也就是IO线程。



最后还有今天看到的一句重点，**Reactor整个模型还是基于同步IO的。**



