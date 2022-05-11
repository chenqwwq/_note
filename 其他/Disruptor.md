# Disruptor

---

[TOC]

---

## Introduction

Disruptor 是 LMAX 开源的内存队列，相对于 JDK 的 BlockingQueue，Disruptor 有更好的性能表现以及更多的消费模式。

Disruptor 有以下几种优化：

1. 对象池（Disruptor 所有的事件都通过 RingBuffer 保存，RingBuffer 就是一个对象池，所有的数据都需要塞到对应的对象池中
2. 填充缓存行，消除伪共享（伪共享是多个 CPU 的缓存行中包含同一段数据，双方各自的修改都会使缓存行失效而重新从主内存中读取，



## Disruptor 的组件

### RingBuffer 

Disruptor 的存储组件，临时保存产生的事件，使用环形数组保存所有数据，RingBuffer 底层就是一个数组，维护了写入和读取两个游标，以此形成一个环。



### Sequence 

（相当于是一个并发安全的 Long 类型。

表示消费的序列，在 RingBuffer（生产者和消费者都是通过 RingBuffer 获取获取） 会保存该值。

对于生产者来说，Sequence 保存了写入的位置，对于消费者来说，Sequence 保存在自身的读取的位置（多个消费者不共享。



### Sequencer

Sequence 的管理者，包含了生产者和消费者的（就是持有了生产者的写入指针和消费者的读取指针。

根据生产者的个数分为 SingleProducerSequencer 和 MutilProducerSequencer。



### EventFactory

Event 就是 RingBuffer 中保存的数据类型，EventFactory 就是用于创建这些实例，在创建 RingBuffer 的时候传入， 初始化的时候预创建所有的对象。



### WaitStrategy

等待策略，表示消费者在等待事件发布的时候采取的动作（生产者在等待消费的时候也可以使用吧。





## 消费者（监听器）注册流程

```java
// Disruptor#handleEventsWith
public final EventHandlerGroup<T> handleEventsWith(final EventHandler<? super T>... handlers){
  // 直接创建 EventProcessor
  return createEventProcessors(new Sequence[0], handlers);
}

// Disruptor#createEventProcessors
// 创建消费者实例
// 参数包含 barrierSequence，是他依赖的上层消费者，当前消费者只能消费上层已经全部消费过的数据
// 例如，当前依赖的三个上层消费者的 offset [1,10,10]，那么此时只能消费 1 的数据
EventHandlerGroup<T> createEventProcessors(final Sequence[] barrierSequences,final EventHandler<? super T>[] eventHandlers){
  // 只能在未注册的时候添加消费者
  checkNotStarted();
	// 每个 Handler 对应一个 Sequence
  final Sequence[] processorSequences = new Sequence[eventHandlers.length];
  // 创建 Barrier
  final SequenceBarrier barrier = ringBuffer.newBarrier(barrierSequences);
	// 遍历创建 BatchEventProcessor
  for (int i = 0, eventHandlersLength = eventHandlers.length; i < eventHandlersLength; i++){
    final EventHandler<? super T> eventHandler = eventHandlers[i];
    // ！！important  创建的最终消费实例好似 BatchEventProcessor
    final BatchEventProcessor<T> batchEventProcessor = new BatchEventProcessor<>(ringBuffer, barrier, eventHandler);
    // 异常处理，这个是 Disruptor 确定的
    if (exceptionHandler != null){
      batchEventProcessor.setExceptionHandler(exceptionHandler);
    }
    // 消费者注册中心？
    consumerRepository.add(batchEventProcessor, eventHandler, barrier);
    // 消费者的 Processor
    processorSequences[i] = batchEventProcessor.getSequence();
  }
  // 更新 Disruptor 的 GatingSequences
  updateGatingSequencesForNextInChain(barrierSequences, processorSequences);
  // 返回一个 EventhandlerGroup，调用当前方法的所有 EventHandler 会被包含在一个 Group 里面
  return new EventHandlerGroup<>(this, consumerRepository, processorSequences);
}

// Disruptor#updateGatingSequencesForNextInChain
// 更新 GatingSequences
private void updateGatingSequencesForNextInChain(final Sequence[] barrierSequences, final Sequence[] processorSequences){
  if (processorSequences.length > 0){
    // 将消费者的 Sequences 添加到 ringBuffer
    ringBuffer.addGatingSequences(processorSequences);
    // 因为当前消费者消费的都是 barrierSequences 中消费过的数据，所以当前消费者肯定是最低的 offset
    // 因此依赖的 barrier 就没必要保存了
    for (final Sequence barrierSequence : barrierSequences){
      ringBuffer.removeGatingSequence(barrierSequence);
    }
    consumerRepository.unMarkEventProcessorsAsEndOfChain(barrierSequences);
  }
}
```

