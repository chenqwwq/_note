# ChannelPipeline构造过程

[TOC]

## 概述

Netty作为一个异步的事件驱动的网络框架，ChannelPipeline毫无疑问是其中的重中之重。

如果以一个管道来理解Pipeline，所有的事件就是流入的水，在管道中多个节点间传递，最终再从入口出去。

ChannelPipeline承载了Netty中几乎所有业务逻辑的处理。

针对每一个Channel都会持有一个ChannelPipeline的实例对象，其间并不会共享，也就是说Channel之间的ChannelPipeline之间都是相互独立的，并不会互相影响。



## 一、创建ChannelPipeline

整个管道的创建，这是起点。

ChannelPipeline的创建在AbstractChannel的构造函数中:

![image-20201112224248294](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201112224248294.png)

newChannelPipeline是直接通过的newChannelPipeline方法创建，这里的This就是指当前的Channel的实例:

![image-20201112224330696](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201112224330696.png)

也**可以看出Netty中ChannelPipeline默认的实现对象就是DefaultChannelPipeline。**

> Netty中的ChannelPipeline默认使用的是DefaultChannelPipeline。

接下来就是DefaultChannelPipeline的构造方法：

![image-20201112224410788](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201112224410788.png)

构造方法中首先指定了所属的Channel，这里指定的Channel，可以直接通过DefaultChannelPipeline.channel()方法获取。

之后就是调用链的初步构造，**每一个ChannelPipeline在创建的时候都会填充HeadContext和TailContext两个节点**。

> 这个非常关键，ChannelPipeline的构造函数中就包含了Head和Tail两个节点的初始化，这是每个ChannelPipeline都有的基本。
>
> 并且Head和Tail使用的都是不同的实例对象。

相当于初始的Pipeline就是下面的样子：

![image-20201112224954785](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201112224954785.png)

经过初始化，也就是构造函数的调用之后，就得到了默认的DefaultChannelPipeline对象，现在的管道中除了对应Channel，也已经有了HeadConetxt和TailContext这两个头尾节点。



> 另外也可以看到的是ChannelPipeline中的节点并不是ChannelHandler类实例，而是AbstractChannelHandlerContext类，
>
> 这是经过了又一层的包装。



## 二、添加自定义ChannelHandler

在Bootstrap绑定端口的流程里，创建完服务端Channel之后就需要对Channel绑定的ChannelPipeline对象完成初步的初始化。

以下是当时的源码:

![image-20201112223953044](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201112223953044.png)

这里首先获取的ChannelPipeline对象，而后调用了addLast方法，将ChannnelInitialzer类加入到了业务链中。

**注意这个对象是嵌套的，内部类中通过config.handler()方法获取到的可能也是一个ChannnelInitialzer。**

在ChannnelInitialzer的方法被调用时，同时又会加入新的ChannnelInitialzer。

> ChannelInitialzer的作用是一次插入多个ChannelHandler，但是最主要的作用不是在这。
>
> addLast方法多次调用就会多次添加ChannelHandler，而在Bootstrap中，多次调用handler却会覆盖。
>
> 不过放在这里的作用也就是将ChannelPipeline的所有初始化逻辑延后，在第一次真实调用的时候才初始化。



以下是addLast的最终实现，上述的addLast经过几次调用后会执行到下面的逻辑：

![image-20201115205341924](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201115205341924.png)

以上的源码包含了添加到ChannelPipeline的基本逻辑，和addFirst/addBefore等逻辑基本一致。

### 1. 检查重复添加

首先会先进行检查，是否被多次添加，以下是检查的源码:

![image-20201115205450808](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201115205450808.png)

就很简单的检查ChannelHandler是否可以共享，isSharble会使用反射检查该ChannelHandler是否有@Sharable注解。

ChannelHandler.added属性就是是否已经添加的标志，**添加过的ChannelHandler该属性会被置为true**。

> 这一步主要还是注意@Sharable的作用。
>
> 简单来说，**只有@Sharable标记的ChannelHandler可以被多个ChannelPipeline使用。**





