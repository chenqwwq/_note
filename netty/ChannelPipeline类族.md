# ChannelPipeline & ChannelHandler & ChannelHandlerContext



Netty作为一个事件驱动的网络框架，ChannelPipeline就是其事件的传播渠道。





## ChannelPipeline类族

ChannelPipeline是全部的ChannelHandler的最外层容器，是它持有着Netty中整个业务逻辑的处理链路。

对于外界来说，ChannelPipeline就像是一个黑盒，你并不知道里面的结构和逻辑，但是在调用ChannelPipeline之后，请求会在其内部流转一番，会不会出来也不好说。



下图是ChannelPipeline接口注释中的配图:

![image-20201220234307264](/home/chen/github/_note/pic/image-20201220234307264.png)

从中可以知道方法的整个ChannelPipeline中，数据的流转过程。

Inbound是按顺序执行，outbound是按照逆序执行。



### ChannelPipeline接口

ChannelPipeline是整个相关类族中最基础的接口。

以下是ChannelPipeline的部分方法签名列表:

![image-20201220234049530](/home/chen/github/_note/pic/image-20201220234049530.png)

从中可以看出，ChannelPipeline作为一个容器的职责，就是负责ChannelHandler的CRUD。

另外的再看ChannelPipeline的类图，



![image-20201112231603678](/home/chen/Pictures/image-20201112231603678.png)

ChannelPipeline另外的实现了ChannelOutboundInvoker接口和ChannelInboundInvoker接口。



### ChannelOutboundInvoker、ChannelInboundInvoker

> 这两个接口定义了事件在Netty中触发的相关方法签名。

以下是ChannelInboundInvoker的方法签名列表:

![image-20201220235845131](/home/chen/github/_note/pic/image-20201220235845131.png)

**入站事件从Sokcet.read开始，都是对请求数据的处理。**

比如每次读取到数据都会调用`fireChannelRead`方法表明有读取到数据，而在读取数据完毕之后又会调用`fireChannelReadComplete`表明数据读取完毕。

另外的有连接可用，以及连接注册到Selector也都算作是入站事件，会调用`fireChannelActive`等方法。

> 以服务端ServerSocketChannel来说，Channel注册到Selector(EventLoop)之后就会触发是`fireChannelRegistered`事件，绑定到本地端口之后就会触发`fireChannelActive`。
>
> 这两个事件存在一定的先后顺序。



再来是ChannelOutboundInvoker的方法签名列表:

![image-20201221000134924](/home/chen/github/_note/pic/image-20201221000134924.png)

出站事件一般都是用户主动触发的，需要Netty帮忙处理的。

方法签名中定义了连接，绑定，注册，读，写，刷新缓冲区等。













## ChannelHandler类族

**ChannelHandler主要定义了Channel相关的事件以及事件处理的方法API，还有在添加和删除ChannelHanler到上下文的逻辑。**



### ChannelHandler接口

> 该接口定义了ChannelHandler的基本方法，是ChannelHanler类族中最原始的接口。
>

方法签名如下：

![image-20201112232458191](/home/chen/Pictures/image-20201112232458191.png)

**Sharable注解表示这个ChannelHandler是可以共享的。**

handlerAdded方法实现ChannelHandler添加到ctx(ChannelHandlerContext)的方法。

handlerRemove表示ChannelHandler删除方法。

ChannelInitializer就是通过该方法(handlerAdded)实现一次性将多个ChannelHandle实现类添加到ctx中。



> ChannelHandler中甚至没有定义任何一个事件处理相关的方法，只有handler新增和删除的方法，以及一个Sharable的基本注解。



### ChannelInboundHandler

> 该接口实现了ChannelHandler，并且定义了Netty中基本入站事件。

![image-20201220221654516](/home/chen/github/_note/pic/image-20201220221654516.png)

ChannelInboundHandler继承了ChannelHandler，也就实现了ChannelHandler中handlerAdded和handlerRemove方法，可以自定义的添加和删除Handler节点。

另外的扩展定义了入站相关的事件处理方法的方法签名。

![image-20201116214309769](/home/chen/Pictures/image-20201116214309769.png)

上图就是方法签名的列表，也就是Netty中的入站事件列表。

包括了以下的事件:



| 方法签名 | 对应事件|
| -------- | -------- |
| channelRegistered/channelUnregistered | Channel注册事件，服务端Channel在注册到Selector之后触发 |
| channelActive/channelInactive | Channel可用事件，服务端Channel在bind和register都完成之后触发。 |
| channelRead | Channel可读事件，EventLoop轮询Selector触发。 |
| channelReadComplete | Channel读取完成事件，在一次读取操作完成之后触发，其中可能包含多次的channelRead |
| userEventTriggered | 用户自定义事件的触发 |



