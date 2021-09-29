# ProcessorSlot 相关

---

[TOC]

---



##  概述

Sentinel 实现了





## ProcessorSlotChain

从 Ctspn 中获取的 ProcessorSlotChain，缓存在 static 的 chainMap 中，以 ResourceWrapper 作为 Key。

> 所以全局范围内，对于相同的 ResourceWrapper 获取的都是同个 ProcessorSlotChain。

使用 ProcessorSlotProvider 创建，内部又包含了 SlotChainBuilder（可以通过  SpiLoader 加载），Builder 中又包含了 ProcessorSlot 的创建（同样可以通过 SpiLoader 加载）。

> SlotChainBuilder 和 ProcessorSlot 都可以通过 SpiLoader 做自定义配置。

<br>

默认加载到的 ProcessorSlot 如下：

![image-20210929164819248](./ProcessorSlot%E7%9B%B8%E5%85%B3.assets/image-20210929164819248.png)





## NodeSelectorSlot 





## ClusterBuilderSlot

![image-20210920172058810](assets/image-20210920172058810.png)

该槽位保存了资源（resource）的运行统计信息，包括：

- 响应时间（response time ）
- QPS
- 线程数
- 异常
- 由 ContextUtil#enter 标记的调用方列表

每个资源共用一个全局的 Cluster Node。







## StatisticSlot 

![image-20210920172837782](assets/image-20210920172837782.png)

用于实时统计的处理器槽位，





## AuthoritySlot

校验权限的处理槽位，通过 AuthorityRule 来检查权限。

通过 AuthorityRuleManager 获取所有需要检查的 AuthorityRule。



## SystemSlot

通过 SystemRule 来检查相关内容。







## FlowSlot

流量控制相关，相关作用如下：

- Traffic Shapping
- Warmup
- Immediately reject
- Uniform Rate Limiting







## DegradeSlot 

请求降级相关处理槽位。







## 参考

- [Sentinel 官方文档](https://github.com/alibaba/Sentinel/wiki/Sentinel%E5%B7%A5%E4%BD%9C%E4%B8%BB%E6%B5%81%E7%A8%8B)