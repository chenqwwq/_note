# Ribbon

---

## 概述 

Ribbon 为 SpringCloud 提供了服务定位，负载均衡的功能，可以结合 RestTemplate 以及 Feign 使用。



## 相关组件

| **接口**                 | 作用                             |
| ------------------------ | -------------------------------- |
| IClientConfig            | 读取配置                         |
| IRule                    | 负载均衡规则                     |
| IPing                    | 实例检测，是否可连接             |
| ServerList<Server>       | 交给Ribbon的实例列表             |
| ServerListFilter<Server> | 过滤掉不符合条件的实例           |
| ILoadBalance             | Ribbon的入口，选择具体的服务实例 |
| ServerListUpdater        | 更新交给Ribbon的List的策略       |



## RestTemplate 的LB

> LoadBalancerInterceptor 是整个负载均衡的实现中心。



### 内置的负载均衡策略



| 实现类                    | 规则                                          |
| ------------------------- | --------------------------------------------- |
| AvailabilityFilteringRule | 过滤不可用的服务实例，可以定义不可用          |
| BestAvailableRule         | 选择一个最小的并发请求的服务实例              |
| RandomRule                | 随机选择一个实例                              |
| RoundRobinRule            | 轮询选择（公司默认）                          |
| ZoneAvoidanceRule         | Ribbon 默认，根据服务分区以及服务的可用性判断 |
| RetryRule                 | 重试机制，失败可选择另外一个服务实例          |
| WeightedResponseTimeRule  | 响应时间加权，时间越长权重越小                |





## 参考

[官方文档](https://github.com/Netflix/ribbon/wiki/Getting-Started)

[Ribbon的负载均衡策略、原理和扩展](https://www.jianshu.com/p/79b9cf0d0519)