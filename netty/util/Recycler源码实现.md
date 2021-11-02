# Recycler

---

[TOC]

---



## 概述

对象池和连接池之类的一样都是享元模式的实现，减少对象的重复创建和回收，减少 GC 压力。

Netty 针对于对象池有一个轻量级的实现 ObejctPool，其基本实现就是 Recycler。

> 除了 Netty，Apache Common Pool2 也是一种常用的对象池实现。
>
> 和 Recycler 不同，Apache 的对象池采用的是阻塞队列来保证并发安全（底层还是 AQS 那一套。

基本使用如下：

```java
public class Main {
    static class Obj {
        static AtomicInteger idGen = new AtomicInteger();
        int id;
        ObjectPool.Handle<Obj> handle;

        public Obj(ObjectPool.Handle<Obj> handle) {
            this.id = Obj.idGen.incrementAndGet();
            this.handle = handle;
        }

        @Override
        public String toString() {
            return "Obj{" +
                "id=" + id +
                '}';
        }

        // 这个方法实现有点膈应
        // 需要通过 handle 来让对象重新回到池子中
        public void recycle() {
            handle.recycle(this);
        }
    }

    public static void main(String[] args) {
        ObjectPool<Obj> objPool = ObjectPool.newPool(new ObjectPool.ObjectCreator<Obj>() {
            @Override
            public Obj newObject(ObjectPool.Handle<Obj> handle) {
                return new Obj(handle);
            }
        });

        final Obj o1 = objPool.get();
        o1.recycle();
        final Obj o2 = objPool.get();
        System.out.println(o1);
        System.out.println(o2);
        System.out.println(o1 == o2);
    }
}

// 输出如下:
// Obj{id=1}
// Obj{id=1}
// 	true
```

使用的时候不推荐直接实现 Recycler，而是采用 ObjectPool.newPool() 去创建。

> 对象的回收方法需要通过 Handle。



### 对于对象池的简单思考

对象池和线程池一样，由**池**这个概念持有全部的对象，对象池持有所有的对象，线程池持有所有的线程，在使用时从池子中获取，使用完毕归还给池子。（所以对于对象池的接口，甚至可以只有简单的 Get / Put 方法。

除了获取和归还的方法之外，还有**对象何时创建以及多线程并发的问题。**

对于何时创建，一般来说是在获取不到的时候，或者创建池子的时候先初始化 n 个对象供急用（例如线程池可以实现创建 coreSize 个线程）。

和线程池类似，对象池也有一个最大容量的上限，或者在增加一个核心容量。

多线程并发的问题，简单一点可以使用 synchronized 直接保证，线程池中使用了 BlockQueue，底层相当于是使用 AQS 保证的并发安全。

对于池中的对象来说，没有必要标记它属于哪个池子，获取之后应该删除所有池子对于该对象的引用，如果不归还对象，则由 GC 带走。（**个人感觉，对于已经被获取的对象，池子不应该对其生命周期做任何限制。**

## ObjectPool 

Netty 实现的 ObjectPool 中定义了如下三种对象：

1. ObjectPool - 最上层的对象池，负责持有所有对象并对获取方法进行调度
2. ObjectCreator - 对象创建
3. Handle - 持有对象的引用，也负责对对象进行回收

![image-20211022143954289](assets/image-20211022143954289.png)

并且默认使用 Recycler 作为基础实现的 RecyclerObjectPool 类：

![image-20211022144031115](assets/image-20211022144031115.png)

RecyclerObjectPool 直接创建了 Recycle 作为其内部工具，构造函数中传入一个 ObjectCreator 就可以创建一个对象池。

<br>

以下从获取和归还两个角度分析 Recycler 的具体实现。

<br>

## Recycler#get 获取对象

以下是 Recycler#get 的方法源码：

```javascript
@SuppressWarnings("unchecked")
public final T get() {
    // 配置是否为空池
    if (maxCapacityPerThread == 0) {
        return newObject((Handle<T>) NOOP_HANDLE);
    }
    // 由 FastThreadLocal 持有 Stack 对象
    // 每个线程一个 Stack，直接快进到 initValue 方法
    Stack<T> stack = threadLocal.get();
    // 从中弹出一个对象
    DefaultHandle<T> handle = stack.pop();
    // 没有返回一个对象
    if (handle == null) {
        // 重新创建
        handle = stack.newHandle();
        handle.value = newObject(handle);
    }
    // Handle 好像是为了持有该值
    return (T) handle.value;
}

```

对象的保存并不是在一个统一的集合里面，而是通过 FastThreadLocal 使每个线程都持有自己的对象。

> ThreadLocal 就是空间换时间，减少锁的消耗。

Stack 并不是 JDK 中的实现，而是 Recycler 中通过数组模拟的，获取到 Stack 之后，就尝试弹出（获取）对象，获取失败则创建。

