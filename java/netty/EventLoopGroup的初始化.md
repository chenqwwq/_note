# EventLoopGroup的初始化

> Netty作为一种事件驱动的网络框架，EventLoop事件轮询器就是它的核心组成。
>
> 本文以NioEventLoop为主要目标分析整个的初始化流程。



[TOC]

## 概述

[EventLoop相关的部分内容](./Netty组件概述.md/#EventLoop)

Netty中，每个NioEventLoop都会绑定一个线程已处理其上绑定的对象所产生的事件。

每个NioEventLoop都会绑定至少一个的Channel，并且启动一个Selector(Nio底层的轮询器)去监视Channel上就绪的事件，在事件发生后又会发起IO的读写操作或者建立新Channel。



## 类族部分

![image-20201019221558931](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201019221558931.png)



整个类族中EventExecutorGroup起到了承上(JDK原生接口)启下(Netty扩展接口)的作用。

EventExecutorGroup重新定义了ScheduleExecutorService中的大部分API，并将返回值从JDK的Future改为Netty的Future，另外提供了next()方法用于在Group中选择一个EventExecutor。



EventLoopGroup和EventLoop，EventExecutorGroup和EventExecutor之间的关系非常类似，通过继承和组合的方式完成结构组织。

EventExecutor和EventLoop提供parent()方法，获取所属的Group，EventLoopGroup和EventExecutorGroup提供next()方法选择合适的Executor或者Loop。

例如EventLoop继承了EventLoopGroup，并且EventLoopGroup中保存了EventLoop的集合，EventLoopGroup和EventLoop中有相同的方法API，在Group中会通过next()选择一个EventLoop来实现相同的功能。





对于EventLoopGroup，是在EventExecutorGroup基础上提供了Channel相关的注册方法，这里可以理解为EventLoopGroup就是**EventExecutorGroup对Channel的扩展，主要还是用来处理Channel相关的事件，但是因为间接继承了Executor接口，也可以用来执行其他任务甚至延时任务。**

对于EventLoop，基本完成承袭EventLoopGroup的方法。







## 源码部分

```java
 	   // Configure the server.
        // doubt: EventLoop，EventLoopGroup，EventExecutor等类之间的关系
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
```

以上就是最基础的遵从Reactor模型的Netty服务端启动时会创建的两个EventLoopGroup。

bossGroup为前置的连接请求处理，负责监听Selector的Accept请求，并创建Channel，分派给workerGroup监听。

workerGroup监听Channel上的IO事件，并处理。



外面看着创建的参数简单，其实是因为内部约定了许多。

以下是NioEventLoop中参数最全的构造方法了。

![image-20201019222615970](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201019222615970.png)

EventLoopGroup本身并不会有直接执行任务的时候，基本上都是选择一个EventLoop，然后调用它的方法执行事务。

所以参数的后四个，也就是**SelectorProvider，SelectStrategyFactory，RejectedExecutionHandler，TaskQueueFactory**，对于创建EventLoopGroup本身是无用的，但是都会在创建NioEventLoop时，作为参数传入。

参数具体的作用在下文会解释。



NioEventLoopGroup往下就是MultithreadEventLoopGroup的构造函数。

![image-20201019222746272](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201019222746272.png)

其中`DEFAULT_EVENT_LOOP_TRHEADS`为默认的线程数，**Netty中定义的是当前CPU数的两倍**，两倍CPU数的线程非常适合执行IO密集型的任务，所以看到创建workerGroup时采用的默认的线程数。

![image-20201019222919297](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201019222919297.png)

再往下就是MultithreadEventExecutorGroup的构造函数了。

```java
// MultithreadEventExecutorGroup的构造函数
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
    	// 首先检查线程数
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }
         // 执行器为空的话需要新建一个
        if (executor == null) {
            // 这个执行器很特别，任何任务都会直接开一个新的线程执行
            // 这里创建的Thread其实经过包装，真实对象是FastThreadLocalThread，肯定是继承了Thread的
            // newDefaultThraedFactory就是默认的线程工厂
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

    	// 初始化Group中的执行器数组
        // EventExecutor的数量和线程数量一致
        children = new EventExecutor[nThreads];
		// 遍历创建执行器
        for (int i = 0; i < nThreads; i++) {
            boolean success = false;
            try {
                /**
                 * 最终创建的就是NioEventLoop
                 *
                 * @see NioEventLoopGroup#newChild
                 */
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // 异常处理暂时忽略
            }
        }

        // 用创建的NioEventLoop，构造一个选择器
        chooser = chooserFactory.newChooser(children);
        // 声明一个监听器
        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                // 操作是否完成
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        // 为每一个事件处理器添加监听
        // 每个EventLoop在关闭之后都会触发上面的监听器
        for (EventExecutor e : children) {
            e.terminationFuture().addListener(terminationListener);
        }

        // 为Group的所有处理器保存一份只读版本
        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }
```

参数中用可变长参数适配不同参数数目的调用，鉴于Netty的牛逼我就不予置评了。

整个构造函数就做了如下几件事情:

1. 创建默认的执行器。
2. 创建nThread个EventLoop。
3. 通过EventExecutorChooserFactory创建对应的EventExecutorChooser。
4. 为每个EventLoop添加关闭时的监听器。





### 创建默认的执行器

此时执行器的具体作用未知。

以下是ThreadPerTaskExecutor的全部代码。

![image-20201019223907388](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201019223907388.png)

这个执行器的功能，简单来说就是每来一个任务就通过ThreadFatcory开启一个新的线程去执行。

再来看ThreadFactory的部分，以下是`DefaultThreadFactory`创建新线程的方法。

![image-20201019223644097](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201019223644097.png)

首先会发现，Netty中对Thread和Runnable也做了进一步的包装，采用了自定义的FastThreadLocalThread和FastThreadLocalRunnable。

对于线程名字的话就是前缀加上编号。

> 这里对FastTreadLocalThread和FastThreadLocalRunnable并不是很清楚，为什么采用自定义的FastThread***



### 创建EventLoop

以下是NioEventLoop#newChild的方法逻辑。

![image-20201019224315626](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201019224315626.png)

简单来说就是创建NioEventLoop的基本对象，但是用到了一些调用EventLoopGroup构造函数时传递的参数。

接下来就是一连串的EventLoop的构造函数的连续调用。

![image-20201019230708750](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201019230708750.png)

以上是NioEventLoop部分的构造函数，主要是就是通过SelectorProvider创建一个新的选择器。





### 创建EventExecutorChooser

以下是DefaultEventExecutorChooserFactory的newChooser方法源码。

![image-20201019232145341](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201019232145341.png)

**通过事务执行器的线程个数是否为二次方判断使用哪种Chooser。**

以下分别是两种Chooser的方法。

PowerOfTwoEventExecutorChooser的如下:

![image-20201019232346486](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201019232346486.png)

GenericEventExecutorChooser的如下:

![image-20201019232445792](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201019232445792.png)

比较之下，PowerOfTwoEventExecutorChooser就是在线程数为2次方的时候作的细微优化，采用&取代%，位操作能提升一些代码性能，

具体方式的也可以参考HashMap中的选址方式。

有意思的是这个方法，判断一个数是否是二次方:

![image-20201019232710692](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201019232710692.png)

恩。。位操作拿来秀操作是真的不错。





### 添加EventLoop关闭时的监听器

