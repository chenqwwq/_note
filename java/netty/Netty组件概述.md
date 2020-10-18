## Netty组件概述

>  基于4.1.53.Final版本

![image-20201018222617287](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018222617287.png)



[TOC]

Netty是用Java实现的高性能网络框架，它所做的事情基本就是对Java原生API的再次封装和扩展。

它有以下的特点：

1. 事件驱动
2. 异步IO(Netty中的IO操作都是异步的)



以下对Netty中涉及的各类组件进行一个系统性的概述，不会涉及很深。

## Bootstrap

Bootstrap就是引导类，负责整合其他的相关组件并对外提供服务。

Netty中ServerBootstrap提供服务端功能，Bootstrap提供客户端功能。

ServerBootstrap中持有了父子Channel的ChannelOption以及AttrbuteMap集合，Bootstrap也保存了自己的一份。



  ![image-20201018220355161](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018220355161.png)

ServerBootstrap和Bootstrap的类图也非常简单，不做介绍了。



## EventLoop/EventLoopGroup

EventLoop就是事件轮询器，而EventLoopGroup就是多个EventLoop的组合。

对于一般的服务端程序来说，会创建单个EventLoop的BossEventLoopGroup负责接受客户端连接请求，有多线程的WorkerEventLoopGroup来负责IO读写的操作。



 		![image-20201018221020963](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018221020963.png)

以上是NioEventLoop的基本类图。

EventLoop最初还是继承自JUC中的Executor接口，并不能说他是个线程池，但是它具有执行事件的功能。

不仅仅是执行事件，因为类图中还有ScheduleExecutorService的身影，简单推测EventLoop还存在定时执行的功能。

NioEventLoop继承自SingleThreadEventLoop，所以很显然的，NioEventLoop只会和一个Thread对象绑定。

​		 ![image-20201018221417144](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018221417144.png)

上图是NioEventLoopGroup的。

发现NioEventLoopGroup和NioEventLoop继承了基本类似的JUC原生接口，所以NioEventLoopGroup也会有类似于NioEventLoop的API方法。



NioEventLoopGroup通过组合的方式实现了对NioEventLoop的功能扩展。

这里可以简单瞄一下Group中具体的功能实现。



![image-20201018221901295](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018221901295.png)

next方法就是对Group中单个NIoEventLoop的选择方法，很显然Group的执行逻辑就是选择已有的EventLoop指派执行。

从这种层面可以验证，Group就是对多个NioEventLoop的整合和管理以及任务的分派，类似于线程池中ThreadPool和Worker的概念。



## Future

Netty中所有IO都是异步的，所以作为异步结果的接收类，Future也是相当重要的。

下图就是Netty中Future的大部分类族。

![image-20201018224213594](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018224213594.png)

Netty中并没有直接采用JUC中的Future，因为原生的接口非常简陋不满足Netty的一些功能。

所以Netty自定义了一个Future，最主要的就是增加了**方法回调的API**。

![image-20201018222307691](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018222307691.png)

从以上的方法可以很明显的看出Netty的Future接口比原生接口多实现的地方。

1. 增加了方法回调功能
2. 增加了不可中断的等待
3. 对任务是否可取消的配置



对于ChannelFuture，主要是增加了Channel的绑定方法，用于Channel相关的异步操作。

ChannelFuture的注释中提供了Channel对应不同的完成阶段验证的方法。

![image-20201018224625154](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018224625154.png)

如图中所示，如果Future对应的事务执行成功，isDone和isSuccess都为true，失败的话isDone为true但是cause不为空。

另外注释中还说了Netty开发过程中特别要注意的点，就是在ChannelHandle的方法中切勿直接调用await方法，该种代码大概率会造成死锁，而导致整个Pipeline失效。

类族中的CompleteFuture以及SuccessedFuture和FailedFuture都是特定状态下的Future，不关心结果，但方便用来设置方法回调。



最后的Promise。

![image-20201018225237042](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201018225237042.png)

由类的组成可以发现，Promise相对于Future新增了最终状态的设置方法，可以人为手动的设置事件的执行结果。

需要注意的是了理论上setFailure()和setSuccess()方法只能执行一个。