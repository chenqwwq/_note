# Netty NIO服务端的启动流程

> 本文主要介绍Netty NIO服务端的启动流程，以Netty-Example包中的EchoServer为切入点。
>

[TOC]

## 概述

Netty是对JDK NIO的进一步封装，所以Netty服务端的启动流程也基本保持NIO的不变。

以下是原生的NIO启动代码的简单版本:

![image-20201021103408477](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201021103408477.png)

总体流程分为以下几步：

1. 创建并配置服务端Channel
2. 绑定端口
3. 创建Selector
4. 将Channel注册到Selector



**在EventLoopGroup的初始化流程中创建EventLoop的过程就会通过SelectorProvider生成一个Selector。**

所以剩下的就是Channel的创建，初始化以及注册。



## 源码分析

Netty的整个服务端启动是从bind方法绑定本地端口开始的，不过在这之前会先创建Bootstrap，初始化相关的参数。



### 创建Bootstrap

以下就是Bootstrap的初始化流程，以链式调用的形式，每个方法就相当于一个Set方法。

![image-20201020220224302](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201020220224302.png)



group方法指定一到两个EventLoopGroup，前面的Group用于响应连接请求建立Channel，后面的Group用于响应Channel的IO事件，这里就是简单的赋值到Bootstrap持有的成员变量。



channel方法的作用是指定服务端本地Channel的类型，如下就是channel方法的具体实现：

![image-20201020221857060](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201020221857060.png)

传入的Class类型会被用来初始化一个ReflectiveChannelFactory对象，该对象负责生成具体的ServerSocketChannel。

该ChannelFactory比较好理解，简单来说就是通过反射来创建一个具体的实例对象。



option方法指定的是TCP的相关参数，这里就是把指定的参数和值的k/v键值对添加到Bootstrap持有的options集合中。



handler方法指定了服务端Channel的业务逻辑，在创建完ServerSocketChannel初始化的时候就会绑定到Pipeline上上。

childHandler是指定对于子Channel的业务逻辑。



Bootstrap创建完成时，其实父子Channel类型等一些基础的配置都已经定了，之后就是借助Bootstrap快速启动一个服务。



### 绑定本地端口

![image-20201020221203610](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201020221203610.png)

Bootstrap的bind方法在经过简单的验证之后就会调用到AbstractBootstrap的doBind方法。

具体的源码如下：

![image-20201020221333204](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201020221333204.png)

方法的整体逻辑如下：

1. 创建，初始化并注册服务端Channel
2. 绑定端口

以上说的注册是指注册到EventLoop中的Selector中。





#### Channel的创建，初始化并注册

如题所说，该阶段会完成服务端Channel的创建，初始化以及注册到EventLoop的流程。

以下是AbstractBootstrap#initAndRegister方法(省略了异常处理部分):

```java
    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            // 创建新的Channel,根据ChannelFactory不同获取不同的Channel
           // channelFactory就是通过Bootstrap创建时的channel设置的
            channel = channelFactory.newChannel();
            /**
             * 对Channel进行初始化
             *
             * @see ServerBootstrap#init(Channel)
             * @see Bootstrap#init(Channel)
             */
            init(channel);
        } catch (Throwable t) {
           ...
        }

        // 注册Channel到EventLoop中
        // register方法会EventLoopGroup中的一个进行绑定
        // doubt: 不懂为什么要通过config()再获取group()，直接group()获取不就好了
        ChannelFuture regFuture = config().group().register(channel);
 		...
        return regFuture;
    
```

**首先是创建Channel，通过调用ChannelFactory。**

Bootstrap在创建的时候会指定服务端Channel的类型，并生成一个Factory对象，就是在这里使用的。

ReflectiveChannelFactory的newChannel方法如下:

![image-20201020233834802](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201020233834802.png)

newChannel方法会直接去调用具体的Channel类的构造函数，以下就是NioServerSocketChannel的无参构造。

![](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201020222846939.png)

可以看到的是从一开始NioEventSocketChannel就通过SelectorProvider创建了一个JDK的Channel对象，之后以Channel为参数进一步调用构造函数。

![image-20201020223026446](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201020223026446.png)