消费者最终的实例对象为 BatchEventProcessor。

每个消费者都会将自身的 Sequence 添加到 ringBuffer（参考 kafka 的 offset，ringBuffer 中只有所有的消费者都消费过才会将数据清除。

Disruptor 通过 ConsumerRepository 来保存所有的 BatchEventProcessor（。





另外可以看一下 ConsumerRepository 的功能，ConsumerRepository 用于持有所有的 EventHandler 以及对应的 Sequence 信息（算是一个辅助类，用于完成一些相对独立的逻辑。

ConsumerRepository 中包含以下三个成员变量，分别从不同用维度保存映射关系。

1. eventProcessorInfoBySequence - Map<Sequence, ConsumerInfo>
2. eventProcessorInfoByEventHandler - Map<EventHandler<?>, EventProcessorInfo<T>>
3. consumerInfos - Collection<ConsumerInfo>

```java
public void add(
  final EventProcessor eventprocessor, 
  final EventHandler<? super T> handler,
  final SequenceBarrier barrier){
    // 将 EventProcessor，EventHandler，Barrier 打包成一个 EventProcessorInfo
    final EventProcessorInfo<T> consumerInfo = new EventProcessorInfo<>(eventprocessor, handler, barrier);
    // 分别存了三份！！！索引加强吗？？
    eventProcessorInfoByEventHandler.put(handler, consumerInfo);
    eventProcessorInfoBySequence.put(eventprocessor.getSequence(), consumerInfo);
    consumerInfos.add(consumerInfo);
}
```







## 事件生产流程

RingBuffer 回预先创建所有的事件对象，所以后续的发布流程就是获取指定对象，填充属性并且发布。



### 获取可写入位置

RingBuffer 是一个环形数组，所以就需要读写两个指针表示进度（emm，因为先学的 InnoDb 的 Redo Log 所以感觉 write point 和 checkpoint 两个还挺好理解的。

**因为是个一对多或者多对多的场景，并且消费者各自保存自己的消费进度，所以写入的场景都需要考虑多个消费者的最低消费速度，而写入的速度保存在 RingBuffer**

RingBuffer 中使用 Sequencer#next 确定下一个写入的位置，Sequencer  都继承了 AbstractSequencer，因此也保存了以下几个变量：

```java
protected final int bufferSize;		// 	RingBuffer 的大小
protected final WaitStrategy waitStrategy;    // 等待策略
protected final Sequence cursor = new Sequence(Sequencer.INITIAL_CURSOR_VALUE);   // 当前写入的位置
protected volatile Sequence[] gatingSequences = new Sequence[0];		// gatingSequences 就是所有消费者的 Sequence
// 在注册流程中会将消费者的 Sequence 全部添加进来
```

<br>

根据消费者的个数可以分为 SingleProducerSequencer 以及 MultiProducerSequencer。

（SingleProduceSequencer 中不存在争用，所以实现相对简单。

<br>

**SingleProducerSequencer** 

```java
@Override
public long next(int n)
{
  if (n < 1){
    throw new IllegalArgumentException("n must be > 0");
  }

  // 下次的可用槽位
  long nextValue = this.nextValue;
	// 一共需要n个槽位
  long nextSequence = nextValue + n;
  // 可能会超过，一般情况下都是负数，超过表示需要重头开始填充
  // 比如当前 bufferSize = 16，但是 nextSequence = 19，wrapPoint = 3 表示已经过了一轮了
  long wrapPoint = nextSequence - bufferSize;
  // cacheValue 是所有消费者的最小的下标（最慢的速度
  long cachedGatingSequence = this.cachedValue;

  // 主要是 cacheValue = 5，但是 wrapPoint = 6 的时候，此时说明生产的速度已经要大于消费的速度了
  // 所以需要进循环等待
  if (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue) {
    cursor.setVolatile(nextValue);  // StoreLoad fence
    long minSequence;
    // 等待直到消费赶上
    while (wrapPoint > (minSequence = Util.getMinimumSequence(gatingSequences, nextValue))){
      LockSupport.parkNanos(1L); // TODO: Use waitStrategy to spin?
    }
	  // 重新设置最小消费槽位
    this.cachedValue = minSequence;
  }
	// 下次的消费槽位
  this.nextValue = nextSequence;
  return nextSequence;
}
```

