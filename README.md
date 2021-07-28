> 人菜还懒不定时间。

## Java

### JDK

- Stream
- CopyOnWrite

#### JUC

> JUC 是 JDK 中的核心并发包。

- [AbstractQueueSynchronized](java/jdk/java并发包/AbstractQueuedSynchronizer.md)
- [ConcurrentHashMap](java/jdk/java并发包/ConcurrentHashMap(1.8).md)
- [FutureTask](java/jdk/java并发包/FutureTask.md)
- [ReetrantLocak](java/jdk/java并发包/ReetrantLock.md)
- [ThreadPoolExecutor](java/jdk/java并发包/ThreadPoolExecutor.md)



<br>

#### JVM

> Java 虚拟机以及一些关键字

- [简单关键字整理](java/jvm/关键字.md)
- [引用类型](java/jvm/引用类型.md)
- [元空间和永久代](java/jvm/元空间和永久代.md)
- [synchronized](java/jvm/synchronized.md)
- [JVM 的类加载子系统](java/jvm/JVM的类加载子系统.md)



<br>

#### Java Collection

> Java 集合框架。

- [HashMap](java/jdk/java集合框架/HashMap源码阅读.md)
- [HashMap 原理(文字表述)](java/jdk/java集合框架/HashMap.md)
- [LinkedHashMap](java/jdk/java集合框架/LinkedHashMap源码阅读.md)
- [ArrayList](java/jdk/java集合框架/ArrayList.md)
- [LinkedList](java/jdk/java集合框架/LinkedList源码阅读.md)

### Netty

- [ChannelOption整理](netty/ChannelOption整理.md)

#### Netty 相关工具类

- [ChannelPool](netty/util/ChannelPool.md)

> ChannelPool 是 Netty 对于 Channel 的池化实现。

- [FastThreadLocal](netty/util/FastThreadLocal.md)

> FastThreadLocal 是 Netty 对 JDK 中 ThreadLocal 的优化类。

#### Netty 的相关流程

- [EventLoop 的初始化流程](netty/netty逻辑流程/EventLoop 的初始化.md)
- [NIO服务端启动流程](netty/netty逻辑流程/NIO服务端启动流程.md)
- [ChannelPipeline的结构和初始化流程](netty/netty逻辑流程/ChannelPipeline的结构和初始化流程.md)
- [NioEventLoop的事件轮询](netty/netty逻辑流程/NioEventLoop的事件轮询.md)
- [客户端Channel的建立过程](netty/netty逻辑流程/客户端Channel的建立过程.md)
- [ChannelPipeline事件传递机制](netty/netty逻辑流程/ChannelPipeline事件传递机制.md) - TODO
- [Netty 中的编解码逻辑](netty/netty逻辑流程/Netty中的编解码逻辑.md)
- [Netty 中写数据的逻辑](netty/netty逻辑流程/Netty中写数据的逻辑.md) - 未开始
- [Netty 中读数据的流程](netty/netty逻辑流程/Netty中读数据的流程.md) - 未开始

## Spring 全家桶
 > Spring 应该是 Java 必学的框架了，但是源码层面看的并不深。

### Spring Core

- [AOP相关概念](spring/spring-core/aop/aop 概念.md)
- [ProxyFactory](spring/spring-core/aop/ProxyFactory.md)
- [Aop 实现的不同方式](spring/spring-core/aop/Aop实现的不同方式.md)
- [Async的实现原理](spring/spring-core/module/async.md)

<br>

### SpringBoot

- [SpringBoot的事件模型](spring/spring-boot/SpringBoot的事件模型 .md)
- [SpringBoot的启动流程简述](spring/spring-boot/springboot的启动流程简述.md)
- [SpringBoot的工厂加载模式](spring/spring-boot/SpringBoot的工厂加载模式.md)

#### SpringBoot 的启动流程

- [SpringBoot的启动流程](spring/spring-boot/SpringBoot启动过程/SpringBoot的启动流程.md)
- [SpringBoot启动流程-环境准备](spring/spring-boot/SpringBoot启动过程/SpringBoot启动流程-环境准备.md)
- [pringBoot启动流程-上下文刷新](spring/spring-boot/SpringBoot启动过程/SpringBoot启动流程-上下文刷新.md)
- [SpringBoot启动流程-上下文准备](spring/spring-boot/SpringBoot启动过程/SpringBoot启动流程-上下文准备.md)
- [SpringBoot启动流程-下文刷新-Bean的预加载](spring/spring-boot/SpringBoot启动过程/SpringBoot启动流程-下文刷新-Bean的预加载.md)

<br>

### SpringCloud

 - [consul](spring/spring-cloud/consul.md) 
   - [feign](spring/spring-cloud/feign.md)
   - [zuul](spring/spring-cloud/zuul.md)


## 消息队列

- [RabbitMQ 如何保证消息不丢失](消息队列/RabbitMQ如何保证消息不丢失.md)

## Redis

- []()
- []
- [Redis缓存相关](redis/Redis缓存相关.md)

## Go 语言深入

[Go 语言设计与实现](https://draveness.me/golang/)



## 系统设计相关

- [Gof 的设计模式](系统设计/23种基本设计模式.md)
