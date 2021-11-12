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



### 参考

[官方文档 ThreadPool 相关配置](https://github.com/Netflix/Hystrix/wiki/Configuration#ThreadPool)