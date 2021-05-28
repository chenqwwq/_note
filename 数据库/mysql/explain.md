# Explain 分析



> Explain 是 MySQL 中常规的 SQL 解析工具，能展示出SQL的部分执行逻辑和过程。
>
> 分析 Explain 的输出就能帮助我们优化和改进 SQL 语句。

---

[TOC]

---



## 





## Extra

### using where

过滤元组， 不代表读取数据文件或索引文件



### using index

只需要读取索引文件就可以获取全部的数据，而不需要读取数据文件。

表示不需要进行回表。	