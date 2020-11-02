## Netty组件概述

>  基于4.1.53.Final版本

![](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018222617287.png)



[TOC]

Netty是用Java实现的高性能网络框架，它所做的事情基本就是对Java原生API的再次封装和扩展。

它有以下的特点：

1. 事件驱动
2. 异步IO(Netty中的IO操作都是异步的)



以下对Netty中涉及的各类组件进行一个系统性的概述，不会涉及很深。

## Bootstrap

Bootstrap就是引导类，负责整合其他的相关组件并对外提供服务，利用Bootstrap可以快速地拉起一个服务。

在Netty中，ServerBootstrap提供服务端功能，Bootstrap提供客户端功能。

ServerBootstrap中持有了父子Channel的ChannelOption以及AttrbuteMap集合。



  ![image-20201018220355161](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018220355161.png)

ServerBootstrap和Bootstrap的类图也非常简单，不做介绍了。



## EventLoop & EventLoopGroup

EventLoop就是事件轮询器，而EventLoopGroup就是多个EventLoop的组合。

对于一般的服务端程序来说，会创建单个EventLoop的BossEventLoopGroup负责接受客户端连接请求，由多个线程的WorkerEventLoopGroup来负责IO读写的操作，另外的业务线程执行一些容易阻塞的业务逻辑(新版本可以直接使用JDK的线程池)。    static final SelectStrategy INSTANCE = new DefaultSelectStrategy();

以下是NioEventLoop的基本类图，以此来分析整个NioEventLoop的作用。

 		![image-20201018221020963](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018221020963.png)



根本上，EventLoop还是继承自JUC中的Executor接口，所以也就具有了**执行任务(Runnable)**的功能。

NioEventLoop直接继承自SingleThreadEventLoop，SingleThreadEventLoop提供了低级别任务的集合实现。

![image-20201101221514529](/home/chen/Pictures/image-20201101221514529.png)

往上的SingleThreadEventExecutor则是实现了完整的执行器逻辑，持有具体的任务队列，以及Thread对象。

![image-20201101222047766](/home/chen/Pictures/image-20201101222047766.png)

再往上一层的AbstractScheduledEventExecutor，持有的是一个优先级队列，以任务的执行时间为排序依据，实现的是定时任务。

![image-20201101222135423](/home/chen/Pictures/image-20201101222135423.png)

从上往下可以说，**NioEventLoop已经具有了多个任务执行模式，定时任务，普通任务以及低级别任务。**



​		 ![image-20201018221417144](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018221417144.png)

上图是NioEventLoopGroup的类族结构。

发现NioEventLoopGroup和NioEventLoop继承了基本类似的JUC原生接口，所以NioEventLoopGroup也会有类似于NioEventLoop的API方法。

MultithreadEventExecutorGroup抽象类提供了对子类的组合以及选择功能，以下是类中的成员变量:

![image-20201101222652273](/home/chen/Pictures/image-20201101222652273.png)

有数组和Set两种形式保存EventExecutor，再通过EventExecutorChooser来选择。

上面说的就是Netty中常见的一种代码结构，组合和继承。

例如EventLoopGroup中保存了EventLoop的集合，并和EventLoop继承相同的父类，然后EventLoopGroup中的实现就是选择集合中其中一个的EventLoop并调用一样的方法。

以下是AbstractEventExecutorGroup的默认实现，可以很清楚的看到这种实现形式。

![image-20201018221901295](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018221901295.png)

next方法就是对Group中单个NIoEventLoop的选择方法，很显然Group的执行逻辑就是选择已有的EventLoop指派执行。

从这种层面可以验证，Group就是对多个NioEventLoop的整合和管理以及任务的分派，类似于线程池中ThreadPool和Worker的概念。



## Future

> Future

Netty中所有IO都是异步的，所以作为**异步结果的接收类**，Future也是相当重要的。

下图就是Netty中Future的大部分类族。

![image-20201018224213594](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018224213594.png)

Netty中并没有直接采用JUC中的Future，因为原生的接口非常简陋不满足Netty的一些功能。

所以**Netty自定义了一个Future**，最主要的就是增加了**方法回调的API**。

![image-20201018222307691](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018222307691.png)

从以上的方法可以很明显的看出Netty的Future接口比原生接口多实现的地方。

1. 增加了方法回调功能
2. 增加了不可中断的等待
3. 对任务是否可取消的配置



**对于ChannelFuture，主要是增加了Channel的绑定方法，用于Channel相关的异步操作。**

ChannelFuture的注释中提供了Channel对应不同的完成阶段验证的方法。

![image-20201018224625154](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018224625154.png)

如图中所示，如果Future对应的事务执行成功，isDone和isSuccess都为true，失败的话isDone为true但是cause不为空。

另外注释中还说了Netty开发过程中特别要注意的点，就是在ChannelHandle的方法中切勿直接调用await方法，该种代码大概率会造成死锁，而导致整个Pipeline失效。

类族中的CompleteFuture以及SuccessedFuture和FailedFuture都是特定状态下的Future，不关心结果，但方便用来设置方法回调。



最后的Promise。

![image-20201018225237042](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018225237042.png)

由类的组成可以发现，Promise相对于Future新增了最终状态的设置方法，可以人为手动的设置事件的执行结果。

需要注意的是了**理论上setFailure()和setSuccess()方法只能执行一个。**



## Channel

> Channel指的是io.netty.channel.Channel，而不是JDK原生的Channel。
>
> 以下内容没有特指都是指Netty的Channel。

Channel在Netty中实际连接的抽象类，其中集成了对网络的读写以及关闭等操作。

Netty中对于服务端和客户端也有不同的区分，例如NioServerSocketChannel和NioSocketChannel。

相对于JDK原生的Channel，Netty自定义的Channel会承担更多的职责。

![image-20201025224424993](/home/chen/Pictures/image-20201025224424993.png)

以上是Netty的Channel类的整个结构。

首先Channel类中包含了一个Unsafe的接口，里面定义了bind，register等偏向底层的操作，是作为上层业务开发人员不需要去管的地方。

其次它继承了AttributeMap，也就获得了属性的配置能力。

另外实现了ChannelOutboundInvoker，表明Channel可以用来处理各类出站事件。

之后就是各种的方法了：

1. eventLoop提供了时间轮询器的获取功能，Netty中Channel都需要注册到一个EventLoop上，来完成自身IO事件处理。
2. parent提供了一个Channel间的层级关系，对于NioServerSocketChannel来说，它的parent就为空，而对于客户端的Channel来说，他的父类就是对应的ServerSocketChannel。
3. pipeline用来获取IO事件的处理管道，Channel上所有的事件都会经过全部或者部分的Pipeline来实现。
4. metadata就是同来存储Channel的属性。



Channel每个实际的Channel中都包含一个Unsafe的实现，以下是Unsafe接口的API部分:

![image-20201102222134985](/home/chen/Pictures/image-20201102222134985.png)

看到的都是和JDK底层实现有关，包括Channel到Selector的注册，本地端口的绑定，远程服务的连接。

![image-20201102233455653](/home/chen/Pictures/image-20201102233455653.png)

以上是Unsafe的部分类图结构，主要是Nio部分，NioMessageUnsafe主要用于服务端的Channel实现，例如NioServerSocketChannel，而NioByteUnsafe主要是客户端Channel的实现，例如NioSocketChannel。

## ChannelHandler & ChannelHandlerContext & ChannelPipeline






## Netty相关流程

