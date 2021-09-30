# Netty 中写数据的逻辑

> 以 4.1.53.Final-SNAPSHOT 中 Nio 相关封装为例。



---

[TOC]

---

## 概述

Netty 中的写事件由两个流程组成：

- write - 将数据添加到写缓存
- flush - 刷新写缓存，发送数据到对端



## 

