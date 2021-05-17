# RabbitMQ 如何保证消息不丢失

> 刚加班到家，已经12点多了，随便写点吧。

---

[TOC]

---

## 概述

简单来说，RabbitMQ 可以分为 Producer（生产者），Consumer（消费者），Exchange（交换机），Queue（队列）四个角色。

消息的流经过程就是 Producer -> Exchange -> Queue -> Consumer。

> 和 Kafka 不同，RabbitMQ 不会直接和 Queue（Topic） 打交道，而是通过 Exchange，生产者甚至不知道消息最终去了哪里。



所以要保证消息不丢就必须保证以下流程：

1. Producer 到 Exchange 的过程，确保 Exchange 接收到消息
2. Exchange 到 Queue 的过程，确保消息被正确的投递
3. Queue 到 Consumer 的过程，确保消息被正常的消费和 ack

还有就是，消息在 Exchange 和 Queue 的持久性，不能因为 Broker 的宕机导致消息的丢失，所以 Exchange ，Queue 和消息都需要持久化。





## Producer 到 Exchange 的过程

该过程可以通过[生产者确认（Publisher Confirm）](https://www.rabbitmq.com/tutorials/tutorial-seven-java.html) 来保证。

**Confirm 机制开启之后，会为生产者的每条消息添加从1开始的id，如果 Broker 确定接收到消息，则返回一个 confirm。**

Confirm 机制只负责到消息是否到达 Exchange 不负责后续的消息投递等流程，另外 RabbitMQ 也提供了事务的情况，事务的作用就是确保消息一定能够全部到达 Broker。



 

Springboot RabbitMQ 中，可以对 RabbitTemplate 添加 RabbitTemplate.ConfirmCallback 回调函数，该回调需要额外配置以下内容

<img src="assets/rabbitmq-publish-confirm配置.png" alt="image-20210324235140208" style="zoom:67%;" />

confirm 的回调方法在消息投递出去之后触发，不论成功还是失败都会。

以回执的方式明确消息是否真正到达 Broker，如果未到达则可以做下一步的处理，重发或者入库等等，方法相关入参如下：

<img src="assets/rabbitmq-publish-confirm%E7%A4%BA%E4%BE%8B.png" alt="image-20210325000559884" style="zoom:67%;" />





## Exchange 到 Queue 的过程

**该过程可以通过 RabbitMQ 提供的 mandatory 参数设置。**

mandatory 参数的作用就是确保消息被正确的投递到具体的队列，如果在 Broker 中无法匹配到具体队列，那么也会触发回调。

Springboot 的客户端封装也提供了 RabbitTemplate.ReturnCallback 回调方法，用来监听消息的状态。

想要该参数生效，以下两个配置必须同时配置。

<img src="assets/rabbitmq-springboot-mandatory%E9%85%8D%E7%BD%AE.png" alt="image-20210325000405539" style="zoom:67%;" />

方法相关入参如下：

<img src="assets/rabbitmq-mandatory%E5%9B%9E%E8%B0%83%E7%A4%BA%E4%BE%8B.png" alt="image-20210325000626949" style="zoom:67%;" />



回调并没有办法直接解决消息的投递失败问题，对失败投递进行报警，然后人工排查情况才关键。

> mandatory 的回调只有消息投递失败的时候才会触发，正常投递不会触发。
>
> 这和 publish confirm 不同，publish confirm 是不管失败还是成功都会触发回调的。



### 备份交换机

**RabbitMQ 中还存在一个备份交换机（alternate-exchange）的概念，如果消息在正常的交换机无法匹配到队列的时候，消息会被转发到该交换机，由该交换机进一步投递。**

**所以就可以使用备份交换机收集无法匹配到 Queue 的消息。**

一般该交换机被设置为 FANOUT 模式，确保消息可以被直接投递。







## Queue 到 Consumer 的过程

RabbitMQ 中保存的消息，只有在被 ack 之后才会主动删除，所以在 ack 消息之前必须要确保消息的正常消费。

> 这个也是 RabbitMQ 和 Kafka 不同的点。
>
> Kafka 在消费者 ack 之后并不会删除消息，只有大

> Consumer 是直接和 Queue 接触的，一个 Queue 可以由多个 Consumer 共同消费，如果一个 Consumer 断线，那么该 Consumer 上未 ack 的消息会被转发到其他的 Consumer 上，此时又会存在重复消费的问题。

RabbitMQ 的消费者端提供了自动和手动两种 ack 方式。

[Consumer Acknowledgement Modes and Data Safety Considerations](https://www.rabbitmq.com/confirms.html#acknowledgement-modes)

在自动确认的模式下，消息被认为在发送之后就算成功处理，因此很容易造成消息丢失，但是自动确认在很大程度上提高了吞吐量。

对于手动确认，RabbitMQ 定义了以下三种形式：

![image-20210325003633390](/home/chen/_note/pic/image-20210325003633390.png)

basic.ack 就是确认消费成功，Broker 在接收到该该条 ack 之后会尝试删除对应的消息。

basic.reject 和 basic.ack 的作用是一样的，区别就在于语义上。

> 消费者连接的时候就需要指定 ack 的模式。



Java 客户端在此基础上提供了三种方法声明：

```java
void basicAck(long deliveryTag, boolean multiple) throws IOException;
    
void basicNack(long deliveryTag, boolean multiple, boolean requeue)
            throws IOException;

void basicReject(long deliveryTag, boolean requeue) throws IOException;
```

deliveryTag 可以简单理解为消息的 id。

multiple 参数的含义是是否为批量操作，例如 basicAck 方法，如果为批量操作，会将 deliveryTag 之前的消息都 ack。

requeue 参数表示是否需要重回队列，如果为 false，那么在方法调用后消息就会被丢弃或者转发到死信队列，如果为 true，消息就会重新进入队列，重新下发到消费者。



SpringBoot 抽象提供了三种 AcknowledgeMode，具体如下：

<img src="/home/chen/_note/pic/image-20210325002202583.png" alt="image-20210325002202583" style="zoom:67%;" />

None 对应的就是 RabbitMQ 的 自动 ack，在消息被下发后就认为是消费成功，Broker 可以删除该消息。

MANUAL 需要用户在 listener 中手动调用 ack/nack 方法。

AUTO 是由 SpringBoot 控制的 ack 模式，如果 listener 返回正常，则调用 ack，如果抛异常则调用 nack。

> SpringBoot 的实现中并不会使用 basic.reject 方法拒绝消息。





另外的还有 default-requeue-rejected 配置，表示在消息处理失败之后是否需要重回队列。

> SpringBoot 的客户端默认是会重回队列的，所以如果 Listener 抛异常而不进一步处理，消息会进入死循环。





## 阅读

[Consumer Acknowledgements and Publisher Confirms](https://www.rabbitmq.com/confirms.html#publisher-confirms)