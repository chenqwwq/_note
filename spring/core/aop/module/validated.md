# Validated

---

[TOC]

---

## 概述

Spring 提供的方法级别参数校验，使用 AOP 实现。

由 MethodValidationPostProcessor（继承了 BeanPostProcessor）拦截实例对象的创建流程，并织入