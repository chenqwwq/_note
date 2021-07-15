# Stream

> Java8 提供了流式操作类。



## 相关概念

### Stream 相关操作 API

Stream 中的所有操作分为**中间操作和终止操作**。

中间操作可以细分为**有状态和无状态的**，终止操作可以分为**短路和非短路的**。

细分如下表：

| Stream操作分类                    | 分类                       |                                                              |
| --------------------------------- | -------------------------- | ------------------------------------------------------------ |
| 中间操作(Intermediate operations) | 无状态(Stateless)          | unordered() filter() map() mapToInt() mapToLong() mapToDouble() flatMap() flatMapToInt() flatMapToLong() flatMapToDouble() peek() |
|                                   | 有状态(Stateful)           | distinct() sorted() sorted() limit() skip()                  |
| 结束操作(Terminal operations)     | 非短路操作                 | forEach() forEachOrdered() toArray() reduce() collect() max() min() count() |
|                                   | 短路操作(short-circuiting) | anyMatch() allMatch() noneMatch() findFirst() findAny()      |

> 以上方法都能在 Stream 接口中找到，并在 ReferencePipeline 中提供了基本实现。

<br>

### Sink 

Stream 中的每一个操作都会被封装为一个 Sink。

Sink 继承于 Consumer，并且定义了几个用于上下 Stage 串联的方法。