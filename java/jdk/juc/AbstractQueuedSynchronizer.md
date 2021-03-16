

# AbstractQueuedSynchronizer

> - chenqwwq 2021/03

---

[TOC]

---

## 概述

AbstractQueuedSynchronizer 是用于**构造锁和基本同步器**的基础框架，是 JUC 中大部分同步工具的实现基础。

> 很大程度上，AbstractQueueSynchronizer 可以同 Monitor 机制类比，AQS 是通过 Java 语言实现的同步模型，Monitor 是通过 JVM 使用 C语言实现的。

AbstractQueuedSynchronizer内部维护了两种队列结构用来存放相关的等待线程:

- CLH等待队列(同步队列)
- 条件队列

> CLH队列锁是自旋锁的一种简单实现，但可以确保线程无饥饿，也可以保证锁等待的FIFO。

**AbstractQueueSynchronizer 的上锁形式又可以分为独占锁和共享锁。**

## 成员变量

<img src="/home/chen/_note/pic/image-20210309220315479.png" alt="image-20210309220315479" style="zoom:50%;" />

head，tail 分别是等待队列的头，尾节点，而 state 表示的是同步器当前的状态。

state 在 JUC 不同的实现类中有不同的含义:

|     实现类     |   state 的含义   |
| :------------: | :--------------: |
| ReentrantLock  |     重入计数     |
|   Semaphore    |      许可数      |
| CountDownLatch |     初始计数     |
| CyclicBarrier  | 需要等待的线程数 |





## 内部类 - Node

> Node 类就是 CLH 同步队列以及条件队列的节点类，它就是组成两种队列的元素。

```java
static final class Node {  
// 共享模式
static final Node SHARED = new Node();
// 独占模式
static final Node EXCLUSIVE = null;      
```

以特殊的 Node 实例对象表示共享或者独占模式（Node.SHARED | Node.EXCLUSIVE）。

```java
// CANCELLED  表示当前的线程被取消
// SIGNAL     表示当前节点的后继节点(包含的线程)需要运行
// CONDITION  表示当前节点在等待 condition
// PROPAGATE  表示当前场景下后续的 acquireShared 能够得以执行
// 值为0，表示当前节点在 sync 队列中，等待着获取锁
// 后面会有许多状态大于0的判断,就是判断是否取消     
static final int CANCELLED =  1; 
static final int SIGNAL    = -1;
static final int CONDITION = -2;
static final int PROPAGATE = -3;        
// 结点状态
volatile int waitStatus;     
```

waitStatus 表示当前节点的状态，其上就是四种不同的节点状态。

- 同步状态具体如下表所示：

|   状态    |                             含义                             | 数值 |
| :-------: | :----------------------------------------------------------: | :--: |
| CANCELLED |        等待队列中的线程被中断或者超时,会变为取消状态         |  1   |
|  SIGNAL   |  表示该节点的后继节点等待唤醒，在完成该节点后会唤醒后继节点  |  -1  |
| CONDITION | 该节点位于条件等待队列,当其他线程调用了 condition.signal() 方法后会被唤醒进入同步队列 |  -2  |
| PROPAGATE |       共享模式中，表示下次的获取锁资源后应该无条件传播       |  -3  |
| 初始状态  |                          初始化状态                          |  0   |



```java
// 前驱结点
volatile Node prev;    
// 后继结点
volatile Node next; 
// 结点所对应的线程
volatile Thread thread;     
```

在同步队列中会使用前驱和后继节点组成一个双端队列，而 thread 则直接保存对应的线程对象。

```java
// 下一个等待者
Node nextWaiter;
```

nextWaiter 是在条件队列中使用的变量。



> 同步队列和条件队列使用 Node 内部类中不同的字段表示链表节点间的关系。
>
> 同步队列使用 prev 和 next，所以它是个双向队列，而条件队列使用 nextWaiter，所以它是个单向队列。

## 上锁形式

AbstractQueueSynchronizer 支持的获取锁的方式有很多：

|                      方法名                       |             获取形式             |
| :-----------------------------------------------: | :------------------------------: |
|                 acquire(int arg)                  | 获取独占锁，必要时等待，无法中断 |
|           acquireInterruptibly(int arg)           |  获取独占锁，必要时等待，可中断  |
|    tryAcquireNanos(int arg, long nanosTimeout)    |       获取独占锁，超时等待       |
|              acquireShared(int arg)               | 获取共享锁，必要时等待，无法中断 |
|        acquireSharedInterruptibly(int arg)        |  获取共享锁，必要时等待，可中断  |
| tryAcquireSharedNanos(int arg, long nanosTimeout) |       获取共享锁，超时等待       |



