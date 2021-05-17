#  Summary Record

## 2021/05/07

这段时间不是没有学习，但是好像已经没啥找工作的动力了，现在的公司业务相对简单，摸鱼好像也挺快乐的。

最近又开始看 Netty 的东西了，从编解码开始又看了读写的流程，还给 Netty 提了一个三行代码的 PR，哈哈 顺手顺手。



晚上重新修改了一下 EventLoop 的初始化 那篇笔记，再细化一些结构上的内容吧。

但还是没有明确到任务队列的实现，采用的应该还是 MPSC 队列，但是构造函数里面使用可变长参数真的是桑心病况。



最近还在学 ES，哈哈哈 感觉 Kafka 啥的又要忘了。

中间件的各种使用还是很重要的，光记概念太容易忘了，希望下家公司多点这种使用吧，en...或者让我去写中间件吧。



想往大厂冲一冲啊！！！



push 之后再看看 es 的内容吧，等小姐姐下班说不定还能聊聊天 哈哈哈，老舔狗了。



## 2021/04/19

晚上简单总结了一下 MySQL 的索引，看了几页 ES 的文档。

还有大顶堆/优先级队列的实现：

> 插入时直接在末尾，逐个对比上升，删除时，直接将末尾元素移动到顶部，然后对比下降。

[索引相关](./mysql/索引相关.md)



## 2021/04/15

工作不忙，生活上有点忙，哈哈哈，侧面算充实吧。

[elasticsearch](/./elasticsearch/极客时间课程.md)



## 2021/04/07

算是告一段落吧，还有前后端的编译没有仔细看，还有Class文件的整体结构没有详细看。

前端编译就是.java文件变成.class文件的过程，期间会有解语法糖的过程，Java里的语法糖不多，包括自动拆装箱，可边长参数，泛型等等，泛型擦除后会变成基本的类型，如果使用了 extend ,super 之类的还会有变化。

后端编译就是JIT将class文件直接变为本地机器码的过程，根据热点探测编译部分代码。

编译的优化涉及到的就多了，比如逃逸分析，这个在Go里面也有，逃逸分析是很多优化的基础，只有确定变量的作用范围才能进一步做细化的，比如标量替换甚至栈上分配，因为栈的生命周期是固定的就是方法的执行期，所以将变量的生命周期直接和栈绑定不失为一个减少gc压力的好办法，只是栈的空间一般较小，不适合分配太大的对象。



Netty 相关的脱稿整理：

Netty 算是 Reactor 模型的实现，可以实现单线程/多线程的 Reactor 模型。



ChannelPipeline 是每个 Channel 都会有的，Channel 就是 Socket 的上层抽象，以 Channel 为模型方便了对下层实现的屏蔽。

比如服务端 Channel 常用的 NioServerSocketChannel ，在创建的时候就会打开一个 Selector 对象，然后打开并绑定 ServerSocket。

初始化的时候同时添加了 ServerBootstrapAcceptor 的 ChannelHandler，响应 ServerSocketChannel 里的 channelRead 事件，创建客户端 Channel 对象并绑定到一个 EventLoop 中。



EventLoop 就是一个事件执行器，感觉上类似于 Redis 的网络模型，但是 Redis 是由前端线程读取后经由事件分派器分发，而EventLoop 中的事件就是由 EventLoop 这个线程执行或者另外指定线程池。

EventLoop 中的事件有以下三种：

一种就是网络的IO事件，ServerSocketChannel 监听 Accept 事件，而 SocketChannel 监听的是 READ 事件。

一种是定时任务，EventLoop 本身就继承了 Schedule 相关的接口，所以有执行定时任务的能力，心跳就是在借此发起的，而且到期的任务会被转移到 taskQueue 中。

最后一种是由程序提交的任务，程序提交的任务又分了两种，一种是按照 io 事件的百分比可能会被执行到的，就是在 SingleThreadEventExecutor 中定义的 taskQueue 。

另外一种是在 SingleThreadEventLoop 中定义的 tailTasks 任务，这些任务不受上述限制，每次轮询都会执行完所有任务。



##  2021/04/05

清明真就啥都没干....

