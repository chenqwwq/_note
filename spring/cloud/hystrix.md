# Hystrix



## 相关配置


```java
# 核心线程数 默认值10
hystrix.threadpool.default.coreSize=10
# 最大线程数 默认值10 在1.5.9版本之前该值总是等于coreSize
hystrix.threadpool.default.maximumSize=10
# 阻塞队列大小 默认值-1表示使用同步队列 
hystrix.threadpool.default.maxQueueSize=-1
# 阻塞队列大小拒绝阈值 默认值为5 当maxQueueSize=-1时，不起作用
hystrix.threadpool.default.queueSizeRejectionThreshold=5
# 释放线程时间 min为单位 默认为1min，当最大线程数大于核心线程数的时
hystrix.threadpool.default.keepAliveTimeMinutes=1
# 是否允许maximumSize配置生效，默认值为false
hystrix.threadpool.default.allowMaximumSizeToDivergeFromCoreSize=true
```

> queueSizeRejectionThreshold 表示拒绝任务的阈值，当该值大于 maxQueueSize 时，maximumSize 失效。
>
> maximumSize 生效的条件如下:
>
> -  queueSizeRejectionThreshold > maxQueueSize
> - allowMaximumSizeToDivergeFromCoreSize = true

> maxQueueSize 表示任务队列的大小，当 maxQueueSize 为 -1 时，使用同步队列，不存储任何任务。



### Command Properties

#### Execution

此类配置控制 HystrixCommand#run() 的执行。

##### execution.isolation.strategy 隔离策略

重点配置项，Hystrix 中提供 Thread（线程池）和信号量（Semaphore）两种形式的隔离策略，默认情况下使用 Thread 策略。

官方更加推荐使用给HystrixCommand 执行 Thread 的隔离策略，而使用 HystrixObservableCommand 实现 Semaphore。

使用 Thread 策略，相当于在程序中有一层额外的保护层，用于减少网络超时带来的影响。



| 配置名                                              | 默认值 | 配置作用                                                     |
| --------------------------------------------------- | ------ | ------------------------------------------------------------ |
| execution.isolation.thread.timeoutInMilliseconds    | `1000` | 在 THREAD 模式下生效，配置整体的请求超时时间，如果需要可以进一步配置每个请求的超市时间 |
| execution.timeout.enabled                           | true   | 请求是否带有超时时间限制                                     |
| execution.isolation.semaphore.maxConcurrentRequests | 10     | 在 SEMAPHORE 模式下生效，配置在请求的最大并发数，超出部分会被拒绝 |



#### Circuit Breaker

| 配置名                                   | 默认值 | 配置作用                                                     |
| ---------------------------------------- | ------ | ------------------------------------------------------------ |
| circuitBreaker.enabled                   | true   | 是否开启断路器                                               |
| circuitBreaker.requestVolumeThreshold    | 20     | 断路器打开阈值，只有在大于该请求数的情况下才会出现短路       |
| circuitBreaker.sleepWindowInMilliseconds | 5000   | 断路器打开后的熔断时间，时间内的请求全部会被熔断，之后会进一步测试 |
| circuitBreaker.errorThresholdPercentage  | 50     | 错误百分比阈值，只有请求错误率超过 50% 的                    |
| circuitBreaker.forceOpen                 | false  | 手动开关，可以强制开启熔断                                   |
| circuitBreaker.forceClose                | false  | 手动开关，可以强制关闭熔断                                   |

**统计的窗口不在熔断器的配置范围之内，在窗口时间之内只有请求数大于 requestVolumeThreshold 并且异常率大于 errorThresholdPercentage 并且 forceClose 是关闭状态才会打开熔断器。**



### Metrics

| 配置名                                       | 默认值 | 配置作用                                                     |
| -------------------------------------------- | ------ | ------------------------------------------------------------ |
| metrics.rollingStats.timeInMilliseconds      | 10000  | 统计窗口总大小                                               |
| metrics.rollingStats.numBuckets              | 10     | 窗口数量（每个窗口对应的时间 = timeInMilliseconds / numBuckets |
| metrics.rollingPercentile.enabled            | true   | 是否计算请求的延迟信息                                       |
| metrics.rollingPercentile.timeInMilliseconds | 60000  | 统计窗口时间                                                 |
| metrics.rollingPercentile.numBuckets         | 6      | rollingPercentile 窗口数                                     |

该配置用于管控 HystrixCommand 和 HystrixObservableCommand 中的指标抽取。





### ThreadPool Properties

| 配置名 | 默认值 | 配置作用 |
| ------ | ------ | -------- |
|        |        |          |
|        |        |          |
|        |        |          |



### 参考

[官方文档 ThreadPool 相关配置](https://github.com/Netflix/Hystrix/wiki/Configuration#ThreadPool)