这里值得注意的是初始化参数中带上了`SekectionKey.OP_ACCEPT`，之后又一路来到了AbstractNioChannel的构造函数。

![image-20201020223346147](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201020223346147.png)

在这里将channel设定为了非阻塞模式，并保存了原生channel和InterestOp，以及父Channel。

至此就完成了Channel的创建，**之后就是初始化Channel**，以下是init方法的源码:

![image-20201020231341009](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201020231341009.png)

初始化一共涉及以下的流程:

1. 配置ChannelOptions
2. 配置Attributes
3. 添加服务端Channel的ChannelHandler

重点在第3步，首先是从Bootstrap中捞出配置的Handler，之后还会增加了一个`ServerBootstrapAcceptor`的Handler，这个ServerBootstrapAcceptor非常关键，后续还会讲到。



**在Channel初始化完成之后就是注册流程了，将生成的Channel注册到EventLoopGroup中。**

以下是



#register()的源码

![image-20201020231649549](/home/chen/Pictures/image-20201020231649549.png)

很清楚就是从EventLoopGroup管理的EventLoop中选择一个，然后调用它的register()方法，看到返回值是ChannelFutures就表示Channel的注册流程也是异步的。

一路Debug下来最终register会调用到AbstractChannel@AbstractUnsafe#register(EventLoop eventLoop, final ChannelPromise promise)

![image-20201020232442518](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201020232442518.png)

这个方法整个的重点就是进一步调用register0方法，但是有意思的是它调用前的判断。

**如果当前线程在EventLoop中就直接调用，否则通过EventLoop执行最终的注册流程**，这里体现了Netty的一个执行原则：Channel相关的事件都由Channel绑定的EventLoop执行。

再来看register0方法：

![image-20201020232753589](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201020232753589.png)

首先doRegister()方法是个模板方法，真实的注册逻辑会延迟到子类实现。

以下是AbstractNioChannel的实现:

![image-20201020232902821](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201020232902821.png)

这里的注册就是将Channel注册到Selector上，参数0表示此时并不会关注Channel上任何的事件，并带上当前对象作为attribute。

之后是设置Promise为成功，并触发`ChannelRegistered`事件。

如果首次注册还会额外触发`ChannelActive`事，如果不是首次触发，并且配置中开启了`AutoRead`选项(ChannelIOption.AUTO_READ)，则直接调用beginRead()方法，开始接受请求。

beginRead方法会一步步调用到AbstractNioChannel的doBeginRead方法:

![image-20201020234945923](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201020234945923.png)

该方法就是为注册到Selector的SelectionKey增加`readInterestOp`的值。

`readInterestOp`是构造函数中传入的从NioServerSocketChannel的构造函数中指定的`SelectionKey.OP_ACCEPT`。



以上就完成了Channel的整个创建，初始化并注册流程，逻辑不复杂，但是调用的链路真的多。

返回的一个ChannelFuture实际就是为了传递Channel。





#### 端口绑定

Channel创建之后就需要绑定到本地的端口。

端口绑定的步骤在一长串的调用之后就来到了NioServerSocketChannel#doBind方法:

![image-20201021112416014](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201021112416014.png)

简单来说就是根据当前的JDK版本选择绑定的方式。

这里还带上了ChannelConfig中的Backlog参数，该参数指定了TCP的半连接队列。





## 总结

Netty的服务端启动流程在代码上非常简单，因为借助Bootstrap完成了大部分组件和配置的管理。

只需要调用bind方法就可以快速开启服务。



整个服务端的启动流程步骤:

1. 创建EventLoopGroup

   EventLoopGroup会自行去创建EventLoop，创建EventLoop的时候也会创建对应的Selector

2. 初始化Bootstrap

   指定两个EventLoopGroup，ChannelOption，ChannelHandler等服务端配置

3. 创建Channel

   Channel是根据Bootstrap中的配置，通过反射创建的，创建之后就会设置为非阻塞模式

4. 初始化Channel

   包括ChannelOption的配置，以及父子Channel的Handler配置。

5. 注册Channel

   这里的注册是指注册到EventLoop绑定的Selector中，并触发ChannelRegistered事件

6. 