<br>

### Stack#pop 弹出对象

```java
DefaultHandle<T> pop() {
   // size 表示当前池子中对象的个数
    int size = this.size;
    // 当前并没有对象未被使用
    if (size == 0) {
        // 尝试回收对象
        if (!scavenge()) {
            return null;
        }
        size = this.size;
        // 尝试回收之后仍然小于0，表示实在没有对象了
        if (size <= 0) {
            // double check, avoid races
            return null;
        }
    }
    // 分配空余对象
    // 这里必须先 size --
    // size -- 就已经占据了一个对象，如果先获取对象，那么对象可能会被多次获取
    size--;
    // 分配最后一个对象
    DefaultHandle ret = elements[size];
    elements[size] = null;
    // 修改当前对象数目
    this.size = size;
    
    // lastRecycledId 表示归属
    if (ret.lastRecycledId != ret.recycleId) {
        throw new IllegalStateException("recycled multiple times");
    }
    ret.recycleId = 0;
    ret.lastRecycledId = 0;
    return ret;
}
```

弹出对象就是从底层数组中获取对象，在获取之前如果发现没有对象，尝试从 WeakOrderQueue 中回收一波。

<br>

### Stack#scavengeSome 收集对象

**Recycler 的无锁化设计重点之一就是和 Stack 所绑定线程不同的线程归还对象时，并不能直接添加到底层的数组，而是添加到 Stack 的 WeakOrderQueue 链表中（下文会说到，剧透下。**

以下是 Stack#scavengeSome 的源码实现：

```java
/**
* 从 {@link WeakOrderQueue} 中回收部分数据
*/
private boolean scavengeSome() {
    WeakOrderQueue prev;
    // 设置当前游标
    WeakOrderQueue cursor = this.cursor;
    if (cursor == null) {
        prev = null;
        cursor = head;
        if (cursor == null) {
            return false;
        }
    } else {
        prev = this.prev;
    }

    boolean success = false;
    do {
        if (cursor.transfer(this)) {
            success = true;
            break;
        }
        WeakOrderQueue next = cursor.getNext();
        if (cursor.get() == null) {
            // If the thread associated with the queue is gone, unlink it, after
            // performing a volatile read to confirm there is no data left to collect.
            // We never unlink the first queue, as we don't want to synchronize on updating the head.
            if (cursor.hasFinalData()) {
                for (; ; ) {
                    if (cursor.transfer(this)) {
                        success = true;
                    } else {
                        break;
                    }
                }
            }

            if (prev != null) {
                // Ensure we reclaim all space before dropping the WeakOrderQueue to be GC'ed.
                cursor.reclaimAllSpaceAndUnlink();
                prev.setNext(next);
            }
        } else {
            prev = cursor;
        }

        cursor = next;

    } while (cursor != null && !success);

    this.prev = prev;
    this.cursor = cursor;
    return success;
}
```

就是从 WeakOrderQueue 组成的链表中回收对象。



## 归还对象

**归还对象是需要区分对象绑定的 Stack 是否由当前线程负责。**

<br>

以下是 DefaultHandle#recycle 的源码实现:

```java
/**
* 释放对象使用
*/
@Override
public void recycle(Object object) {
    if (object != value) {
        throw new IllegalArgumentException("object does not belong to handle");
    }
    // 
    // 获取对象从属的 Stack
    Stack<?> stack = this.stack;
    if (lastRecycledId != recycleId || stack == null) {
        throw new IllegalStateException("recycled already");
    }
    // 将对象重新塞入 Stack
    stack.push(this);
}
```

从 Handle 中取出绑定的 Stack，之后就是通过 push 归还对象，以下是 Stack#push 的实现:

```java
void push(DefaultHandle<?> item) {
    // 获取当前线程
    Thread currentThread = Thread.currentThread();
    // 判断是否是当前线程
    // important: 可以存在一个线程取出之后给别的线程使用的情况
    if (threadRef.get() == currentThread) {
        // The current Thread is the thread that belongs to the Stack, we can try to push the object now.
        pushNow(item);
    } else {
        // The current Thread is not the one that belongs to the Stack
        // (or the Thread that belonged to the Stack was collected already), we need to signal that the push
        // happens later.
        pushLater(item, currentThread);
    }
}
```

**根据是否为当前线程绑定的 Stack，区分立即归还和延迟归还。**

立即归还的实现如下:

