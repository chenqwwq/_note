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
- [JVM 的类加载子系统](java/jvm/JVM 的类加载子系统.md)



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

- [0. EventLoop 的初始化流程](netty/netty逻辑流程/0. EventLoop 的初始化.md)
- [1. NIO服务端启动流程](netty/netty逻辑流程/1. NIO服务端启动流程.md)
- [2. ChannelPipeline的结构和初始化流程](netty/netty逻辑流程/2. ChannelPipeline的结构和初始化流程.md)
- [3. NioEventLoop的事件轮询](netty/netty逻辑流程/3. NioEventLoop的事件轮询.md)
- [4. 客户端Channel的建立过程](netty/netty逻辑流程/4. 客户端Channel的建立过程.md)
- [5.ChannelPipeline事件传递机制](netty/netty逻辑流程/5.ChannelPipeline事件传递机制.md)
- [6. Netty 中的编解码逻辑](netty/netty逻辑流程/6. Netty 中的编解码逻辑.md)
- [7. Netty 中写数据的逻辑](netty/netty逻辑流程/7. Netty 中写数据的逻辑.md)
- [8. Netty 中读数据的流程](netty/netty逻辑流程/8. Netty 中读数据的流程.md)

## Spring 全家桶

## 消息队列

- [RabbitMQ 如何保证消息不丢失](消息队列/RabbitMQ 如何保证消息不丢失.md)

## Redis

- []()
- []
- [Redis缓存相关](redis/Redis缓存相关.md)

## Go 语言深入

[Go 语言设计与实现](https://draveness.me/golang/)



## 系统设计相关

- [Gof 的设计模式](系统设计/23种基本设计模式.md)