# Sentinel

---

[TOC]

---

## 随记



Snetinel 的作用：

- 流量控制
- 线程数隔离
- 熔断降级
- 黑白名单 





Sentinel 中会对各类调用建立一个完整的调用链，根节点为 Constants.ROOT，作为所有调用链的最初。

刚开始会以**上下文名称**建立一个 EntranceNode，作为真实的入口（Constants.ROOT  的所有 childs 就是所有的上下文入口节点，在 ContextUtil 中以静态 Map 的形式保存，在全局共享。

之后第一个 **NodeSelectorSlot** 会先尝试用上下文名称获取一个 EntranceNode，并将其作为 LastNode（创建的 EntranceNode 还是以资源名称为标识的，**完成相同上下文下的资源统计。**

> 以上下文名称作为缓存的 Key，可以保证同一个上下文中不同的 SphU#entry 在不同的资源创建不同的节点。
>
> 考虑一个情况，相同资源名称，多次获取不同的资源。
>
> ```java
> ContextUtil.enter("Demo");
> SphU.entry("A");
>     
> ContextUtil.enter("Demo1");
> SphU.entry("A");
> ```
>
> 类似以上的情况，因为 ProcessorSlotChain 是和 资源名称绑定的，所以两个 entry("A") 使用的是同一个 ProcessorSlotChain，如果继续使用 ResourceWrapper 去获取第二层的 EntranceNode 获取到的就会是相同的一个（在 Demo 上下文和 Demo1 上下文）。
>
> 所以在第二层的节点需要以**上下文名称**作为 Key，来保存 EntranceNode。

接下来是 **ClusterBuilderSlot**，这个 Slot 使用静态 Map 保存了资源名称和 DefaultNode 的映射关系，**完成对同一个资源下的指标统计**。

之后是 LogSlot，在接下来是 StatisticSlot，



CtSph 会根据 ResourceWrapper 选择对应的处理链（ProcessorSlotChain）。

> 这里就可以说一个请求具体走哪条执行链都是用 ResourceWrapper 决定的。
>
> **ProcessorSlotCahin 和 ResourceWrapper 绑定。**







> 在 ProcessorSlotChain 中参与统计的节点和对应维度：

首先还是强调 ProcessorSlotChain 和 ResouceWrapper 绑定，一一对应。

1. ContextName  -  在 NodeSelectorSlot 中创建，ContextUtil#entry 中指定 `ContextUtil#entry(name)`，节点类型为 EntranceNode。
2. ResourceWrapper  -  在 ClusterBuilderSlot 中创建，相同资源的统计节点全局共享，节点类型为 ClusterNode。
3. Origin  -  在 ClusterBuilderSlot 中创建，ContextUtil#entry 中指定 `ContextUtil#entry(name,.origin)`，节点类型为 StatisticNode，可以为空。

> OriginNode 在 ClusterNode 中创建，也就是说 ClusterNode 中（同个资源的统计节点中）会根据不同的 origin 进行细分。

## 参考

- [Sentinel 官方文档](https://github.com/alibaba/Sentinel/wiki/%E4%BB%8B%E7%BB%8D)
- [Sentinel源码解析一（流程总览）](https://www.cnblogs.com/taromilk/p/11750962.html)

- [Sentinel 教程](https://gitee.com/all_4_you/sentinel-tutorial#/all_4_you/sentinel-tutorial/blob/master/sentinel-practice/sentinel-flow-control/sentinel-flow-control.md)

