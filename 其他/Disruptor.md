# Disruptor

---

[TOC]

---

## 总体介绍

> 以下源码基于 3.4.4 版本。

以下是 Disruptor 官网的介绍图，其中包含了所有的相关对象和概念：

![models](assets/models.png)



（Disruptor 最开始听说的是一个高性能无锁队列，但是实际上它不仅仅是队列。

Disruptor 类似于一套本地的 MQ 系统（Message Queue，消息队列），也可以看做是一套较为完整的生产者/消费者模型。



其中 RingBuffer 等同于中间队列，保存待消费事件以及协调生产者和消费者之间的依赖关系。

Sequence 表示的就是各类进度包括生产者和消费者的进度，由 RingBuffer 和各消费者持有，期间的依赖关系由 SequenceBarrier 实现。

（Sequence 表示偏移或者说进度，Sequencer 是上层的包装控制类。

生产者由 RingBuffer 统一管理，Disruptor 支持**单生产者和多生产者两种模式**，在多生产者模式下就需要注册 Sequence 的并发安全。

消费者则由各个消费者各自管理（因此各个消费者会分别消费事件，不会互相影响，类似 Kafka 的消费者组。





> 某些层面上 Disruptor 和 Guava 的 EventBus 有点像，后续可以对比一下两者的实现。
>
> EventBus 是监听器模型，而 Disruptor 则是生产者/消费者模型，相对来说 Disruptor 的实现更加复杂。



## 使用示例

```java
/**
 * 定义的事件类型
 */
static class DateEvent {
  private Date value;
  public void setValue(Date value) {
    this.value = value;
  }
  
  /**
   *  创建事件工厂
   */
  public static EventFactory<DateEvent> factory() {
    return DateEvent::new;
  }
  
  @Override
  public String toString() {
    return "DateEvent{" +
      "value=" + value +
      '}';
  }
}
/**
 * 事件处理器
 */
static class LongEventHandler implements EventHandler<DateEvent> {

  @Override
  public void onEvent(DateEvent event, long sequence, boolean endOfBatch) throws Exception {
    System.out.println("Event:" + event + ",sequence:" + sequence + ",endOfBatch:" + endOfBatch);
  }

}

public static void main(String[] args) throws InterruptedException {
  Disruptor<DateEvent> disruptor = new Disruptor<>(DateEvent.factory(), 4, DaemonThreadFactory.INSTANCE);
  disruptor.handleEventsWith(new LongEventHandler());
  // ！！！需要开启整个队列
  disruptor.start();
  for(int i= 0;i < 100;i++){
    TimeUnit.SECONDS.sleep(3);
    //	创建 Event 的时候也可以获取到 sequence
    disruptor.publishEvent((event, sequence) -> event.setValue(new Date()));
  }
}
```

每个 Disruptor 实例都对应一种事件类型，通过泛型指定，并且需要启动之后才可以使用，在定义好 EventHandler 之后，持有 Disruptor 的对象都可以发布事件。



<br>

## Disruptor 的创建流程

Disruptor 是整个框架的核心，负责协调生产者和队列，队列和消费者之间的关系，对外提供`start()`	、`shutdown()`、`handleEventsWith`、`publishEvent`

等基础 API。

（先通过创建过程来了解 Disruptor 整个对象的构造。

Disruptor 的创建方法如下：

```java
public Disruptor(
  final EventFactory<T> eventFactory,
  final int ringBufferSize,
  final ThreadFactory threadFactory,
  final ProducerType producerType,
  final WaitStrategy waitStrategy)
{
  this(
    RingBuffer.create(producerType, eventFactory, ringBufferSize, waitStrategy),
    new BasicExecutor(threadFactory));
}

/**
     * Private constructor helper
     */
private Disruptor(final RingBuffer<T> ringBuffer, final Executor executor)
{
  this.ringBuffer = ringBuffer;
  this.executor = executor;
}
```

Disruptor 创建最终只要求 RingBuffer 和 Executor，传入的类似 EventFactory 都是为了创建 RingBuffer。

**RingBuffer 是在启动前就创建好的（具体创建流程可以参考下文的 RingBuffer**

