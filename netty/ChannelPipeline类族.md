# ChannelPipeline & ChannelHandler & ChannelHandlerContext



Netty作为一个事件驱动的网络框架，ChannelPipeline就是其事件的传播渠道。





## ChannelHandler

该接口定义了ChannelHandler的基本方法。

方法签名如下：

![image-20201112232458191](/home/chen/Pictures/image-20201112232458191.png)

Sharable注解表示这个ChannelHandler是可以共享的。

handlerAdded方法实现ChannelHandler添加到ctx(ChannelHandlerContext)的方法。

我们熟悉的ChannelInitializer就是通过该方法实现一次性将多个ChannelHandle实现类添加到ctx中。

handlerRemove表示ChannelHandler删除方法。



ChannelHandler中甚至没有定义任何一个handler相关的方法，只有handler新增和删除的方法，以及一个Sharable的基本注解。



## ChannelInboundHandler

ChannelInboundHandler继承了ChannelHandler，并且扩展定义了入站相关的事件处理方法的方法签名。

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





## ChannelOutboundHandler

Netty出站事件的方法签名定义。

![image-20201116215405538](/home/chen/Pictures/image-20201116215405538.png)

看见方法名应该就知道了具体是哪些事件。



## ChannelHandlerAdapter

ChannelHandler的基础扩展实现，扩展了对ChannelHandler是否已经添加的状态判断。

![image-20201112233835559](/home/chen/Pictures/image-20201112233835559.png)

added就是ChannelHandler是否已经被添加到ctx的标记。

isSharable()就是对当前ChannelHandler是否可以被多个ctx共享的标记。

以下是isSharable的基本实现:

![image-20201112234028438](/home/chen/Pictures/image-20201112234028438.png)

方法不难，在通过反射判断当前类是否标记了@Sharable之前还会通过ThreadLocal缓存来判断。ChannelInboundHandler

该接口定义了Channel的入站事件。

方法签名如下:

![image-20201112233101035](/home/chen/Pictures/image-20201112233101035.png)

除了从ChannelHandler接口继承的两个方法之外

## ChannelInboundInvoker & ChannelOutboundInvoker

ChannelOutboundInvoker和ChannelInboundInvoker分别定义了Netty中的出站和入站事件的调用形式。

首先是InboundInvoker的方法签名列表:

![image-20201112232209999](/home/chen/Pictures/image-20201112232209999.png)

然后是outboundInvoker的方法签名:

![image-20201116225951926](/home/chen/Pictures/image-20201116225951926.png)





## ChannelPipeline

以下是ChannelPipeline的类图:

![image-20201112231603678](/home/chen/Pictures/image-20201112231603678.png)