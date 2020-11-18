# FastThreadLocal及其相关

> FastThreadLocal相关源码阅读

## 概述

FastThreadLocal是Netty中实现的一个线程局部变量，是对ThreadLocal的优化，其实就是重新写了一种ThreadLocal的方式。

FastThreadLocal需要搭配FastThreadLocalThread使用，如果线程仍然是原生的Thread类，那么就会走原生的ThreadLocal。



Netty实现的FastThreadLocal其实还包括了另外一些相关类:

1. FastThreadLocalThread
2. InternalThreadLocalMap
3. FastThreadLocalRunnable





## FastThreadLocal的get流程

以下是FastThreadLocal的

![image-20201119001005945](/home/chen/github/_note/pic/image-20201119001005945.png)







## InternalThreadLocalMap

原生的ThreadLocal的实现方式重点就是在Thread类中添加了一个ThreadLocalMap类，以存放对应的数据。

同样的先看FastThreadLocalThread