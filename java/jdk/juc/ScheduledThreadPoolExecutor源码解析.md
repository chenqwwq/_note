# ScheduledThreadPoolExecutor源码解析

> 2021/01/01 祝自己新年快乐。

[TOC]

---

## 概述

`ScheduleThreadPoolExecutor`的继承类图如下:

![image-20210103234108744](/home/chen/github/_note/pic/image-20210103234108744.png)

`ScheduledThreadPoolExecutor`直接继承了`ThreadPoolExecutor`，**所以它也完全可以当做一个线程池来使用。**

另外的相对于线程池，额外继承了`ScheduledExecutorService`接口，表明它可以提供`定时`以及`固定延迟`的周期性任务。

以下是`ScheduledExecutorService`接口的方法API列表:

![image-20210103234428384](/home/chen/github/_note/pic/image-20210103234428384.png)

其中`scheduleAtFixedRate()`方法就是定时任务，如果上一次任务结束时下次任务时间已到则直接开始下一次任务。

另外`scheduleWithFixedDelay()`方法就是固定延时的任务，任意两个任务之间保证固定的延时，就算上个任务拖到天荒地老，下个任务也会在完成后的一定延迟后执行。

> 简单总结一下，`ScheduledThreadPoolExecutor`可以执行的任务种类有如下四种种:
>
> 1. 正常任务（Runnable/Callable）
> 2. 延迟任务（One-Shot）
> 3. 定时任务（FixedRate）
> 4. 固定延迟任务 （FixedDelay）
>
> 定时任务和固定延迟任务其实也是基于延迟任务实现的，只要在任务完成之后重新将其加入等待队列就好。



> 线程池控制任务延迟调度的关键就在于`等待队列`，工作线程的执行逻辑照旧，只要等待队列控制任务的下发时间就可以控制任务的延时执行。

## 等待队列 - `DelayedWorkQueue`

`ScheduledThreadPoolExecutor`使用的内部类`DelayedWorkQueue`，就是类似于JDK中`DelayQueue`的实现，使用堆排序保证每次获取的任务都是最近的。

以下是`DelayedWorkerQueue`的成员变量:

![image-20210104115620871](/home/chen/github/_note/pic/image-20210104115620871.png)

因为是内部类只供内部使用，所以直接使用的`RunnableScheduledFuture`的数组。

从`TreadPoolExecutor`的`getTask()`方法中直到从等待队列中获取任务主要使用的是阻塞队列的`take()`和`poll()`方法。

以下是`DelayedWorkerQueue`的`take()`方法源码:

```java
// ScheduledThreadPoolExecutor$DelayedWorkQueue#task()
public RunnableScheduledFuture<?> take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        // 循环获取
        for (;;) {
            RunnableScheduledFuture<?> first = queue[0];
            // 队列为空的时候直接阻塞
            if (first == null)
                available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0)
                    return finishPoll(first);
                first = null; // don't retain ref while waiting
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && queue[0] != null)
            available.signal();
        lock.unlock();
    }
}
```