| 参数名称       | 含义                                                         |
| -------------- | ------------------------------------------------------------ |
| eventFactory   | 事件工厂（RingBuffer 会调用该接口方法，创建 RingBufferSize 个对象重复使用 |
| ringBufferSize | RingBuffer 的大小                                            |
| threadFactory  | 线程工厂（用于创建消费者所需要的线程，可以指定线程池         |
| producerType   | 生产者类型（单生产者还是多生产者会使用不同的并发策略         |
| waitStrategy   | 等待策略（生产者的等待策略，消费者的等待策略在指定消费者的时候决定 |



### RingBuffer 的作用和创建流程

以下是 3.4.4 版本中 RingBuffer 的定义注释：

![image-20230523164808847](assets/image-20230523164808847.png)

（基于环形数组实现的可重复使用实例对象的存储组件，保存的数据的在事件发生器和事件处理器之间交换。



RingBuffer 对外提供的创建方法如下：

```java
public static <E> RingBuffer<E> create(
  ProducerType producerType,	// 生产者类型
  EventFactory<E> factory,		// 事件工厂
  int bufferSize,							// Buffer 的大小
  WaitStrategy waitStrategy)	// 生产者的等待策略
{
  // 根据生产类型调用不同的创建方法
  // （其实最终创建的都是 RingBuffer，但是创建的生产者 Sequencer 不同
  switch (producerType)
  {
    case SINGLE:
      // 对应创建的是 SingleProducerSequencer
      return createSingleProducer(factory, bufferSize, waitStrategy);
    case MULTI:
      // 对应创建的是 MultiProducerSequencer
      return createMultiProducer(factory, bufferSize, waitStrategy);
    default:
      throw new IllegalStateException(producerType.toString());
  }
}
```



RingBuffer 在创建的过程中间就会调用 EventFactory#newInstance 方法创建所需要的所有对象，为了后续的重复使用。

（该部分逻辑在 RingBufferFields 中。







## 消费者

Disruptor 在启动前就需要指定消费者，同时也可以指定各消费者之间的依赖关系（也就是层级消费。

消费者的依赖关系也就是层级消费，以 EventHandlerGroup 作为基本单位进行依赖关系的编排，GroupA 可以根据 GroupB 的消费进度进行事件消费。



### 注册流程

Disruptor 提供了多种方式来进行注册：（消费者是想 Disruptor 对象注册的

1. EventHandler（最终会被包装为 EventProcessor 进行注册，当前 Disruptor 所持有的 RingBuffer 会作为 DataProvider 传入。
2. EventProcessorFactory（会使用工厂类直接创建 EventProcessor 进行消费者的注册
3. EventProcessor（继承了 Runnable，在启动时执行
4. WorkHandler



以下是通过 EventHandler 创建消费者的过程：

```java
// Disruptor#handleEventsWith
public final EventHandlerGroup<T> handleEventsWith(final EventHandler<? super T>... handlers){
  // 直接创建 EventProcessor
  // 初始化一个空的 Sequence 表示当前的消费者无依赖关系（除了生产者依赖
  return createEventProcessors(new Sequence[0], handlers);
}

// Disruptor#createEventProcessors
// 创建消费者实例
// 参数包含 barrierSequence，是他依赖的上层消费者，当前消费者只能消费上层已经全部消费过的数据
// 例如，当前依赖的三个上层消费者的 offset [1,10,10]，那么此时只能消费 1 的数据
EventHandlerGroup<T> createEventProcessors(final Sequence[] barrierSequences,final EventHandler<? super T>[] eventHandlers){
  // 只能在未开始的时候添加消费者
  checkNotStarted();
	// 每个 Handler 对应一个 Sequence，这里对应的就是每个 EventHandler 的消费进度
  final Sequence[] processorSequences = new Sequence[eventHandlers.length];
  // 创建 Barrier（是当前消费者依赖的 Barrier
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
  // barrierSequences 是指定当前消费者的依赖对象
  updateGatingSequencesForNextInChain(barrierSequences, processorSequences);
  // 返回一个 EventhandlerGroup，调用当前方法的所有 EventHandler 会被包含在一个 Group 里面
  // 通过 EvntHandlerGroup 可以进一步编排后续处理逻辑
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
    // barrierSequences 代表的的是当前消费者集的依赖，需要取消 endOfChain 的标记
    consumerRepository.unMarkEventProcessorsAsEndOfChain(barrierSequences);
  }
}
```



注册消费主要流程如下：

