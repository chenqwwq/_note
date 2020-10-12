## Netty源码级分析

> 基于4.1.53.Final版本

Netty是用Java实现的高性能网络框架，它所做的事情基本就是对Java原生API的再次封装和扩展。





重点看到几个部分：

1. EventLoopGroup的创建流程
2. Pipeline的初始化和执行流程
3. 异步化编程基础，Future,Promise
4. 对象池
5. SO_REUSEPORT
6. 

