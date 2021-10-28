# Sentinel 基础流程

---

[TOC]

---



## 概述

Sentinel 将每个调用都可以抽象为一个资源的获取，获取成功则返回 Entry，获取失败则抛出异常或返回 false。





## 创建调用上下文



