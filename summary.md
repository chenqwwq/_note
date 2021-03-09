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

AQS，ConcurrentHashMap