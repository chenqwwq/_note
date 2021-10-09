# HashedWheelTimer 源码分析

> 2021/10/09



## 概述

HashedWheelTimer 是 Netty 中实现的时间轮结构（在Kafka，Akka，Zookeeper 中都有时间轮的实现。

**时间轮的主要作用就是存储并调度延时任务。**

> JDK 的 ScheduledThreadPoolExecutor 使用的是小顶堆（二叉树）来完成任务的延时调度。
>
> 存放的时候以执行时间作为排序依据，每次判断堆顶任务是否到期来确定是否需要调度执行，相对于时间轮来说，小顶堆的维护更加复杂，接近 O(logN) 的复杂度。

时间轮的整体结构如下：

![时间轮结构](assets/%E6%97%B6%E9%97%B4%E8%BD%AE%E7%BB%93%E6%9E%84.jpg)

（盗我杰哥一个图。

时间轮可以简单看作一个环形队列，队列中每个 Bucket（槽或者桶，怎么叫都行），都保存了一个任务链表，另外每个槽位代表一定的时间，时间到了之后将槽内的任务全部取出来执行。

添加任务的时候需要根据执行的时间，将任务插入到不同的槽位，然后等待具体的调度时间。

**因为是以槽作为一个调度时间单位，所以时间轮的精度就不会太高。**





## 时间轮的基本结构

HashedWheelTimer 中保存了 HashedWheelBucket 的数组，作为任务的存储结构（就是上述环形队列的简单实现。

![image-20211009140554597](assets/image-20211009140554597.png)

其中单个 Bucket 的表示是其内部类 HashedWheelBucket，结构如下：