[JVM 知识点脑图](https://www.processon.com/view/link/6064961d6376891a06bbe35e)



## 2021/03/07

Async，ProxyFactory

> Async 是 Spring 中的简化异步模型。
>
> Async 不会提供常规的返回值，可以以 Future 作为方法返回值获取结果，也可以以 ListenableFuture 为方法的返回值。
>
> ListenableFuture 是 Spring 对 Future 的扩展，类似于 Netty 中的 Future，增加了回调机制，默认实现有 AsyncResult。
>
> Async 默认使用容器中的 TaskExecutor 实例，同样的可以在 @Async 中指定。
>
> TaskExecutor 是 Spring 对 Executor 的基础扩展，也可以说专门用来实现 Async 的。
>
> 感觉上可以考虑实现一个指定默认执行器的 EnableAsync。
>
> 
>
> Async 使用了 BeanPostProcessor 拦截类的初始化，为需要的类创建代理对象，区别于 Feign 的直接扫描。
>
> 内部创建的 AnnotationAsyncExecutionInterceptor，都没有向容器彻底暴露。

> Spring 中存在 JDK Proxy 以及 CGLIB 两种代理方式。
>
> 如果有实现接口或者代理的对象就是接口（CrudRespority）采用的 JDK Proxy，也就是 JdkDynamicAopProxy，不然就是 ObjenesisCglibAopProxy。
>
> JdkDynamicAopProxy 就是 InvovationHandler 的子类，该类最终通过 Proxy.newProxyInstance 来创建代理对象，然后在 invoke 中插入执行 Advice 的方法。

> 在 Spring 的 Aop 模型中:
>
> Pointcut 表示的切点，负责匹配某些方法或者类，分别 ClassFilter 和 MethodMatcher 两种基础类。
>
> Advice 表示增强逻辑，在匹配到的地方会额外执行 Advice 的方法。
>
> Advisor 是以上两者的整合。
>
> Advised 包含了所有的 Advisor 和 Advice，代表的是一个代理的相关的所有配置，它不是代理对象，但是是生成代理对象的前提。



## 2021/03/08

木了。



## 2021/03/09

AQS

> AQS 内部队列可以分为阻塞队列和条件队列。
>
> 阻塞队列是实现独占锁和共享锁的基础，独占锁和共享锁一样，竞争失败之后直接进入阻塞队列(阻塞队列是延迟初始化的)，自旋修改前驱节点状态为 SIGNAL 后挂起当前线程，并在前驱为头节点的时候才会真正自旋等待。
>
> 独占和共享的区别就是共享锁在获取锁时会扩散，扩散就是锁状态的往后传播，每次只会扩散到当前节点的后继节点。



## 2021/03/10

AQS - ConditionObject

>条件队列完全有 ConditionObject 实现，每个 AQS 都可以有多个 ConditionObject。
>
>ConditionObject 通过 Node.nextWaiter 字段维护一个单向的队列。
>
>！Node 类在同步队列中使用 prev 和 next 维护节点的双向关系，在条件队列中通过 nextWaiter 维护单向关系。
>
>在调用 await 后，先以 CONDITION 状态入条件队列，在释放所有锁资源，锁资源释放失败就表示原本就每持有锁资源，就抛异常，已经入队列的节点变为 CANCELLED 状态。
>
>唤醒方法 signal 就是从队头开始唤醒第一个有效节点，将其状态从 CONDITION 变为 0，然后入同步队列，并不一定直接唤醒线程。
>
>await 状态下的线程被唤醒后，会先检查中断状态，判断唤醒原因，然后检查是否在同步队列中，如果是开始执行同步队列的自旋操作。
>
>自旋仍然可能被阻塞，唤醒后继续检查中断状态。



## 2021/03/11

AQS - interprete，CountDownLatch

> ! Thread#interrupt 方法不仅会修改线程的中断标志位，也会唤醒被阻塞的线程。
>
> 在 AQS 实现的条件队列中，await 方法阻塞的线程，除了 signal 正常唤醒的逻辑之外，也能被中断方法唤醒。
>
> 由中断唤醒的线程会照常入队列，**在竞争到锁之后才会进一步处理中断标志位**。
>
> THROW_IE 模式表示中断在 SIGNAL 之前发生，相当于线程是单纯的被中断唤醒的，所以处理方式就是直接抛出异常。
>
> REINTERRUPT 表示中断发生在 SIGNAL 方法调用之后，线程可能是被中断唤醒的，也可能是获取到锁被唤醒的，此时就是再次修改中断标志位就好。
>
> 这是 await 方法对中断的处理方式。



> CountDownLatch 是建立在 AQS 共享锁基础上的计数器。
>
> 初始化的 count 表示需要释放锁的次数，也就是 countDown 的调用次数，在达到这个次数之前，所有调用获取锁方法，也就是 await 方法的线程都会被阻塞。



## 2021/03/12

更新简历。

后续开始复习 Redis，Kafka之类的，四月初开始面试，没有是满意的就不换了，感觉涨点工资并不是我的目标。





## 2021/03/16

借由 @Async 理解 Spring 的代理和 JDK Proxy 的关系。

> JDK Proxy 中使用 Proxy.newInstance 方法创建的代理对象，可以转化为所有实现的接口。
>
> 代理对象必须是 **InvocationHandler** 的子类，并实现该类的 invoke 方法，之后对产生的代理的所有接口方法的调用都会转到 invoke 方法中。

> JdkDynamicAopProxy 自身就是一个 InvocationHandler 的实现，所有实现的代理类最终都会调用到该类的 invoke 方法。

TODO: JdkDynamicAopProxy 的源码有空可以看一下，现在先到此为止吧。

重新回顾一下 Spring 事务相关的知识点，包括传播和隔离。





## 2021/03/18

下午和群里小伙伴讨论了以下 Spring 的事务在**加入**的情形下，内层事务对外层事务的影响，外层 catch 异常之后的逻辑。

> 内层事务和外层事务公用一个底层事务，所以内层和外层的事务同时提交，同时回滚。
>
> 在执行内部方法的时候如果抛出异常，就已经让事务代理类感知到了，所以即使是 catch 住了内外也是一起回滚。
>
> 群里的说他试了并没有回滚，可能情况如下:
>
> 1. 同个类内使用 this 指针调用
> 2. 非 public 方法
>
> 总结就是内层的事务并未生效。

Feign 相关

Feign 会为每个 FeignClient 创建是一个服务上下文，使用的 FeignContext 保存，继承了 NamedContextFactory，每一个上下文中保存了 Encoder，decoder 等组件。

**FeignClientFactoryBean** 在创建代理的时候通过有没有 url 属性判断，有 url 就不会走负载均衡。

**Contract 在 Feign 中用来解析方法上的注解**，常用的就有 SpringMvcContract，用来解析 RequestMapping 等注解。

**Feign#Builder** 是用来创建最终代理对象的类，HystrixFeign#Builder 继承了 Feign#Builder，添加了 HystrixInvocationHandler 的工厂类，也替换了 Contract。

**Targeter** 是在创建之前做一些额外的配置，比如 HystrixTargeter 就在默认的 Targeter 的基础上多解析了 fallback 和 fallbackFactory。

**InvocationHandler** 就是 JDK Proxy 中的那个，是最终执行的请求的类，默认是 ReflectiveFeign.FeignInvocationHandler，而 Hystrix 采用的是 HystrixInvocationHandler。

**Client** 是最终执行的请求的类，常用的有 LoadBalancerFeignClient 类。



## 2021/03/21

又浪费了周末的两天，自律性看来有待提高啊，嘿嘿，好歹单词还是继续背的，这两天复习加背诵的应该快400个单词了。

周赛又是两题下机，第三题数据量有点大，开赛时没想到二分。

接下来的计划，整理一下 RabbitMQ / Kafka 的 Spring 客户端配置，重新看一下 Kafka。

面试前还想再看一些东西，Redis 已经 over 了，AQS 线程池什么的也没问题，Spring IOC / AOP / Async / 事务 / Boot 的启动流程 / Cloud / Consul / feign 。

还差 Ribbon / Hystrix 的原理部分。

1.  RabbitMQ 客户端的配置
2.  MQ ，消费的幂等性保证，消息积压问题，ACK 相关
3.  JVM 
4.  MYSQL / 分表分库





## 2021/03/22

RabbitMQ

Docker 启动 RabbitMQ UI 管理界面一直有问题，rlgl。

RabbitMQ 的备份交换机，通过 alternate-exchange 绑定某交换机，那么该交换机上所有为匹配到队列的消息都会被转发到该交换机。

mandatory 参数，保证消息实际投递到队列，发送者设置了 mandatory 参数之后，如果参数在 Broker 端无法找到匹配的队列，则会调用 return 方法。

> 对应 SpringBoot#RabbitMq 中的 ReturnCallback，该回调只有在匹配队列失败之后返回，而不管有没有发送到交换机。
>
> 而 ConfirmCallback 对应的则是生产者到交换机的步骤，如果消息未到交换机则回调，而不管消息是否有匹配到队列。



RabbitMQ 相对于 Kafka 来说，

在生产者端，发消息并不是直接和 Queue 关联，而是通过一种 Exchange 的角色中转，可以实现多队列发送，或者模糊匹配发送，而 Kafka 通过分区器也只是发给单个 Topic，相对来说 实现了 AMQP 的 RabbitMQ 更加灵活。

在消费者端，Kafka 通过消费者组的概念，可以让多个消费者同时消费一个队列，或者多个消费者组重复消费，而在 RabbitMQ 中，单个队列无法被多个消费者消费，所以如果想实现类似 Kafka 的多消费者消费相同的消息，可以在 Exchange 的时候就将消息发送到多个队列。



## 2021/03/24

[RabbitMQ 保证消息不丢失](mq/RabbitMQ 如何保证消息不丢失.md)





## 2021/03/31

[不同AOP实现方式的整理](./spring/spring-core/aop/Aop 实现的不同方式.md)

[SpringBoot启动流程简述](./spring/spring-boot/springboot的启动流程简述.md)