单线程并没有并发和伪共享问题，所以直接使用的一个 long 类型的 nextValue 保存写指针（读指针保存在各个消费者那，会在 gatingSequences 中保存。

**MultiProducerSequencer**

MultiProducerSequencer 表示多个生产者，所以会存在写入位置并发写入的问题，该类中使用 CAS 实现并发更新的安全性。

```java
// MultiProducerSequencer#next
public long next(int n){
  if (n < 1){
    throw new IllegalArgumentException("n must be > 0");
  }
  long current;
  long next;
  do{
    // cursor 是 Sequence 实现
    // 不再和 Single 一样单纯的 long 了
    current = cursor.get();
    // 后续的判断都差不多
    next = current + n;
    long wrapPoint = next - bufferSize;
    long cachedGatingSequence = gatingSequenceCache.get();
    if (wrapPoint > cachedGatingSequence || cachedGatingSequence > current){
      long gatingSequence = Util.getMinimumSequence(gatingSequences, current);
      if (wrapPoint > gatingSequence){
        LockSupport.parkNanos(1); // TODO, should we spin based on the wait strategy?
        continue;
      }
      gatingSequenceCache.set(gatingSequence);
      // 更新的时候使用的 CSA，外层套了个自旋
    }else if (cursor.compareAndSet(current, next))]{
      break;
    }
  }
  while (true);
  return next;
}
```



如果是一个无限长的序列，可写入的位置必须小于多生产者的最小为止，因为是环形数组所以需要额外判断超出的情况。

ProducerSequencer 中保存了所有消费者的 Sequence，每次都会获取最小的消费下标，写入不能超过这个下标，在 Sequencer 中另外有缓存避免每次遍历消费者（感觉多此一举，消费者能有多少。

### 获取指定下标对象

```java
// RingBuffer#get
public E get(long sequence){
  return elementAt(sequence);
}

// RingBufferFields#elementAt
protected final E elementAt(long sequence){
  return (E) UNSAFE.getObject(entries, REF_ARRAY_BASE + ((sequence & indexMask) << REF_ELEMENT_SHIFT));
}
```

通过 Unsafe 获取数组对象对象指定下标的值。

因为 indexMask 是2此幂，所以 sequence & indexMask 相当于是 sequence % indexMask 了（真就大量优化。



### 发布事件

发布事件修改的是生产者的写入指针，对于 SingleProducerSequencer 来说就是简单的修改 cursor。

```java
// SingleProducerSequencer#pushlish
public void publish(long sequence){
  // 修改 cursor
  cursor.set(sequence);
  // 唤醒所有阻塞的消费者
  waitStrategy.signalAllWhenBlocking();
}
```

主要是多线程生产的时候，因为整个发布顺序是先获取可写位置，赋值，发布，所以后获取的位置（比较靠后）可能先发布，直接替换就不成立了。

MultiProducerSequencer 使用的 availableBuffer 表示每个槽位是否可读。

```java
// 长度和 RingBuffer 相同，表示每个位置的版本号，每次修改某个位置，该位置的 flag + 1
private final int[] availableBuffer;
// availableBuffer 保存的都是每个 sequence >>> indexShift
private final int indexShift;
```

MultiProducerSequencer 的发布就是修改每个位置上的 flag:

```java
// MultiProducerSequencer#publish
public void publish(final long sequence){
  setAvailable(sequence);
  waitStrategy.signalAllWhenBlocking();
}
// MultiProducerSequencer#setAvailable
private void setAvailable(final long sequence){
  setAvailableBufferValue(calculateIndex(sequence), calculateAvailabilityFlag(sequence));
}

// MultiProducerSequencer#setAvailableBufferValue
private void setAvailableBufferValue(int index, int flag){
  long bufferAddress = (index * SCALE) + BASE;
  UNSAFE.putOrderedInt(availableBuffer, bufferAddress, flag);
}

// MultiProducerSequencer#calculateIndex
// indexMask 可以参考 HashMap，以 & 来实现取余
private int calculateIndex(final long sequence){
  return ((int) sequence) & indexMask;
}

// MultiProducerSequencer#calculateAvailabilityFlag
// 根据 sequence 确定 flag，sequence 增加 bufferSize 的时候 flag 才会+1
// bufferSize = 16 的时候，indexShift = 4，懂？
private int calculateAvailabilityFlag(final long sequence){
  return (int) (sequence >>> indexShift);
}
```

