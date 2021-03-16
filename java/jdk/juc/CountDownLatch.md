# CountDownLatch

> - chenqwwq 2021/03

---

[TOC]

---

## 概述

CountDownLatch 是利用 AQS 的共享锁机制实现的计数器。



> 调用 CountDownLatch#await 方法的所有线程会阻塞直到有指定个线程调用 CountDownLatch#countDown 方法。



## 实现简述

CountDownLatch#await 方法调用的是 AQS 中共享锁的获取方法：

<img src="/home/chen/_note/pic/image-20210311225241495.png" alt="image-20210311225241495" style="zoom:67%;" />

而 CountDownLatch#countDown 方法调用的也是共享锁的撤销方法：

<img src="/home/chen/_note/pic/image-20210311225334959.png" alt="image-20210311225334959" style="zoom:67%;" />

每次固定的释放一个资源。



再来就是 CountDownLatch 内部的同步控制器的实现：

![image-20210311225400166](/home/chen/_note/pic/image-20210311225400166.png)



其实只要清除 tryAcquireShared 和 tryReleaseShared 两个模板方法的定义就能清除逻辑了。

> tryAcquireShared 在普通意义上是获取共享资源的方法，返回值大于0表示获取成功。

所以在 state 不等0的时候，所有的 await 方法调用线程都会进入同步队列，并阻塞。

> 同步队列是尾插法，所以先进入的节点会在前面。



> tryReleaseShared 在普通意义上表示释放共享资源，返回值为 true 表示资源释放成功。

CountDownLatch$Sync 的 tryReleaseShared 每次都会 -1，只有在 state = 0 时，才会返回 true。

之后会从头节点开始唤醒阻塞的有效节点，每个节点再各自向后扩散，直到阻塞的线程都被唤醒。





> 因为 CountDownLatch 的 tryReleaseShared 实现，只要求 state 等于0就算是获取成功，所以只要在 await 之前调用的 countDown 方法也是有效的。

