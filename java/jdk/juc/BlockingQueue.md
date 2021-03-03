# BlockingQueue 阻塞队列

[TOC]

---

## 概述

BlockingQueue 阻塞队列是 JUC 中新增的集合类，在集合为空时获取阻塞，在集合满时添加阻塞。



> 阻塞队列的基本实现就是采用 JUC 中提供的 ReentrantLock 重入锁和 Condition 竞态条件。
>
> 使用 ReentrantLock 保证线程安全，使用 Condition 在特定条件下阻塞或唤醒线程。 

阻塞队列提供了以下几种方法套件:

| 结果               | 插入        | 移除       | 获取    |
| ------------------ | ----------- | ---------- | ------- |
| 异常               | add         | remove     | element |
| true/false or null | offer       | poll       | peek    |
| 阻塞               | put         | take       |         |
| 等待               | offer(time) | poll(time) |         |



> await/signal 和 wait/notify 的对比:
>
> 1. 前者基于 JUC 的 AQS 机制实现，而后者基于 JVM 的 Monitor 机制实现。
> 2. await/signal 体系一次上锁可以对应多个 Condition，而 wait/notify，一次的上锁只支持一个等待队列

## LinkedBlockingQueue

以链表为底层结构的阻塞队列，以下为链表的 Node 节点:

![](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20210302154637945.png)

节点数据非常简单，所谓的阻塞都是通过 Condition 实现的。

以下是实现阻塞的相关变量:

![image-20210302154844242](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20210302154844242.png)

takeLock 和 putLock 是在新增和获取的时候的锁对象。

![image-20210302155205123](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20210302155205123.png)

在 take() 方法中通过 takeLock 上锁，保证添加过程的线程安全，如果此时队列为空则使用 notEmpty 阻塞获取线程，并调用 notFull 唤醒因为队列已满被阻塞的线程。

以下是 put 方法的源码:

![image-20210302170438492](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20210302170438492.png)

put 方法使用 putLock 上锁，使用 notFull 阻塞添加线程，并且使用 notEmpty 唤醒阻塞线程。

> 因为底层采用的是链表结构，数据之间相互的影响不大，所以 LinkedBlockingQueue 的锁是读写分离的。
>
> (个人感觉)

## ArrayBlockingQueue

数组实现的有界阻塞队列，新增和获取时都有可能被阻塞。

**不同于 LinkedBlockingQueue，ArrayBlockingQueue 只有单个重入锁，生成两个 Condition。**

![image-20210302163949275](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20210302163949275.png)

ArrayBlockingQueue 的入队和出队基本都使用 enqueue 和 dequeue 方法来实现，在 put 方法上锁，然后线程安全的执行 enqueue 方法。

以下显示 put 方法的源码:

![image-20210302170552537](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20210302170552537.png)

上锁后开始判断容量，如果容量已满则使用 notFull 阻塞当前线程，等待唤醒，唤醒之后进入 enqueue 方法。

![image-20210302164348387](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20210302164348387.png)

添加元素到原数组，然后通过 notEmpty 唤醒因为集合为空被阻塞的获取线程。

> ArrayBlockingQueue 没有扩容的说法，容量从初始化之后就是固定的。