```java
private void pushNow(DefaultHandle<?> item) {
    if ((item.recycleId | item.lastRecycledId) != 0) {
        throw new IllegalStateException("recycled already");
    }

    // 配置归属的线程Id
    item.recycleId = item.lastRecycledId = OWN_THREAD_ID;

    int size = this.size;
    // 对象池满了
    // 在 size 小于 maxCapacity 的时候，尝试 dropHandle
    if (size >= maxCapacity || dropHandle(item)) {
        // Hit the maximum capacity or should drop - drop the possibly youngest object.
        return;
    }
    // 到这里之后,size 肯定小于 maxCapacity
    // 存放对象的数组满了,就扩容
    if (size == elements.length) {
        size <<= 1;
        if (size > maxCapacity) {
            elements = Arrays.copyOf(elements, min(size << 1, maxCapacity));
        }
    }
    // 放入数组
    elements[size] = item;
    this.size = size + 1;
}
```

如果池子满了直接淘汰对象，如果池子没满则将对象塞入底层数组，数组长度不够则进行扩容。

**其中 dropHandle 方法是用来确定对象回收的比例，在 Netty 的实现中并不是所有的对象都会被重复使用，会通过一定的比例淘汰掉一部分的对象。**

再来是延迟归还的情况：

```java
private void pushLater(DefaultHandle<?> item, Thread thread) {
    if (maxDelayedQueues == 0) {
        // We don't support recycling across threads and should just drop the item on the floor.
        return;
    }

    // we don't want to have a ref to the queue as the value in our weak map
    // so we null it out; to ensure there are no races with restoring it later
    // we impose a memory ordering here (no-op on x86)
    // 每个线程还绑定了一个 Stack -> WeakOrderQueue 的映射
    // 每个线程独立的 Map
    Map<Stack<?>, WeakOrderQueue> delayedRecycled = DELAYED_RECYCLED.get();
    // 每个线程往同一个 Stack 延迟归还的时候，都是放到不同的 WeakOrderQueue 中。
    WeakOrderQueue queue = delayedRecycled.get(this);
    if (queue == null) {
        if (delayedRecycled.size() >= maxDelayedQueues) {
            // Add a dummy queue so we know we should drop the object
            // 在下面就可以看到 如果是 DUMMY 队列的话会直接抛弃
            delayedRecycled.put(this, WeakOrderQueue.DUMMY);
            return;
        }
        // Check if we already reached the maximum number of delayed queues and if we can allocate at all.
        if ((queue = newWeakOrderQueue(thread)) == null) {
            // drop object
            return;
        }
        delayedRecycled.put(this, queue);
    } else if (queue == WeakOrderQueue.DUMMY) {
        // drop object
        return;
    }

    queue.add(item);
}
```

> 延迟归还发生在归还线程并不是对象归属线程的时候。

此时并不是存放到线程对应的 Stack 中，而是当前线程的 WeakOrderQueue 中（使用目标线程的 Stack 绑定。



## 随记

Stack 是一个用数组模拟的栈，每一个线程都会绑定一个 Stack 对象，使用 FastThreadLocal 作绑定。

所有的对象都被包装为 DefaultHandle，DefaultHandle 是 Recycler 的内部类，它的 recycle 方法实现了对象重入 Stack 的逻辑。

理论上，因为用线程做了对象的区分，所以应该由获取的线程去释放对象的使用，但是可能会出现引用逃逸的情况，当前线程可能将对象传递给别的线程，此时释放就需要由别的线程完成。

多线程归还对象（将对象添加到 Stack 底层的数组中，此时就需要上锁保证其并发安全性。

Netty 做了无锁化处理，即只有 Stack 的线程才可以处理 Stack 中的数组（添加或者删除。

所以由别的线程归还对象时，并不是直接归还到该 Stack 中，而是保存在 WeakOrderQueue 中，使用另外一个 FastThreadLocal 保存了 Stack 和 WeakOrderQueue（持有着对象）的映射关系。

一个 Stack 中不同的 Thread 会创建不同的 WeakOrderQueue，然后添加当前线程所释放的对象。





> 获取对象的流程：

1. 获取当前线程绑定的 Stack（通过 FastThreadLocal
2. 获取 Stack 数组中的对象，如果不存在则尝试从 WeakOrderQueue 中回收（由其他线程所释放的对象，WeakOrderQueue 中使用 Link 保存对象
3. 如果还没有获取到对象，则直接使用 newObject 创建对象（该方法需要自定义



> 对象释放的流程：（对象的释放由于 DefaultHandle 触发，在创建的时候就会传递一个 Handle  进参数，但是真实的逻辑是在 Stack 中

1. 判断当前线程是否是 Stack 绑定的 Thread
2. 如果是则直接添加到数组（中间还有扩容以及部分丢弃的逻辑
3. 如果不是则尝试添加到当前 Stack 的 WeakOrderQueue



## 参考

- [netty源码学习笔记——对象池](https://huzb.me/2019/10/17/netty%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E2%80%94%E2%80%94%E5%AF%B9%E8%B1%A1%E6%B1%A0/)

- [Netty源码之对象池](https://juejin.cn/post/6922779981101662222#heading-15)