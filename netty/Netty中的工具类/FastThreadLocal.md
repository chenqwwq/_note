# FastThreadLocal及其相关

> FastThreadLocal相关源码阅读

## 概述

原生的ThreadLocal主要原理就是在Thread对象上个上绑定的一个ThreadLocalMap结构。

每个线程都会从各自的线程中获取这个Map，并以ThreadLocal对象为Key，所以每个线程间的变量不会冲突。

总得来说，也就是空间换时间的概念，来解决共享变量的冲突。



FastThreadLocal并不是基于ThreadLocal的优化，而是直接重新实现了一种。

这里的重新实现是指新的数组组织形式，原来的ThreadLocal以一个Map的形式存储所有数据，而FastThreadLocal则是以数组的形式，数组良好的随机访问性能也就保证了FastThreadLocal的访问速度会优于ThreadLocal。