1. 检查 Disruptor 是否已经开启
2. 创建对应的 EventProcessor （具体对象为 BatchEventProcessor，包含了ExceptionHandler 和当前的 RingBuffer。
3. 向 ConsumerRepository 注册当前的消费者信息（消费者并未启动，所以此时需要集中管理
4. 处理 Sequence
   - 向 RingBuffer 添加当前的消费者的 Sequence（相互协调保证生产者的 Sequence 不超过消费者
   - 处理具有依赖关系的消费者之间的 Sequence（将传入的 barrierSequences 添加到创建的消费者中，消费者根据依赖的 Sequence
   - 移除 RingBuffer 中当前消费者依赖的 Sequence（传入的 barrierSequences 参数，
5. 返回 EventHandlerGroup（EventHandlerGroup 对象包含 after 等方法可以作为顺序处理逻辑的编排方法

**消费者最终的实例对象为 BatchEventProcessor，通过 RingBuffer 获取事件以及调用对应处理方法的逻辑都在该类中实现。**

很关键的是，Disruptor 不允许在运行过程中添加消费者，所以在  `Disruptor#start()` 前就需要注册全部的消费者。

<br>

Disruptor 的 ConsumerRepository 对象用于持有所有的 EventHandler 以及对应的 Sequence 信息（算是一个辅助类，用于完成一些相对独立的逻辑。

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
    // 分别存了三份！！！应该有不同的获取需求吧
    eventProcessorInfoByEventHandler.put(handler, consumerInfo);
    eventProcessorInfoBySequence.put(eventprocessor.getSequence(), consumerInfo);
    consumerInfos.add(consumerInfo);
}
```



### 启动流程

启动流程对应的是 `Disruptor#start` 方法，在启动之前所有的消费者都以 EventProcessor 的形式保存在 ConsumerRepository 中。

方法的源码如下：

```java
public RingBuffer<T> start(){
  // 只能启动一次
  checkOnlyStartedOnce();
  // 遍历所有注册的消费者并启动
  for (final ConsumerInfo consumerInfo : consumerRepository){
    consumerInfo.start(executor);
  }
	// 返回 RingBuffer
  return ringBuffer;
}
```

启动的过程就是创建并启动所有消费者对象的过程，EventProcessor 继承了 Runnable 方法可以直接使用线程池执行该类。



> Disruptor 的启动流程主要就是启动所有的消费者。
>
> （消费者的执行线程由传入的线程池控制，一般情况下创建一个固定大小的线程池数量较为合适。





### 消费流程







## 事件生产流程

RingBuffer 回预先创建所有的事件对象，所以后续的发布流程就是获取指定对象，填充属性并且发布。

过程中主要控制生产者的生产速度，不能把没消费完的事件覆盖了。



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





### 消费者等待策略（Consumer Wait Strategy

消费速度大于生产速度的时候，消费者就需要阻塞等待生产者的事件生产（按照我看的源码，其实消费太慢的时候，生产者也在考虑使用 WaitStrategy 阻塞。

WaitStrategy 是顶层的接口，定义了对应的等待策略：

```java
public interface WaitStrategy
{
    /**
     * Wait for the given sequence to be available.  It is possible for this method to return a value
     * less than the sequence number supplied depending on the implementation of the WaitStrategy.  A common
     * use for this is to signal a timeout.  Any EventProcessor that is using a WaitStrategy to get notifications
     * about message becoming available should remember to handle this case.  The {@link BatchEventProcessor} explicitly
     * handles this case and will signal a timeout if required.
     *
     * @param sequence          to be waited on.
     * @param cursor            the main sequence from ringbuffer. Wait/notify strategies will
     *                          need this as it's the only sequence that is also notified upon update.
     * @param dependentSequence on which to wait.
     * @param barrier           the processor is waiting on.
     * @return the sequence that is available which may be greater than the requested sequence.
     * @throws AlertException       if the status of the Disruptor has changed.
     * @throws InterruptedException if the thread is interrupted.
     * @throws TimeoutException if a timeout occurs before waiting completes (not used by some strategies)
     * 等待直到可以消费
     */
    long waitFor(long sequence, Sequence cursor, Sequence dependentSequence, SequenceBarrier barrier)
        throws AlertException, InterruptedException, TimeoutException;

    /**
     * Implementations should signal the waiting {@link EventProcessor}s that the cursor has advanced.
     * 唤醒所有等待的线程
     */
    void signalAllWhenBlocking();
}

```

对应的子类实现有如下几种：

