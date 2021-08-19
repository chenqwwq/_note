# SynchronousQueue

---

[TOC]

---



## 概述

SynchronousQueue 继承于 BlockingQueue，因此也保留了 BlockingQueue 的几个特性：

1. 不接受任何 NULL 值
2. 在队列满时添加元素会阻塞当前线程
3. 在队列空时删除元素会阻塞当前线程

和其他 BlockingQueue 不同的是，**SynchronousQueue 不保存任何元素，甚至无法查看或者遍历其中的元素，只有在有线程删除的时候才可以进行添加元素，反之亦然。**

并且 **SynchronousQueue 的主要逻辑集中在 Put / Take 这对方法**上，调用 offer 添加元素时，如果没有配对的删除操作也不会阻塞。



> 可以将 SynchronousQueue 当做是**线程的一对一匹配器**，生产和消费两种线程互相匹配。





## 实现原理





## 参考

[SynchronousQueue 实现原理](https://zhuanlan.zhihu.com/p/29227508)