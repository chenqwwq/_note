#  Summary Record



---

[TOC]

---

## TO DO

Hystrix，Ribbon，Sentinel，Retry，Transation

## 2020/03/07

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



## 2020/03/08

木了。



## 2020/03/09

AQS

> AQS 内部队列可以分为阻塞队列和条件队列。
>
> 阻塞队列是实现独占锁和共享锁的基础，独占锁和共享锁一样，竞争失败之后直接进入阻塞队列(阻塞队列是延迟初始化的)，自旋修改前驱节点状态为 SIGNAL 后挂起当前线程，并在前驱为头节点的时候才会真正自旋等待。
>
> 独占和共享的区别就是共享锁在获取锁时会扩散，扩散就是锁状态的往后传播，每次只会扩散到当前节点的后继节点。



## 2020/03/10

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