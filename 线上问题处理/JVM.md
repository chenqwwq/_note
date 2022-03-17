# JVM 相关线上问题



## 2019-EntranceNode-OOM

### 概况

线上 OOM 告警，直接看 PingPoint，此时堆内存为 4G，OOM 之前多次 FullGC 并未收回多少空间，最后 OOM 退出。

### 临时处理

OOM 之前的 FullGC 并没有回收多少内存空间，所有基本可以断定是内存溢出。

线上切了一个节点然后执行 jmap histo:live [pid] | head -n 10 ，获得当前最多的10个对象，不是 String 或者 Char 数组，而是 Sentinel 的 Node 节点。

线上加了依赖包，但是并未开启 Sentinel，后发现 Sentinel 是默认开启，紧急处理就是关闭 Sentinel。（此时并未找到真实原因。

### 第二次发生

第二次 OOM 是相同的情况，但是此时 Sentinel 已经关闭，照旧使用 jmap 查看了线上的情况，发现最大的对象是 Metrix（普罗米修斯监控的相关统计类。

测试环境压测无法复现。

后翻看源码发现 Metrix 是根据 RequestMapping 创建的，发现了 SpringMVC 的问题，SpringMVC 会在匹配结束后将 RequestMapping 作为 BEST_MATCH 类似的 Attribute，

公司习惯使用 RESTFull 风格的 API 地址，包含 PathVariable，但是公司内部使用的 Spring Lite MVC 修改了该项，BEST_MATCH 会直接拿真实参数替换 PathVariable，导致的大量创建 Metrix。

去除 Spring Lite MVC 引用后正常。

（使用 Spring Lite MVC 的原因，是因为包含 PathVariable 的请求地址匹配性能并不高，会先和所有的 RequestMappingInfo（RequestMapping 解析而来）尝试完全匹配，失败时候又会逐个进行正则匹配。





## 2020-云作业项目组线上FullGC频繁问题

### 概述

是别的项目组的问题，线上 Full GC 非常频繁导致的请求大量延时。

当场进入 pingpoint 查看，发现 Full GC 密集到看不出时间（连片的 Full GC，但是每次 Full GC 之后回收的内存空间很大，说明大量并非是长久存在的对象进入了老年代，在老年代空间不足时进行了 Full GC。

第一时间是认为参数问题，可能是进入老年代的对象大小设置错误，但是发现并没有该参数设置。

后面使用 jmap -heap 查看堆的情况，惊奇发现年轻代整个只有 10mb 左右的大小，此时就知道原因了，晋升的原因要不是频繁的 YGC，导致对象年龄暴增，就是 S1 区过小无法存放剩余对象导致直接晋升到老年代。

之后就是寻找年轻代只有 10mb 的原因，并没有配置相关参数，2G内存，也是 1:2 以及 1:1:8 的默认配置，讲道理应该有 700mb 的年轻代。

最后发现是使用的 GC 处理器的问题，使用的是 Parallel Scavenge 搭配 Parallel Old 的组合，而 Parallel Scavenge 处理器包含一个参数就是  -XX:+UseAdpativeSizePolicy 会动态调整各个分区的大小以达到目标系统规定的最低相应时间或者收集频率等。

解决办法就是将我们服务的 JVM GC 配置全套抄过去。

