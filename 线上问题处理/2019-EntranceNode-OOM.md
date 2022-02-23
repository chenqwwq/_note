# 线上的 OOM 问题处理



## 概况

线上 OOM 告警，直接看 PingPoint，此时堆内存为 4G，OOM 之前多次 FullGC 并未收回多少空间，最后 OOM 退出。



## 处理

OOM 之前的 FullGC 并没有回收多少内存空间，所有基本可以断定是内存溢出。

线上切了一个节点然后执行 jmap histo:live [pid] | head -n 10 ，获得当前最多的10个对象，不是 String 或者 Char 数组，而是 Sentinel 的 Node 节点。

线上加了依赖包，但是并未开启 Sentinel，后发现 Sentinel 是默认开启，紧急处理就是关闭 Sentinel。（此时并未找到真实原因。





## 211111第二次发生

第二次 OOM 是相同的情况，但是此时 Sentinel 已经关闭，照旧使用 jmap 查看了线上的情况，发现最大的对象是 Metrix（普罗米修斯监控的相关统计类。

测试环境压测无法复现。

后翻看源码发现 Metrix 是根据 RequestMapping 创建的，发现了 SpringMVC 的问题，SpringMVC 会在匹配结束后将 RequestMapping 作为 BEST_MATCH 类似的 Attribute，

公司习惯使用 RESTFull 风格的 API 地址，包含 PathVariable，但是公司内部使用的 Spring Lite MVC 修改了该项，BEST_MATCH 会直接拿真实参数替换 PathVariable，导致的大量创建 Metrix。

去除 Spring Lite MVC 引用后正常。

（使用 Spring Lite MVC 的原因，是因为包含 PathVariable 的请求地址匹配性能并不高，会先和所有的 RequestMappingInfo（RequestMapping 解析而来）尝试完全匹配，失败时候又会逐个进行正则匹配。