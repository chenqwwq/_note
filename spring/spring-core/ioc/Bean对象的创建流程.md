# Spring 中 Bean 对象的创建流程

---

[TOC]

---



## 概述

Spring 将 Bean 对象的创建分为了两大块内容:

1. 实例化 - 根据 BeanDefinition 创建具体的实例对象
2. 初始化 - 对创建的实例对象进行一系列的配置

在以上两个步骤中再穿插着各类 BeanPostProcessor 的调用，就组成了 Bean 对象基本的创建流程。



## 