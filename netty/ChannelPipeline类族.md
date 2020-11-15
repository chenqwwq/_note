# ChannelPipeline



Netty作为一个事件驱动的网络框架，ChannelPipeline就是其事件的传播渠道。





## ChannelHandler

该接口定义了ChannelHandler的基本方法。

方法签名如下：

![image-20201112232458191](/home/chen/Pictures/image-20201112232458191.png)

Sharable注解表示这个ChannelHandler是可以共享的。

handlerAdded方法实现ChannelHandler添加到ctx(ChannelHandlerContext)的方法。

我们熟悉的ChannelInitializer就是通过该方法实现一次性将多个ChannelHandle实现类添加到ctx中。

handlerRemove表示ChannelHandler删除方法。



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

ChannelOutboundInvoker和ChannelInboundInvoker分别定义了Netty中的出站和入站事件。

首先是InboundInvoker的方法签名列表:

![image-20201112232209999](/home/chen/Pictures/image-20201112232209999.png)







## ChannelPipeline

以下是ChannelPipeline的类图:

![image-20201112231603678](/home/chen/Pictures/image-20201112231603678.png)