| 实现类                          | 作用                                                    |
| ------------------------------- | ------------------------------------------------------- |
| BlockingWaitStrategy            | 使用 ReentrantLock$Condition#await 实现的阻塞等待       |
| BusySpinWaitStrategy            | 调用 Thread#onSpinWait 实现等待（可能没有，那就是空轮训 |
| LiteBlockingWaitStrategy        |                                                         |
| LiteTimeoutBlockingWaitStrategy |                                                         |
| PhasedBackoffWaitStrategy       |                                                         |
| SleepingWaitStrategy            |                                                         |
| TimeoutBlockingWaitStrategy     |                                                         |
| YieldingWaitStrategy            |                                                         |

（懒得看了，有空再写作用。



### 层级消费实现

如果消费者存在依赖关系，例如A只能消费B消费过的数据，这种时候需要做的就是 A 等待 B 的消费，A甚至不再需要等待生产者（不再直接等待。

类似的依赖关系是通过 SequenceBarrier 来实现的，Barrier 中保存了依赖的消费者的 Sequence，通过对 Sequence 的比较来判断自己的消费下标。

（最上层的消费者根据的是生产者的 Sequence 来判断自己的消费进度，如果存在依赖关系之后，下级的消费者只需要关注上级消费者的 Sequence 就好。



ProcessingSequenceBarrier 中保存了 WaitStrategy 和依赖的所有消费者的 Sequence。

```java
// ProcessingSequenceBarrier 构造函数
ProcessingSequenceBarrier(
  final Sequencer sequencer,
  final WaitStrategy waitStrategy,
  final Sequence cursorSequence,
  final Sequence[] dependentSequences){			// 依赖的所有 Sequence
  this.sequencer = sequencer;
  this.waitStrategy = waitStrategy;
  this.cursorSequence = cursorSequence;
  if (0 == dependentSequences.length){
    dependentSequence = cursorSequence;
  } else{
    // 如果是多个会被包装成 Sequence
    dependentSequence = new FixedSequenceGroup(dependentSequences);
  }
}
```

之后看如何实现的依赖关系：

```java
// ProcessingSequenceBarrier#waitFor
// 传入的参数是下次希望消费的位置
// 返回的是可以消费的位置，可能比传入的大
public long waitFor(final long sequence)
  throws AlertException, InterruptedException, TimeoutException{
  checkAlert();
  long availableSequence = waitStrategy.waitFor(sequence, cursorSequence, dependentSequence, this);
  if (availableSequence < sequence){
    return availableSequence;
  }
  return sequencer.getHighestPublishedSequence(sequence, availableSequence);
}
```





#### Sequence 

![image-20230523164830084](assets/image-20230523164830084.png)

（同步序列类用于追踪 RingBuffer 和事件处理器的进程，支持多种形式的同步操作，包括 CAS 以及顺序写，另外也尝试使用更加有效率的方式消除伪共享，比如在属性前后增加填充。



对于生产者来说，Sequence 保存了写入的位置，保存在 RingBuffer。

对于消费者来说，Sequence 保存在自身的读取的位置，多个消费者不共享。

Sequence 本身使用 Padding 的方式来避免缓存行的伪共享问题（在 JDK1.8之后应该也可以用 @Contended 来实现的。



##### Sequencer

![image-20230523164736005](assets/image-20230523164736005.png)

Sequence 的管理者，包含了生产者和消费者的相关 Sequence（就是持有了生产者的写入指针和消费者的读取指针。

根据生产者的个数分为 SingleProducerSequencer 和 MutilProducerSequencer。

如果消费者有多个，Single 就是保存单生产者和消费者的关系，而 Mutil 保证的就是多生产者和消费者的关系，以及多生产者之间的并发安全。







## 总结

> Disruptor 实现高性能的基础。

1. RingBuffer 对于对象的复用

RingBuffer 就是 Disruptor 实现的对象池。

复用的对象数组可以降低了 GC 频率，提高 CPU 的利用率，相对于 ArrayBlockedQueue 来说，RingBuffer 创建的事件对象数目是固定的。

2. 避免了伪共享（缓存行

（伪共享的影响可以参考 Java 中横向和纵向访问二维数组的时间消耗，存在几倍的延迟。

Sequence 中通过填充 long 对象的形式来避免伪共享。

3. 无锁化实现

在 RingBuffer 生产者的实现中，区分了单生产者和多生产者，多生产者以及消费者层面都是通过 CAS 来保证并发安全。