### 2. 包装成ChannelHandlerContext

这一步是对ChannelHandler进一步包装变成ChannelHandlerContext。

> 由单一性原则来讲，**ChannelHandler只负责业务逻辑的实现，在业务链中传递事件的机制主要还是由ChannelHandlerContext实现的。**
>
> ChannelPipeline中都是对



以下是具体实现的源码:

![image-20201115210005935](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201115210005935.png)

首先Group就是在调用addLast时指定的线程池，可以为空，也可以指定EventExecutorGroup类型，这个是Netty定义的事件处理器组的概念，其实也就相当于平时用的线程池。

**childExecutor方法就是从group中获取一个EventExecutor，不过中间还有一个SINGLE_EVENTEXECUTOR_PER_GROUP的事情。**

也就是说会从指定的线程池中选取一个线程与该ChannelHandlerContext绑定。

该参数就是控制ChannelPipeline和EventExecutorGroup的绑定关系，配置该参数之后，每个ChannelPipeline通过childExecutor方法都只能获取到同一个EventExecutor。

**如果对多个ChannelPipeline配置了同个EventExecutorGroup时，最终都会选择同一个线程，这样也就避免了线程上下文的切换。**

不过如果配置的不同的线程池就好像没啥用处。

这里的绑定关系也很简单，在ChannelPipeline中保存了`childExecutors`属性，该属性保存了以EventExecutorGroup为Key，获取的线程为Value的缓存。

从EventExecutorGroup中筛选的逻辑就是next()方法，总得来说就是轮询。



> 这里可以直到ChannelHandler的执行线程就是从传入的Group中选择的，可能为空。
>
> 另外还有`SINGLE_EVENTEXECUTOR_PER_GROUP`的作用



### 3. 添加到链表

包装成ChannelHandlerContext之后就是真正的入队列逻辑了。

下面是真实的添加逻辑:

![image-20201115215940177](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201115215940177.png)

不论怎么添加，自定义的ChannelHandler都是在head和tail两个默认的ChannelHandlerContext之间的。



在添加到队列之后，就根据当前是否已经注册分为了几个逻辑。

1. 未注册
2. 注册并且就是当前线程
3. 注册但不是当前线程

> 尾插法，新加入的ChannelHandler会放在链表的最后面，tail节点的前面。



此时ChannelHandlerAdapter已经加入到了ChannelPipeline中，之后的就是附加的添加ChannelHandler的逻辑了。

就是ChannelHandler中定义的`handlerAdded`方法的调用逻辑，因此每个ChannelHandler的实现类都有附加的逻辑。

ChannelInitializer就是在添加之后进一步移除自己。

### 4.  未注册到EventExecutor

![image-20201115222208845](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201115222208845.png)

根据上图，在未注册到EventExecutor的时候，会先设置为ADD_PENDING状态。

然后会调用callHandlerCallbackLater方法，将HandlerAdded作为一个延时任务保存。

> 这里完全遵守Netty中Channel的执行逻辑的，Channel的任何逻辑都由绑定的EventLoop执行，没有就添加任务并等待。

![image-20201115221651953](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201115221651953.png)

pendingHandlerCallbackHead就是ChannelPipeline中保存的延时任务队列的链表头。

未注册时候的逻辑很简单，**就是将当前的ChannelHandlerContext包装为PendingHandlerAddedTask，然后放到链表中。**

然后这个延时链表什么时候被执行呢？

这里执行的时机有两个:

1. 当Channel被注册到EventLoop的时候
2. 触发channelRegistered的时候



#### a). Channel注册

![image-20201115223309026](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201115223309026.png)

以上是Channel注册差不多末尾的逻辑，在doRegister方法调用完之后就会直接调用invokeHandlerAddedIfNeeded方法。



#### b). channelRegister事件

以下是DefaultChannelPipeline对channelRegistered的响应:

![image-20201115223442413](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201115223442413.png)

这里也会调用invokeHandlerAddedIfNeeded方法，并且传播channelRegistered事件。

