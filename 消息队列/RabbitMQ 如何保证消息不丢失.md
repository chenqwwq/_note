# RabbitMQ 如何保证消息不丢失?

---

## 脑图

![RabbitMQ 脑图1.png](assets/RabbitMQ 脑图1.png)



## 概述

RabbitMQ 可以分为 Producer（生产者），Consumer（消费者），Exchange（交换机），Queue（队列）四个角色。

消息的流经过程就是 Producer -> Exchange -> Queue -> Consumer。

> 和 Kafka 不同，RabbitMQ 不会直接和 Queue（Topic） 打交道，而是通过 Exchange，生产者甚至不知道消息最终去了哪里。

<br>

所以要保证消息不丢就必须保证以下流程：

1. Producer 到 Exchange 的过程，确保 Exchange 接收到消息
2. Exchange 到 Queue 的过程，确保消息被正确的投递
3. Queue 到 Consumer 的过程，确保消息被正常的消费和 ack

还有就是，消息在 Exchange 和 Queue 的持久性，不能因为 Broker 的宕机导致消息的丢失，所以 Exchange ，Queue 和消息都需要持久化。

> 持久化十分损耗内存，使用时谨慎判断是否必要。

<br>

<br>

## Producer 到 Exchange 的过程

该过程可以通过[生产者确认（Publisher Confirm）](https://www.rabbitmq.com/tutorials/tutorial-seven-java.html) 来保证。

**Confirm 机制开启之后，会为生产者的每条消息添加从1开始的id，如果 Broker 确定接收到消息，则返回一个 confirm。**

Confirm 机制只负责到消息是否到达 Exchange 不负责后续的消息投递等流程，另外 RabbitMQ 也提供了事务的情况，事务的作用就是确保消息一定能够全部到达 Broker。

<br>

Springboot 的 RabbitMQ 实现中，可以对 RabbitTemplate 添加 RabbitTemplate.ConfirmCallback 回调函数，该回调需要额外配置以下内容

<img src="assets/rabbitmq-publish-confirm配置.png" alt="image-20210324235140208" style="zoom:67%;" />

**confirm 的回调方法在消息投递出去之后触发，不论成功还是失败都会。**

以回执的方式明确消息是否真正到达 Broker，如果未到达则可以做下一步的处理，重发或者入库等等，方法相关入参如下：

<img src="assets/rabbitmq-publish-confirm%E7%A4%BA%E4%BE%8B.png" alt="image-20210325000559884" style="zoom:67%;" />

<br>

<br>

## Exchange 到 Queue 的过程

**该过程可以通过 RabbitMQ 提供的 mandatory 参数设置。**

mandatory 参数的作用就是确保消息被正确的投递到具体的队列，如果在 Broker 中无法匹配到具体队列，那么也会触发回调。

<br>

Springboot 的客户端封装也提供了 RabbitTemplate.ReturnCallback 回调方法，用来监听消息的状态。

想要该参数生效，以下两个配置必须同时配置。

<img src="assets/rabbitmq-springboot-mandatory%E9%85%8D%E7%BD%AE.png" alt="image-20210325000405539" style="zoom:67%;" />

方法相关入参如下：

<img src="assets/rabbitmq-mandatory%E5%9B%9E%E8%B0%83%E7%A4%BA%E4%BE%8B.png" alt="image-20210325000626949" style="zoom:67%;" />

<br>

回调并没有办法直接解决消息的投递失败问题，对失败投递进行报警，然后人工排查情况才是关键。

> mandatory 的回调只有消息投递失败的时候才会触发，正常投递不会触发。
>
> 这和 publish confirm 不同，publish confirm 是不管失败还是成功都会触发回调的。

<br>

### 备份交换机

**RabbitMQ 中还存在一个备份交换机（alternate-exchange）的概念，如果消息在正常的交换机无法匹配到队列的时候，消息会被转发到该交换机，由该交换机进一步投递。**

**所以就可以使用备份交换机收集无法匹配到 Queue 的消息。**

一般该交换机被设置为 FANOUT 模式，确保消息可以被直接投递。

<br>

<br>



## Queue 到 Consumer 的过程

RabbitMQ 中保存的消息，只有在被 ack 之后才会主动删除，所以在 ack 消息之前必须要确保消息的正常消费。

> 这个也是 RabbitMQ 和 Kafka 不同的点。
>
> Kafka 在消费者 ack 之后并不会删除消息，只有消息累积到一定阈值（时间或者大小）之后才会删除，甚至可以不删除，因此 Kafka 即使作为存储服务也没啥问题。



<br>

### RabbitMQ 原生 ack 模式

RabbitMQ 的消费者端提供了**自动和手动两种 ack 方式**。

[Consumer Acknowledgement Modes and Data Safety Considerations](https://www.rabbitmq.com/confirms.html#acknowledgement-modes)

**在自动确认的模式下，消息被认为在发送之后就算成功处理，因此很容易造成消息丢失，但是自动确认在很大程度上提高了吞吐量。**

> Consumer 是直接和 Queue 接触的，一个 Queue 可以由多个 Consumer 共同消费，如果一个 Consumer 断线，那么该 Consumer 上未 ack 的消息会被转发到其他的 Consumer 上，此时又会存在重复消费的问题。

**对于手动确认**，RabbitMQ 定义了以下三种形式：![image-20210325003633390](assets/rabbitmq手动ack类型.png)

<br>

basic.ack 就是确认消费成功，Broker 在接收到该条 ack 之后会尝试删除对应的消息。

basic.reject 和 basic.nack 的作用是一样的，区别就在于语义上，作用都是拒绝消息，并且可以通过参数确定消息是否需要重新入队列。

<br>

> **消费者连接的时候就需要指定 ack 的模式。**
>
> RabbitMQ 提供了两大类 ack 模式：手动和自动
>
> 1. 自动 ack 会在消息到达消费者之后直接删除队列中的消息
> 2. 手动 ack 分为 ack / nack / reject 三种。

<br>

<br>

### Java 原生客户端 ack 实现

在创建 Consumer 的时候就需要指定 ack 的形式:

![](assets/rabbitmq-创建consumer.png)

上图中的方法参数 autoAck 就表示是否开启**自动 ack**。

对于三种手动确认的方法也分别提供了方法。

```java
void basicAck(long deliveryTag, boolean multiple) throws IOException;
    
void basicNack(long deliveryTag, boolean multiple, boolean requeue)
            throws IOException;

void basicReject(long deliveryTag, boolean requeue) throws IOException;
```

deliveryTag 可以简单理解为消息的 id。

multiple 参数的含义是是否为批量操作，例如 basicAck 方法，如果为批量操作，会将 deliveryTag 之前的消息都 ack。

requeue 参数表示是否需要重回队列，如果为 false，那么在方法调用后消息就会被丢弃或者转发到死信队列，如果为 true，消息就会重新进入队列，重新下发到消费者。



> Java 原生的 RabbitMQ 客户端基本就简单实现了。

<br>

### SpringBoot 中的 ack 实现

SpringBoot 根据以上的 ack 方法抽象提供了三种 AcknowledgeMode，具体如下：

<img src="assets/springboot-rabbitmq-ackmode.png" alt="image-20210325002202583" style="zoom:67%;" />

None 对应的就是 RabbitMQ 的 自动 ack，在消息被下发后就认为是消费成功，Broker 可以删除该消息。

> 实际开发中慎用该配置！
>
> 但是如果处理失败就无法对该条消息进行重试，因为已经从队列中删除。
>
> 自动 ack 能稍微提高消息速度，ack 之后 Broken 会立马补消息到 prefetch 个。

MANUAL 需要用户在 listener 中手动调用 ack / nack / reject 方法。

**AUTO 是由 SpringBoot 控制的 ack 模式，如果 listener 返回正常，则调用 ack，如果抛异常则调用 nack。**

> 是在 RabbitMQ 官方提供了客户端实现的基础上封装的记住。

另外的还有 default-requeue-rejected 配置，表示在消息处理失败之后是否需要重回队列。

> **SpringBoot 的客户端默认是会重回队列的，所以如果 Listener 抛异常而不进一步处理，消息会进入死循环。**

<br>

## 阅读

[Consumer Acknowledgements and Publisher Confirms](https://www.rabbitmq.com/confirms.html#publisher-confirms)