简单的可以理解为 avaliableBuffer 中保存的是每个位置的版本号（从0开始的，因此可以根据 Flag 判断是否可读。

flag 也不需要单独保存，可以根据 sequence 计算。



## 事件消费流程

在消费者注册流程中可以看出来，Disruptor 的事件消费实例是 BatchEventProcessor，消费逻辑当然也在里面。

（BatchEventProcessor 继承了 Runnable，所以可以直接看 run 方法。



```java
// BatchEventProcessor#run
public void run(){
  // CAS 更新运行状态
  if (running.compareAndSet(IDLE, RUNNING)){
    // 清除 Alter，这里应该是有依赖关系的消费者
    sequenceBarrier.clearAlert();
    // 调用 LifecycleAware#onStart 的钩子方法
    notifyStart();
    try
    {
      // 确定状态
      if (running.get() == RUNNING){
        // 处理事件（外层没 while 应该是包在里面了
        processEvents();
      }
    } finally {
      // 调用钩子方法
      notifyShutdown();
      // 关闭状态
      running.set(IDLE);
    }
  } else{
    // CAS 失败表示已经开启过了
    if (running.get() == RUNNING) {
      throw new IllegalStateException("Thread is already running");
    } else{
      // 提前退出
      earlyExit();
    }
  }
}

// 已经开启过了，在开启的时候如果发现不是 RUNNING 状态，先调用 Start 钩子，在调用 shutdown 钩子？？？？？？？？ 
private void earlyExit(){
  notifyStart();
  notifyShutdown();
}
```

开始消费的时候会在前后调用钩子方法，Handler 的状态比较简单就省略吧，直接看消费的核心 processEvent 方法：

```java
// BatchEventProcessor#processEvent
private void processEvents(){
  T event = null;
  long nextSequence = sequence.get() + 1L;

  while (true){
    try{
      // 等待并获取可用的下标，返回的是可用的，传参表示期望的，返回的可能更大
      final long availableSequence = sequenceBarrier.waitFor(nextSequence);
      // 没有实现，看着好像是钩子方法
      if (batchStartAware != null){
        batchStartAware.onBatchStart(availableSequence - nextSequence + 1);
      }
      // 遍历 nextSequence -> availableSequence
      while (nextSequence <= availableSequence){
        // 获取对应事件
        event = dataProvider.get(nextSequence);
        // 处理事件
        eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence);
        nextSequence++;
      }
      // 更新消费进度
      sequence.set(availableSequence);
    }catch (final TimeoutException e){
			// 。。。
    }
  }
}

// ProcessingSequenceBarrier#waitFor
// 参数就是需要等待的下标
public long waitFor(final long sequence)
  throws AlertException, InterruptedException, TimeoutException{
  checkAlert();
  // 直接调用等待策略的 waitFor 方法
  long availableSequence = waitStrategy.waitFor(sequence, cursorSequence, dependentSequence, this);
  if (availableSequence < sequence){
    return availableSequence;
  }
  return sequencer.getHighestPublishedSequence(sequence, availableSequence);
}
```

事件的处理调用的事 EventHandler#onEvent





### 消费者阻塞策略（Consumer Wait Strategy

消费速度大于生产速度的时候，消费者就需要阻塞等待生产者的事件生产（按照我看的源码，其实消费太慢的时候，生产者也在考虑使用 WaitStrategy 阻塞。



### 层级消费实现

如果消费者存在依赖关系，例如A只能消费B消费过的数据，这种时候需要做的就是 A 等待 B 的消费，A甚至不再需要等待生产者（不再直接等待。







## Others

Disruptor 是广播的实现，对于每个消费者都能获取到全部的 Event。

Disruptor 在 start 之后不允许添加消费者，并且在创建的时候就需要单/多生产者场景。