两个触发的途径最终都会调用到invokeHandlerAddedIfNeeded方法，以下就是该方法源码:

![image-20201115223621907](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201115223621907.png)

firstRegisteration属性是为了保证这个方法植被调用一次，也就是说在channel注册完或者channelRegister事件发布之后就不能再通过相同的方法添加ChannelHandler了。

以下是具体执行任务的方法callHandlerAddedForAllHandlers：

![image-20201115231257755](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201115231257755.png)

就是将pendingHandlerCallbackHead中的任务遍历执行，其中registered保证该方法也仅仅被执行了一次。

执行完ChannelHandler的handlerAdded方法包装的任务之后，整个事件的传播链也就搭建好了。



### 5. 已注册到EventExecutor

已经注册到EventLoop则会判断当前线程是否为轮询线程。

![image-20201115222339374](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201115222339374.png)

当前线程不是的时候会调用如下方法:

![image-20201115222349025](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201115222349025.png)

使用指定的线程池执行当前的逻辑，这里的ExentExecutor就是之前childExecutor选择出来的，相当于异步调用callHandlerAdded0方法。

最终还是回到了callHandlerAdded0方法:

![image-20201115222542157](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201115222542157.png)

忽略掉异常处理，其实就只有上面几行代码，直接调用的ChannelHandlerContext的callHandlerAdded方法。

![image-20201115222650392](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201115222650392.png)

callHandlerAdded方法也不难，**就是设置状态为REMOVE_COMPLETE，之后调用ChannelHandler的handlerAdded方法。**

因为handlerAdded方法是在ChannelHandler中声明的，所以也不是只有ChannelInitializer才能增加多个ChannelHandler。







## 总结

> ChannelPipeline的结构简述

ChannelPipeline作为整个事件传播链的主体，**持有了双端链表的head和tail节点**，分别对应这HeadContext和TailContext。

另外**调用链中的主体并不是ChannelHandler，而是ChannelHandlerContext。**

初始化的过程如下:

首先在注册之前我们通过handler方法指定的ChannelHandler都会被包装成ChannnelInitialzer添加到链表中。

之后在首次注册的时候还会调用这个ChannelHander的handlerAdded方法，完成首次的完整添加，期间嵌套的ChannnelInitialzer被释放出来，各自的handlerAdded方法被调用，最终形成一个完整的事件传播链。

在注册之后的添加都会直接调用ChannelHandler的handlerAdded方法，也就是在业务执行期间我们也可以添加Handler的。



最后尤其要注意的就是@Sharable的作用，只有标注了@Sharable的ChannelHandler才可以一个对象被多个ChannelPipeline使用。

因为可能涉及到多线程的调用，所以应该只有无状态的ChannelHandler标记@Sharable比较合适。



> @Sharable注解的实现逻辑

@Sharable表示ChannelHandler接口，准确来说是ChannelHandlerAdapter类，是否可以被多个Pipeline共享。

因为在addLast方法的最开始就会检查ChannelHandler是否为ChannelHandlerAdapter的子类，如果不是则跳过检查。

**@Sharable仅可以于ChannelHandlerAdapter联用，表明ChannelHandler可以被多个Pipeline共享。**

**如果ChannelHandlerAdapter的实现类没有标注@Sharable，那么就只能被添加到一个ChannelPipeline中，不然会报错。**



对于@Sharable的使用，建议ChannelHandler的实现都继承ChannelHandlerAdapter，并在**无状态的ChannelHandler**上标注@Sharable。



> 构建的整体逻辑简述

<img src="/home/chen/github/_note/pic/ChannelPipeline%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96%E6%B5%81%E7%A8%8B.png" style="zoom: 50%;" />

1. 检查方法不违反Netty的规定

   ChannelHandlerAdapter实例不能被多个ChannelPipeline添加，除非标注了@Sharable

2. 包装为ChannelHandlerContext

   此时从传入的EventLoopGroup中选择一个执行器。

3. 添加到链表末尾

4. 执行连带的添加方法

   handlerAdded方法可以在添加一个ChannelHandler之后，附带的添加另外的。