![HashedWheelTimer#HashedWheelBucket](assets/image-20211009135932291.png)

HashedWheelBucket 中持有任务 HashedWheelTimeout 的头尾节点，相当于**持有当前 Bucket 的任务链表**。

以下是 HashedWheelTimer 的结构：

![image-20211009140311833](assets/image-20211009140311833.png)

HashedWheelTimer 中所以的定时任务都会被包装为 HashedWheelTimeout，自身通过前驱（prev）和后继（next）节点组成了一个双向的链表。

相关参数含义如下：

| 参数名          | 含义                                                         |
| --------------- | ------------------------------------------------------------ |
| next / prev     | 链表的前驱和后继                                             |
| deadline        | 与时间轮启动时间对齐的相对执行时间                           |
| remainingRounds | 剩余的轮次，每个轮次表示时间轮转动一周，每次访问到该节点算是一个轮次 |
| state           | 任务状态                                                     |
| task            | 需要执行的任务                                               |

> **HashedWheelTimer 中整个时间轮的结构就是双向链表组成的数组，数组中每个元素就是所谓的 Bucket，任务保存在双向链表中。**



## 添加定时任务

往 HashedWheelTimer 中添加任务，以下是 HashedWheelTimer#newTimeout 的源码实现：

```java
@Override
public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
        // 前置的参数检查
        ObjectUtil.checkNotNull(task, "task");
        ObjectUtil.checkNotNull(unit, "unit");
    	// 当前等待的任务数+1
        long pendingTimeoutsCount = pendingTimeouts.incrementAndGet();
    	// maxPendingTimeouts 默认为-1，表示无上限
        if (maxPendingTimeouts > 0 && pendingTimeoutsCount > maxPendingTimeouts) {
            pendingTimeouts.decrementAndGet();
            throw new RejectedExecutionException("Number of pending timeouts ("
                                                 + pendingTimeoutsCount + ") is greater than or equal to maximum allowed pending "
                                                 + "timeouts (" + maxPendingTimeouts + ")");
        }
		// 开启时间轮
        start();
    	// 任务执行的时间
        // 不是真实的时间，需要于 startTime 对齐 
        long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;
        // Guard against overflow.
        if (delay > 0 && deadline < 0) {
            deadline = Long.MAX_VALUE;
        }
    	// 包装为 HashedWheelTimeout
        HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
        // 保存到 timeouts
        timeouts.add(timeout);
        return timeout;
}
```

整个添加的过程除了基本的检查就是往 timeouts 中添加。

![image-20211009142149062](assets/image-20211009142149062.png)

timeouts 就是一个简单的任务队列，此时并没有往时间轮中插入任务。

> 具体的插入到桶其实是通过时间轮的工作线程来完成的。
>
> **单线程的进行桶中任务的添加可以完全避免上锁的问题，避免任务调度期间因为插入过于频繁而导致的迟滞。**



## 时间轮的开启流程

HashedWheelTimer 在第一次添加任务的时候开启工作线程（也就启动了时间轮。

以下是 HashedWheelTimer#start 方法实现：

![image-20211009150809514](assets/image-20211009150809514.png)

替换当前 Worker 线程的状态，并启动线程。

主要还是后面的 await，在工作线程启动前阻塞当前线程。

> 不是很明确这里的原因。
>
> 具体的目的就是添加任务的线程需要等待工作线程先启动。



## 时间轮的转动流程

（就是时间轮每次轮转需要执行的逻辑分析。

HashedWheelTimer 中包含一个 Worker 线程，来负责期间的任务调度等逻辑。

Worker 线程中有一个非常关键的变量：

![image-20211009151246382](assets/image-20211009151246382.png)

unprocessedTimeouts 用来存储在关闭 HashedWheelTimer 时来不及执行的任务。

**tick 是一个逻辑时钟周期，每次经过 tickDuration 的时间 tick + 1，时间轮就是根据这个变量确定当前应该从哪个 Bucket 中获取任务。**

以下是 HashedWheelTimer$Worker#run 的实现（Worker 肯定是继承 Runnable 的：

```java
        @Override
        public void run() {
			// 初始化时间轮开始执行的时间
            startTime = System.nanoTime();
            if (startTime == 0) {
                // We use 0 as an indicator for the uninitialized value here, so make sure it's not 0 when initialized.
                startTime = 1;
            }
			// 唤醒在 start() 方法中被阻塞的线程
            // CountDownLatch 先执行 countDown 之后，await 也不会阻塞了
            startTimeInitialized.countDown();
            do {
                    // 睡眠直到下次任务时间到
                    final long deadline = waitForNextTick();
                    // 方法中的定义,deadline为负表示下次任务已经到期
                    if (deadline > 0) {
                        // tick表示当前轮次
                        // 求出应该执行任务的桶下标
                        int idx = (int) (tick & mask);
                        // 清空取消的任务
                        processCancelledTasks();
                        // 取出桶
                        HashedWheelBucket bucket = wheel[idx];
                        // 处理timeouts中的任务，将其分配到具体的桶中
                        transferTimeoutsToBuckets();
                        // 执行bucket中的所有任务
                        bucket.expireTimeouts(deadline);
                        // 时钟周期+1
                        tick++;
                }
                // 判断工作状态来确定好似否
            } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);

            // 到这里就表示 HashedWheelTimer 就被关了
            // Fill the unprocessedTimeouts so we can return them from stop() method.
            // 收集未执行的任务
            for (HashedWheelBucket bucket : wheel) {
                	bucket.clearTimeouts(unprocessedTimeouts);
            }
            for (; ; ) {
                    HashedWheelTimeout timeout = timeouts.poll();
                    if (timeout == null) {
                        break;
                    }
                    if (!timeout.isCancelled()) {
                        unprocessedTimeouts.add(timeout);
                    }
            }
            // 清空取消的任务
            processCancelledTasks();
        }
```

**整个执行周期就是从 timeouts 中搬任务到时间轮，并且执行当前的 Bucket 的到期任务。**

HashedWheelTimer 中任务的执行是单线程的（除非在 TimerTask 中使用线程池去执行），所以执行可能会拖慢下次的任务调度。



### 往时间轮中插入任务

```java
        /**
         * 将timeouts中的任务移动到桶中,最多移动 1000000 个
         */
        private void transferTimeoutsToBuckets() {
                for (int i = 0; i < 100000; i++) {
                        HashedWheelTimeout timeout = timeouts.poll();
                        // 没有任务了
                        if (timeout == null) {
                            break;
                        }
                        // 任务被取消了
                        if (timeout.state() == HashedWheelTimeout.ST_CANCELLED) {
                            continue;
                        }
                        // 任务的 deadline 是任务真实的执行时间和 startTime 对齐的时间
                        // tickDuration 表示的是一个槽位的时间
                        // 所以 calculated 表示的是任务执行的逻辑时钟
                        long calculated = timeout.deadline / tickDuration;
                        // tick 表示当前的逻辑时钟周期
                        // remainingRounds 表示的是时间轮的完整轮次
                        // 当前是第一次，每次转到之后 remainingRounds 减1，到0的时候表明时间到了
                        timeout.remainingRounds = (calculated - tick) / wheel.length;
                        // 如果tick大说明任务执行时间已经过了,所以当前轮次就需要执行
                        // 如果任务的时钟周期比当前的短，则直接在当前的时钟执行
                        final long ticks = Math.max(calculated, tick); // Ensure we don't schedule for past.
                        // 计算插入的桶的下标
                        int stopIndex = (int) (ticks & mask);
                        // 插入到桶中
                        HashedWheelBucket bucket = wheel[stopIndex];
                        bucket.addTimeout(timeout);
                }
        }
```





### 执行到期的任务

以下是 HashedWheelTimer$HashedWheelBucket#expireTimeouts 的方法实现：

```java
/**
 * Expire all {@link HashedWheelTimeout}s for the given {@code deadline}.
 * 执行桶中的到期任务
 */
public void expireTimeouts(long deadline) {
    HashedWheelTimeout timeout = head;
    // 这里需要遍历整个链表
    while (timeout != null) {
        HashedWheelTimeout next = timeout.next;
        // 当前的执行轮次
        if (timeout.remainingRounds <= 0) {
            // 先从链表中删除
            next = remove(timeout);
            if (timeout.deadline <= deadline) {
                    // 执行任务
                    timeout.expire();
            } else {
                // The timeout was placed into a wrong slot. This should never happen.
                throw new IllegalStateException(String.format(
                        "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
            }
        } else if (timeout.isCancelled()) {
            // 任务已经被取消，从链表中删除
            next = remove(timeout);
        } else {
            // 任务需要等待的轮次减1
            timeout.remainingRounds--;
        }
        timeout = next;
    }
}
```