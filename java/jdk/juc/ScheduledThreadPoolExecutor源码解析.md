# ScheduledThreadPoolExecutor源码解析

> 2021/01/01 祝自己新年快乐。

[TOC]

---

## 概述

### 类图

![image-20210103234108744](/home/chen/github/_note/pic/image-20210103234108744.png)

`ScheduledThreadPoolExecutor`直接继承了`ThreadPoolExecutor`，所以它也完全可以当做一个线程池来使用。

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



## Fe