## 独占锁

### acquire(int)  - 获取独占锁

```java
// AbstractQueueSynchronizer#acquire
public final void acquire(int arg) {
    // tryAcquire仅为模板函数,需要在子类中实现获取锁的逻辑
    // 获取锁失败之后才会执行&&后面的代码
    if (!tryAcquire(arg) &&	   
        // addWaiter中以独占模式入CLH队列
        // acquireQueued是在入队列之后阻塞线程
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 中断当前线程
        selfInterrupt();	
}

/**
 * Convenience method to interrupt current thread.
 */
static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

> 需要先理解逻辑判断的短路特性，上图如果 tryAcquire(arg) 为 true 就会直接退出方法。

方法首先调用 tryAcquire 尝试快速获取锁，失败时调用 addWaiter 添加到阻塞队列，最后调用 acquireQueued 自旋，必要时阻塞当前线程，异常退出中断当前线程。



#### tryAcquire(int) - 尝试获取锁资源

<img src="/home/chen/_note/pic/image-20210309221820230.png" alt="image-20210309221820230" style="zoom: 67%;" />

以上是 AbstractQueuedSynchronizer#tryAcquire() 的源码**，arg 可以理解为希望获取的资源数，返回 true 即为上锁成功**。

tryAcquire() 是模板方法，具体的实现在各个实现类中，以下是 ReentrantLock$Fair#tryAcquire() 的源码实现:

```java
// ReentrantLock中Fair的tryAcquire实现
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    // 获取当前状态
    int c = getState();
    // 0表示锁空闲状态
    if (c == 0) {
        // 查看是否有前驱节点，公平的体现，如果有前驱会获取锁失败，继续等待
        if (!hasQueuedPredecessors() &&
            // 无前驱只有获取成功
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 以下是重入的部分
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

**在 ReentrantLock 中 AQS 的 state 表示的就是重入次数，**为0时就表示空闲状态并没有上锁，所以一顿操作之后返回 true。

如果返回的 false，就进入等待队列，首先调用的就是 addWaiter 方法。

> ReentrantLock 中 state 表示的是重入次数，所以为0时表示没有线程持有当前锁，所以 CAS 尝试上锁。



#### addWaiter(Node) - 争锁失败节点的首次入队列

以下为 AbstractQueuedSynchronizer#addWaiter 的源码实现：

```java
/**
  * 该方法用于以指定模式添加当前线程到等待队列
  * params： mode -> 共享还是私有模式
  * return:  返回实际入队节点
  */
private Node addWaiter(Node mode) {
    // 创建node实例，包装当前线程入队列
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    // 获得尾节点
    Node pred = tail;
    // 如果尾节点不为空，在此处尝试入队列
    // 尾节点为空表示等待队列未创建，在 enq 中新建
    if (pred != null) {
        // 当前节点的上一个节点指向尾节点
        node.prev = pred;
        // CAS操作，将node置为尾节点，同时只能有一个线程操作成功，其他失败的进入enq方法
        if (compareAndSetTail(pred, node)) {
            // pred下一个节点指向当前节点
            pred.next = node;
            return node;	// 直接返回就不调用enq方法
        }
    }
    // 此处调用enq是在首次入队失败之后
    enq(node);
    return node;
}
```

**addWaiter 方法只做了一次入队列尝试，失败则会调用 enq 完成后续入队列的动作，包括初始化链表节点。**



##### enq  -  节点循环入队

```java
/**
 *  node:   目标入队列的节点
 *  return： 返回入队节点的前驱节点
 */
private Node enq(final Node node) {
    // 自旋 
    for (;;) {
        // 获取尾部节点
        Node t = tail;
        // 尾节点为空时表示队列未初始化，必须创建!!
        if (t == null) { // Must initialize
            // 此处注意!!!是以一个空的节点作为头节点
            if (compareAndSetHead(new Node()))
                // 首尾一致,此时CLH队列中中只有一个元素
                tail = head;
        } else {    // 头节点不为空时
            // 挂在尾节点的后面
            node.prev = t;
            // CAS操作将当前节点置换为尾节点
            if (compareAndSetTail(t, node)) {
                // 尾节点的下一个节点为当前节点
                t.next = node;
                return t;	// 在成功后返回入队节点，否则为无限循环
            }
        }
    }
}
```

> 该方法包含了初始化队列的逻辑，**同步队列以空节点为队头元素。**

该节点确保节点能够正常的被添加到同步队列中。

> 在锁竞争激烈的时候，可能同时有多个线程准备入队列，此时依靠 CAS + 自旋保证了并发安全，只有一个节点可以置换成功。



enq 方法执行之后，当前线程已经进入了等待队列。

#### acquireQueued  -  等待自旋

> 该方法传入的参数 node，必须是已经在同步队列中的节点，arg 则表示希望获取的资源数。
>
> 如果方法正常返回就表示获取成功，返回值为线程当前的中断状态。
>
> 
>
> 该方法可以直接理解为线程在等待队列时候的逻辑。

```java
// AbstractQueueSynchronizer#acquireQueued
final boolean acquireQueued(final Node node, int arg) {
    // 失败标识
    boolean failed = true;
    try {
        // 是否中断标志
        boolean interrupted = false;
        for (;;) {              // 自旋 无限循环 ！
            // 获取前驱节点
            final Node p = node.predecessor();
            // 重点 ！！！ 在前驱节点变为头节点时，才再次尝试获取锁.
            if (p == head && tryAcquire(arg)) {
                // 将该节点置为头结点 表示成功获取到锁
                setHead(node);  
                // 头结点的next为空 帮助GC回收	
                p.next = null; // help GC
                failed = false;
                // 获取成功时返回 
                return interrupted;  
            }
            //  !!! 重点 获取失败之后的准备操作，将前驱节点变为SIGNAL等
            if (shouldParkAfterFailedAcquire(p, node) &&
                // 调用park***方法后会park当前线程，等待唤醒
                parkAndCheckInterrupt()) 
                // 只有线程被中断唤醒时，才会执行该方法
                interrupted = true; 
        }
    } finally {    
        if (failed)
            cancelAcquire(node);
    }
}	
```

**该方法是节点在等待队列中的自旋过程，自旋会大量消耗 CPU，所有该过程中也会夹杂着阻塞线程的逻辑。**

阻塞线程之前会将前驱节点的状态变为 SIGNAL。



##### shouldParkAfterFailedAcquire  -  线程阻塞前的准备逻辑

> 进入该方法表示当前线程的**前驱节点不是头节**点或者**获取锁失败**。

```java
/**
 * params：分别为前驱节点，当前节点
 * returns: 是否准备完成,前驱节点状态为SIGNAL表示准备完成
 */
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;   // 获取前驱节点的状态
    // 前驱节点的状态为Signal
    if (ws == Node.SIGNAL)  
        return true;	
    // 如前面提示>0就表示节点是取消
    if (ws > 0) {
        // 循环遍历找到不是取消状态的节点,取消表示前驱不在尝试获取锁
        // 从后往前遍历找到第一个不是取消的锁,并牌子啊他的后面
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        // 找到前一个不在取消的锁并排在他的后面
        pred.next = node;
    } else {
        // 如果前驱状态不是取消，则改变前驱状态为Signal，即表示本节点需要锁
        // 此处将前驱节点置为SIGNAL,再经过acquireQueued再一次进入时,
        // 会在第一个判断直接返回true
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

在线程阻塞前的准备逻辑,包括:
- 将前驱节点的节点状态变为 SIGNAL。

- 清除状态为 Node.CANCELLED 的无用前驱节点，这里是从后往前遍历的。

在前驱节点为 SIGNAL 的时候直接成功，成功之后回到 acquireQueued的逻辑调用 parkAndCheckInterrupt 方法阻塞线程。



##### parkAndCheckInterrupt  -  直接阻塞线程的方法

> 阻塞线程，并在线程被唤醒时返回中断状态。

```java
/**
 *  阻塞线程,并在被唤醒后返回中断状况
 */
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);		 //调用park()使线程进入waiting状态
    return Thread.interrupted();
}
```

使用 LockSupport#park 直接阻塞当前线程，此时线程就进入了阻塞状态。

阻塞状态被唤醒之后，通过 Thread#interrupted 方法获取了当前线程的终端状态，并且返回。



##### cancelAcquire - 取消获取

> acquireQueued 方法中除非正常的获取锁退出，不然任何异常都会走到这个方法，进行取消操作，出队列。

```java
/**
 *   取消获取锁,acquireQueued方法的fianlly内进入
 *  保证不会是因为错误无限循环 浪费系统资源
 *  params：获取锁失败的节点
 */
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    // 取消状态的节点，可能直接被GC回收了，此时直接退出就好。
    if (node == null)
        return;
    node.thread = null;
    // Skip cancelled predecessors
    Node pred = node.prev;
    // 前驱节点状态为取消时，直接删除前驱节点
    while (pred.waitStatus > 0)			
        node.prev = pred = pred.prev;
    Node predNext = pred.next;	// 获取后继节点.. 是被跳过的节点
    // 修改前驱节点状态为取消
    node.waitStatus = Node.CANCELLED;	
    // node为尾节点，将该节点前驱变为tail，并将前驱节点的next置空
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        int ws;
        // 不为头结点 执行下面判断CANCELLED
        if (pred != head &&	
            // 赋值并判断 不为SIGNAL进行下面判断
            ((ws = pred.waitStatus) == Node.SIGNAL ||	
             // 非取消状态就将状态改为SIGNAL，使用CAS保证并发		
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&	
            // 此判断会将前驱节点全变为SIGNAL状态
            pred.thread != null) {  
            // 当前节点的后继节点
            Node next = node.next;   
            // 后继不为空，且状态不为取消
            if (next != null && next.waitStatus <= 0)	
                // 在同步队列中删除当前节点
                compareAndSetNext(pred, predNext, next);
        } else {
            // 唤醒方法在释放流程中
            unparkSuccessor(node);		
        }

        node.next = node; // help GC
    }
}
```

cancelAcquire 可以看作是放弃锁竞争的流程，流程如下：

1. 往前删除所有取消状态的节点
2. 将当前节点置为取消状态
3. 当前节点是尾节点，将前驱置换成尾节点并置空其后继节点
4. 当前节点是头节点，直接唤醒后继节点
5. 当前节点不是头节点，并且前驱节点已经为 SIGNAL(该状态表示后续节点在等待锁)，如果前驱不是 SIGNAL，就将其置为SIGNAL，然后从队列中删除当前节点(就是拿后继节点置换前驱的后继)。



### 独占锁获取总结

> 整个获取独占锁的的流程

1. 首先调用 tryAcquire 方法尝试直接获取锁资源

2. 获取锁失败之后,先调用 addWaiter 方法以独占方式将锁加入到等待队列末尾

   - addWaiter 中会尝试单次直接入队，失败后调用 enq 自旋入队

3. acquireQueued 负责自旋等待锁，将前驱置为 SIGNAL，并阻塞当前线程

   - 自旋的过程中只有前驱节点为头节点，才会尝试获取锁

   - 使用 **LockSupport 类**阻塞阻塞线程

4. 锁获取失败之后，将节点状态改为 CANCELLED，并从队列中剔除当前节点

5. 阻塞被唤醒之后，如果获取锁成功，则将当前节点置为头节点。



### release(int) - 独占锁的释放

```java
public final boolean release(int arg) {
    // 尝试释放锁 失败进入if代码
    if (tryRelease(arg)) {
        // 获取头节点
        Node h = head;      
        // 如果头节点不为空 且想头节点的状态不是等待获取锁
        if (h != null && h.waitStatus != 0)
            // 唤醒队列中下一个等待的线程
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```



#### tryRelease(int) - 尝试释放锁

<img src="/home/chen/_note/pic/image-20210309225912867.png" alt="image-20210309225912867" style="zoom:67%;" />

同样的，这里也是模板方法，真实逻辑在子类中实现，以下是 tryRelease 在 ReentrantLock#Sync 中的实现：

<img src="/home/chen/_note/pic/image-20210309230030350.png" alt="image-20210309230030350" style="zoom:67%;" />

tryRelease 只是将 releases 加回到 state 中，如果最终的 state 为0，表示资源完全释放，返回 true。

>只有资源完全释放的时候返回 true，在重入锁 ReentrantLock 中，如果重入两次仅释放一次，锁资源也不是完全释放，tryRelease 方法返回的仍然是 false。

#### upparkSuccessor - 唤醒后继有效节点

```java
/**
 *  唤醒等待队列中下一个线程  
 * 		入参: 希望唤醒的节点的前驱节点
 */
private void unparkSuccessor(Node node) {
    ws = node.waitStatus;
    // 判断并置0当前线程状态(取消状态不论)
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    // 获取后继节点
    Node s = node.next;
    // 1. 后继节点为空 或者 后继节点的状态为取消 
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从队尾开始向前找第一个有效节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            // 判断是否是有效节点
            if (t.waitStatus <= 0)    
                s = t;
    }
    // 2. 不为空时直接唤醒后继节点
    if (s != null)
        LockSupport.unpark(s.thread);
}

```

### 独占锁的释放总结

> 独占锁的释放流程：

1. tryRelease 中尝试释放锁资源，通常就是 state 的修改
2. 唤醒队列中等待的后继有效节点(非取消状态的节点)，只会唤醒一个。





## 共享锁

### acquireShared - 获取共享锁

以下是 AQS#acquireShared 的方法源码:

<img src="https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20210309231606765.png" alt="image-20210309231606765" style="zoom:67%;" />

同样的以 tryAcquireShared 这个模板方法尝试获取锁资源。



#### tryAcquireShared - 尝试获取共享锁

<img src="https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20210309231719390.png" alt="image-20210309231719390" style="zoom:67%;" />

> 在独占锁中以 bool 类型表示是否成功获取锁，但在共享锁中，获取锁大于等于0表示锁获取成功。
>
> **更进一步的，tryAcquireShared 返回的是目前的锁资源**。

以下是在 CountDownLatch 中对于 tryAcquireShared 的具体实现:

<img src="/home/chen/_note/pic/image-20210309232239454.png" alt="image-20210309232239454" style="zoom:67%;" />

当前的 state 为0即为获取成功，连修改都没有。

> 对于 CountDownLatch 来说，await 方法相当于上锁，countDown 方法相当于释放锁。
>
> 所以初始化 CountDownLatch 为 n 的时候，getState() 返回不是0，则获取锁失败返回-1，线程进入阻塞队列。



#### doAcquireShared(int) - 共享节点入队列，自旋阻塞

```java
/**
 * Acquires in shared uninterruptible mode.
 * @param arg the acquire argument
 */
private void doAcquireShared(int arg) {
    // 直接以 SHARED 模式入队列
    // addWaiter 中包含了 enq，会有队列的初始化逻辑
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        // 自旋
        for (;;) {
            // 获取前驱节点
            final Node p = node.predecessor();
            // 前驱为头节点
            if (p == head) {
                // 再次尝试获取资源
                int r = tryAcquireShared(arg);
                // 获取成功，则进入退出阶段
                // 在获取失败的时候，头节点的下一个节点无限自旋
                if (r >= 0) {
                    // 将当前节点置为头节点并进行传播
                    setHeadAndPropagate(node, r);
                    // p 已经不再是头节点了，node 才是
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            // 将前驱状态改为 SIGNAL，并阻塞当前节点
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

该方法包含了入队列以及入队列后的自旋操作。

首先是 addWaiter 方法，会将当前节点以 Node.SHARED 的模式入队列。然后通过类似于 acquireQueued 的操作来实现共享锁的自旋。

> 共享锁和独占锁相同的地方，只有在前驱节点为头节点的时候才会尝试去获取锁，获取成功之后还会进行传播。





##### setHeadAndPropagate(Node, int)  -  设置头节点并传播

```java
private void setHeadAndPropagate(Node node, int propagate) {
    // 设置头节点
    Node h = head; // Record old head for check below
    setHead(node);
    // propagate 就是 tryAcquireShared 的返回值
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        // 后继节点为空或者后继为共享节点，则继续释放共享锁
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

> 锁资源传播的首要条件是，**后继节点为空或者后继节点为共享节点**，除此之外需要满足以下条件的其中之一：
>
> 1. 获取的资源数大于0
>
> 1. 旧的头节点为空，或者状态正常
> 2. 新的头节点为空，或者状态正常
>
>
> **Q: 什么情况下，头节点会为空？**



> 这里会出现一个状况，就是同步队列里面如下结构：
>
> ​				读 - 读 - 读 - 写 - 读
>
> 在首节点唤醒并扩散之后，在写节点这里卡住了，而不会扩散到最后的读节点。









###### doReleaseShared - 传播（释放共享锁）

```java
private void doReleaseShared() {
    // 自旋
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }  // 在头节点状态为正常的时候，将其修改为 PROPAGATE 状态。
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        // 头节点未改变就推出
        if (h == head)                   // loop if head changed
            break;
    }
}
```

> 在独占锁中，获取锁成功只会将当前节点置为头节点。
>
> 而在共享锁中，获取锁成功除了将当前节点置为头节点之外，还会扩散。

如果当前节点的状态为 SIGNAL，表示后继有节点需要获取锁，但若当前节点状态为 0，表示后续节点并没有节点需要锁，所以将状态修改为 PROPAGATE 之后，就算释放锁成功。

> 扩散的方式就是唤醒下个节点，下个节点获取成功的话会继续唤醒下个节点。



### 共享锁获取总结

> 共享锁的获取流程如下：

1. 尝试 tryAcquireShared 获取共享锁资源，获取失败同样进入入队列的流程

> 共享锁以 SHARED 模式入队列，流程基本和独占锁类似，入队列之后判断前驱状态，前驱非头节点，在改前驱状态为　SIGNAL 之后，阻塞当前线程。

2. 在获取成功之后，除了将当前节点置为头节点，还会扩散

> 这里的扩散，就是指唤醒下一个有效状态的节点，每次只唤醒后续一个节点，后续节点在 doAcquireShared 中再次尝试获取还成功的话会再向后扩散。







### 共享锁的释放

#### releaseShared(long) - 共享锁的释放

<img src="/home/chen/_note/pic/image-20210310000341515.png" alt="image-20210310000341515" style="zoom:67%;" />

通过 tryReleaseShared  尝试释放共享锁，释放成功之后唤醒后续节点，和独占锁的释放区别并不大。

doReleaseShared 在上述就有介绍。





## 独占锁和共享锁的区别

**独占锁和共享锁的区别主要在获取锁之后的行为，独占锁会将自己置为头节点后直接退出，而共享锁会进行扩散，进一步唤醒后继节点。**





> 共享锁和独占锁在获取失败之后进入的都是阻塞队列，而条件队列是存在 await 方法阻塞的线程。

## 条件队列

> 条件队列是由 AQS 中 ConditionObject 内部类维护的单向队列，区别于 AQS 自身维护的阻塞队列。
>
> 条件队列可以和 Monitor 的等待队列做比较，但是 Monitor 只有一个等待队列，但是 AQS 可以有很多的条件队列。

一般的可以通过 ReentrantLock#newCondition 获取条件队列，方法直接调用了内部类 Sync#newCondition 方法，源码如下：

<img src="/home/chen/_note/pic/image-20210310225756380.png" alt="image-20210310225756380" style="zoom:67%;" />

直接使用的 ConditionObject。

> ConditionObject 本身就是一个完整的同步工具，不需要继承实现模板方法。



### 成员变量

以下为 ConditionObject 的成员变量:

<img src="/home/chen/_note/pic/image-20210310230757243.png" alt="image-20210310230757243" style="zoom:67%;" />

简单的持有了一个队列的队首和队尾元素。





### 等待方法

#### 等待形式

|         方法名         |                   获取形式                   |
| :--------------------: | :------------------------------------------: |
|        await()         |            无限期阻塞，直到被唤醒            |
| awaitUninterruptibly() |          无限期阻塞，并不会响应中断          |
|    awaitNanos(long)    |            阻塞参数时长，单位为ms            |
|    awaitUntil(Date)    |        阻塞直到参数之前，Date表示时间        |
| await(long, TimeUnit)  | 阻塞一定市场，long表示事件，TimeUnit表示单位 |

> 除了 signal 方法，thread.interrupt 方法也能唤醒被 LockSupport#park 阻塞的线程，区别就在于中断标志位。



#### await - 阻塞等待

> 调用 await 阻塞等待方法前，需要先获取到 ConditionObject 所在 AQS 的锁资源。
>
> 并且在阻塞之后，释放对应的 AQS 中，当前线程持有的所有的锁资源。

```java
// ConditionObject#await
public final void await() throws InterruptedException {
    // 当前线程是否中断
    if (Thread.interrupted())
        throw new InterruptedException();
    // 添加当前节点到条件队列
    Node node = addConditionWaiter();
    // await 之后需要完全释放占有的锁资源
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 检查是否在同步队列
    while (!isOnSyncQueue(node)) {
        // 阻塞当前线程
        LockSupport.park(this);
        // 被唤醒之后检查中断状态
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 节点被唤醒，并已经存在同步队列之后，会尝试获取之前释放的所有资源。
    // acquireQueued 是入队列后的自旋操作，如果满足条件会重新被阻塞。
    // acquireQueued 返回的是线程的阻塞状态。
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    // 走之前先清理一遍，此时会把当前节点也清理掉
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    // 中断处理
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

await 方法会将当前线程直接添加到条件队列，并释放所有资源，检查是否持有当前锁资源的逻辑就在释放的过程中。

之后会判断当前线程是否在同步队列，如果不在则会阻塞，唤醒后检查中断标志位，然后继续检查是否在同步队列。

> 在 signal 方法中，唤醒线程的方式就是修改其节点状态并且调用 enq 方法入同步队列。
>
> while 循环起到了防止异常唤醒的作用，如果不在同步队列中继续会被阻塞。

唤醒可以分为 signal 唤醒和 interrupt 唤醒，中断唤醒在检查标志位之后跳出循环。

> 不论哪种唤醒方式，都会进入同步队列再次尝试获取所，只有获取到锁，才能够继续后面的逻辑，包括异常的处理。



> Q: THROW_IE 和 REINTERRUPT 两种中断模式的区别。

THROW_IE 表示中断在 SIGNAL 之前，线程就是由中断唤醒的，而 REINTERRUPT 表示线程中断发生自 SIGNAL 之后，可能是获取到锁的唤醒也可能是中断方法的唤醒。

两种模式的具体处理，在 reportInterruptAfterWait 方法，THROW_IE 会直接抛出中断异常，而 REINTERRUPT 会再次中断当前线程。

这是 await 方法对两种不同时间的中断的响应方式。





##### addConditioonWaiter - 添加到条件队列

```java
// ConditionObject#addConditionWaiter
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    // 如果最后节点不是 CONDITION 状态，开始清理整个条件队列
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // 包装当前节点，CONDITION 状态
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    // 入队列
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

以 CONDITION 状态包装当前线程入队列，整个入队列方法没有任何的并发控制。



###### unlinkCancelledWaiters - 清除取消节点

源码如下：

<img src="/home/chen/_note/pic/image-20210310234153642.png" alt="image-20210310234153642" style="zoom:67%;" />

从队列节点开始一一剔除状态不为 CONDITION 的节点。





##### fullyRelease - 完成释放锁资源

> 在添加到条件队列之后，当前线程就要释放所有持有的锁资源，重入锁需要全部撤销，例如 ReentrantLock 重入3次之后 await，在被唤醒后也需要获取3个资源数。
>
> **在之前添加到条件队列的时候并没有校验是否是锁资源的持有者，而是添加之后通过 fullyRelease 在释放锁的时候校验。**

<img src="/home/chen/_note/pic/image-20210310234113584.png" alt="image-20210310234113584" style="zoom:67%;" />

fullyRelease 最终还是通过 release 方法实现的，一次释放所有持有的锁资源。

release 中会通过 tryRelease 撤销 state 的修改，ReentrantLock#tryRelease 的逻辑在上文可以看到。

> ReentrantLock#tryRelease 会对当前线程进行检查是否是持有锁资源的线程，不是则排除异常，抛出异常之后节点会被修改为 CANCELLED状态。
>
> **等待节点先入队列，再释放锁，若释放失败则为取消状态。**
>
> 最主要的，**该方法会将节点从同步队列中删除，配合上一步的 addConditionWaiter 方法，此时已经完成节点从同步队列到条件队列的转移。**





##### isOnSyncQueue  - 判断节点是否在同步队列

> Q： 为什么需要判断节点是否在同步队列？
>
> 在 signal 的逻辑中可以看到，节点在被唤醒后会通过 enq 的方法添加到队列中。

<img src="/home/chen/_note/pic/image-20210310235214860.png" alt="image-20210310235214860" style="zoom:67%;" />

有两个简化判断：

1. 节点没有前驱节点，肯定不在同步队列

> 在同步队列中，前驱节点为一个空节点，任何在同步队列中的节点肯定都有前驱节点。

2. 节点存在后继节点，肯定在同步队列

> 条件队列是单向队列，通过 nextWaiter 维护节点关系，同步队列是双向队列通过 prev 和 next 维护节点关系。
>
> 所以在条件队列时，肯定不会存在后继节点。





###### findNodeFromTail - 从后往前查找节点

<img src="/home/chen/_note/pic/image-20210310235322334.png" alt="image-20210310235322334" style="zoom:67%;" />

从 tail 节点开始的遍历，到头节点后就结束。





##### checkInterruptWhileWaiting - 中断检查

<img src="/home/chen/_note/pic/image-20210311112558047.png" alt="image-20210311112558047" style="zoom:67%;" />

方法的注释如下：

> 检查中断，**如果返回 THROW_IE 则中断发生在 signal 之前，如果返回 REINTERRUPT 则中断发生在 signal 之后。**
>
> signal 发生指的是状态的修改，如果连状态都没改变仍然为 CONDITION，那么就是在 signal 发生前改变的。
>
> 具体可以看 transferAfterCancelledWait 方法。

Thread#interupted 方法会在返回当前线程的中断标志位之后修改中断标志，**如果是中断导致的退出，则进入到 transrferAfterCancelledWait 方法。**





###### transferAfterCancelledWait - 转移节点

> 正常的被 signal 唤醒时，线程会通在 signal 线程中被转移到同步队列，中断唤醒缺少这一流程。

<img src="/home/chen/_note/pic/image-20210311113200910.png" alt="image-20210311113200910" style="zoom:67%;" />

中断唤醒之后，尝试修改其 CONDITION 状态为0，修改成功之后入队列，并返回 true，修改失败之后，只要节点不在同步队列就让出CPU。







###### reportInterruptAfterWait - 中断标志位的处理

<img src="/home/chen/_note/pic/image-20210312003019436.png" alt="image-20210312003019436" style="zoom:67%;" />

THROW_IE 和 REINTERRUPT 就是 AQS 条件队列的异常处理模式。



#### 阻塞逻辑总结

调用 await 方法之后不需要检查锁持有状态，优先尾插法入队列，之后完全释放持有的锁资源，并进入阻塞状态。

从条件队列的阻塞状态中脱离，可以是 signal 也可以是中断。





#### signal - 唤醒阻塞线程

<img src="/home/chen/_note/pic/image-20210311000007729.png" alt="image-20210311000007729" style="zoom:67%;" />

signal 会先检查当前线程是否是 AQS 锁资源的持有者，不是就抛出异常，然后从头节点开始唤醒被挂起的线程。



##### doSignal 

<img src="/home/chen/_note/pic/image-20210311000243871.png" alt="image-20210311000243871" style="zoom:67%;" />

首先先剔除了从等待队列剔除了 firstWaiter 节点，如果 first 节点状态不为 CONDITION 时，继续尝试唤醒下一个节点。

> transferForSignal 在 node 节点不为 CONDITION 状态时才会返回 false。



###### transferForSignal -入队列并修改前驱节点

```java
// AQS#transferForSignal
final boolean transferForSignal(Node node) {
    // 修改节点状态，修改失败 false 退出
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    // enq 就是循环入队列的逻辑，返回的是前驱节点
    Node p = enq(node);
    int ws = p.waitStatus;
    // 前驱节点状态为取消 或者 不为取消时，替换前驱节点状态为 SIGNAL 失败时，唤醒node节点
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```



#### 唤醒逻辑总结

> signal 方法调用必须先获取对应的 AQS 锁资源。

> signal 方法会唤醒从条件队列队首开始的第一个有效节点，并且修改该节点状态，并将该节点入队列。













> 条件队列，多条件队列是 AQS 实现阻塞队列的基础。



## AbstractOwnableSynchonizer

```java 
 /**
   * A synchronizer that may be exclusively owned by a thread.  a
   * This class provides a basis for creating locks and related synchronizers 
   * that may entail a notion of ownership. 
   */
```
该类主要定义了线程独占的方式拥有的同步器，提供了创建锁和相关同步器的基础，并且可能会涉及到所有权的概念。

AbstractOwnableSynchronizer 的全部源码如下:

```java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {
    private transient Thread exclusiveOwnerThread;      // 独占线程
    // 设置当前拥有独占访问权的线程
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }
    // 获取当前拥有独占访问权的线程
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}
```

## AQS 相关

> 待总结。

### AQS 中断相关问题

### PROPAGATE 状态的含义

