## 为什么说Netty是异步的

首先，Netty 主要还是基于 NIO 的，NIO 就是一个同步非阻塞的网络模型，那么为什么还说 Netty 是异步的呢？

其实这里的异步准确说是异步事件驱动，异步的不是底层的NIO这个网络模型，而是 Reactor 这种编程模型，Netty 底层的 IO 操作还是同步的，比如 read 和 write 等还是在 EventLoop 的主线程中执行的。

Reactor 将底层的同步 IO 抽象成事件，然后通过事件的分发器发给处理器进行处理，可以有效提高执行效率，除了 Netty，Redis 也使用的该类模型。



Netty 中分为前端的 bossGroup，以及后端的 workerGroup，实现上都是 EventLoopGroup。

通常来说 bossGroup 只包含一个线程，而 workerGroup 包含多个，每个线程就对应的一个 EventLoop 也就是事件轮询器。

上述的 EventLoop 每个还会持有一个 Selector 对象。

将 NioServerSocketChannel 注册到 bossGroup 的 EventLoop 就相当于注册到其上的 Selector 对象，负责监听OP_ACCEPT事件。

EventLoop响应OP_ACCEPT事件生成对应的SocketChannel并注册到IO线程池，并各自处理IO事件。



整理来说Netty中对应的三种线程有

1. Accept线程 - bossGroup中的线程，负责接受请求，创建SocketChannel
2. IO线程 - worker中的线程，负责处理read，connec，write等IO事件
3. 业务线程 - 负责处理read等IO事件中读取出来的数据



需要注意的是，Netty自身默认并不带有业务线程的概念，按照Reactor的规范，read操作读取出来的数据会被封装为事件然后交给业务线程处理。

但在Netty中，如果在创建ServerBootstrap的时候childHandler或者handler没有指定工作线程池的话，相应的逻辑还是在EventLoop的主线程执行的，也就是IO线程。



最后还有今天看到的一句重点，**Reactor整个模型还是基于同步IO的。**



