# Tomcat 的线程模型

---

[TOC]

---

## 概述

对 Tomcat 的理解不深，感觉上 Tomcat 算是对 Multi Raactor 的实现，但是 Reactor 是对所有网络请求处理方式的抽象，包括 TCP / UDP，而 Tomcat 仅仅针对于 Http 等部分应用层请求。

（所以以下算是整理，不会去细究源码。



## Tomcat 中的线程类型

- Acceptor 线程

往往是单线程组成的线程组，负责持有服务端的 Socket / SocketChannel（BIO 就是单纯的 Socket，NIO 会抽象成为 SocketChannel，并对以上的 accpet 事件做响应，建立客户端的 Socket / Channel。

Acceptor 建立的 Socket 会封装为 PollerEvent 并且放入 PollerEvent。

该线程类似于 Multi Reactor 的 Main Reactor（参考 Netty 的 ParentGroup，主要都是负责连接的建立。

<br>

- Poller 线程组

Poller 此时细分为 Poller 和 BlockPoller。

该线程组负责持有客户端的 Socket / Channel，通过 PollEvent 事件将其绑定到组中某个线程，并监听其上的事件。

该线程组类似于 Multi Reactor 的 Sub Reactor（参考 Netty 的 childGroup，默认的也是 2 * CPU 的线程数。

Netty 算是更加正规的 Reactor 的实现，再将客户端连接绑定到 childGroup 之后，连接的所有 IO 都由 childGroup 处理。

**在 Netty 中，对于 Http 请求，整个的请求中包含请求行，请求头，请求体都是由 childGroup 读取，而 Tomcat 中 Poller 线程组并不负责读取请求体（因此在判断为 Get 请求之后，有些实现可以直接忽略请求体。**

（以上特点可以从 Servlet 中看出，HttpServletReqeust 中获取请求体方法返回的实际是 InputStream。

<br>

- Worker 线程组

Worker 线程组就是具体的工作线程，类似于 Reactor 的 Handler 线程。

Worker 线程组在 Tomcat 中就是 Servlet 的执行线程，传入的 Request 以及 Response，线程组中的读写会另外交给 Block Poller 线程组。







## Refrence

- [Tomcat-线程模型详解](https://zzcoder.cn/2020/08/30/Tomcat-%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B%E8%AF%A6%E8%A7%A3/)