### ChannelOutboundHandler

> 该接口实现了ChannelHandler，并且定义了Netty中基本出站事件。

![image-20201220221718801](/home/chen/github/_note/pic/image-20201220221718801.png)

同样的ChannelOutboundHandler也直接继承了ChannelHandler接口，可以控制实际上下文中处理的逻辑增删。

另外就是Netty出站事件的方法签名定义：

![image-20201116215405538](/home/chen/Pictures/image-20201116215405538.png)

看见方法名应该就知道了具体是哪些事件，这些方法在`HeadContext`中都有实现。





> 以上三个类基本奠定了ChannelHandler的作用，
>
> 1. 实现上下文中业务逻辑的增删(ChannelHandler)
> 2. 处理出站/入站事件，但是出站事件基本由Netty实现(ChannelOutboundHandler/ChannelInboundHandler)



### ChannelHandlerAdapter

> 该类并没有集上面三家所长，而是对ChannelHandler接口的基本实现，增加了对ChannelHandler是否可共享的判断。

![image-20201220222749264](/home/chen/github/_note/pic/image-20201220222749264.png)

ChannelHandlerAdapter是直接继承ChannelHandler的，是对ChannelHandler的基本实现，增加了对Sharable接口的判断。

接口的方法签名如下:

![image-20201112233835559](/home/chen/Pictures/image-20201112233835559.png)

`added`实例变量就是ChannelHandler是否已经被添加到ctx的标记。

`isSharable()`方法就是对当前ChannelHandler是否可以被多个ChannelPipeline共享的判断。

以下是isSharable的基本实现:

![image-20201112234028438](/home/chen/Pictures/image-20201112234028438.png)

**方法不难，在通过反射判断当前类是否标记了@Sharable之前还会通过ThreadLocal缓存来判断。**

`ensureNotSharable()`方法用来确保该ChannelHandler不能被共享，实现如下：

![image-20201220223527118](/home/chen/github/_note/pic/image-20201220223527118.png)



> 组合之下，ChannelHandler还有`ChannelInboundHandlerAdapter`以及`ChannelOutboundHandlerAdapter`等实现接口。



### ChannelInitializer

![image-20201220225920694](/home/chen/github/_note/pic/image-20201220225920694.png)

从类图中看起来，ChannelInitializer好像才是ChannelHandler的集大成者。

类图中的Channel是ChannelInitializer的泛型，ChannelInitializer需要指定一个`Channel`的子类为泛型对象类型。

以下为ChannelInitializer的方法签名列表:

![image-20201220230308997](/home/chen/github/_note/pic/image-20201220230308997.png)

首先ChannelInitializer实现了ChannelInboundHandlerAdapter中的`channelRegistered`方法，说明该类会在Channel注册的事件下生效。

以下是channelRegistered的源码:

![image-20201220230448223](/home/chen/github/_note/pic/image-20201220230448223.png)

首先调用了`initChannel(ctx)`方法，调用成功之后向后传递调用`removeState(ctx)`方法，如果调用失败则直接向后传递。

再来看两个方法的逻辑，首先是initChannel的:

![image-20201220230612631](/home/chen/github/_note/pic/image-20201220230612631.png)

方法中先添加ctx到initMap中。

![image-20201220232749479](/home/chen/github/_note/pic/image-20201220232749479.png)

initMap就是一个set，表示ChannelHandlerContext是否有已经初始化过，添加成功后开始正常的初始化逻辑。

initChannel(Channel)是ChannelInitial中定义的方法，以模板方法形式，子类实现该方法可以扩展初始化逻辑。

![](/home/chen/github/_note/pic/image-20201220233004713.png)

执行方法之后主要的来了，忽略异常处理代码之后还有finally代码块中的逻辑。

**不论添加的成功和时候，ChannelInitializer都会调用pipeline.remove删除自身**。

可以将ChannelInitializer当做容器，在里面的类已经初始化到外部的ChannelPipeline之后，ChannelInitializer就没用了，自己就删除了自身。



> ChannelInitializer是继承了inbound和outbound的逻辑，按道理来说也可以作为正常的ChannelHandler使用。
>
> 但是在这个方法中定死了，不论如何ChannelInitializer都会在ChannelPipeline初始化之后被删除。





## ChannelHandlerContext

整个ChannelPipeline的体系中，ChannelPipeline是外层的主要容器，负责ChannelHandler的整合以及调用顺序的流转。

ChannelHandler是具体的请求处理者，负责处理相关的请求数据以及事件，但是可以看到的是Channel

而ChannelHandlerContext




