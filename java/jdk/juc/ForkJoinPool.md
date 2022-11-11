# ForkJoinPool

---

[TOC]

---



## Introduction

> 没有完整的看，暂时整理一些博客上的概念和逻辑。

forkJoinPool 中每个线程都被包装成一个 Worker，每个 Worker 都会绑定两个 WorkQueue，奇数的保存当前线程 fork 出来的任务，偶数保存外部提交的任务。

绑定的线程根据线程的 Probe 计算（Probe 是 ThreadLocalRandom 的概念，初始为0，在第一次调用时候初始化，可以递进）。

WokeQueue 是双端队列，支持 LIFO 以及 FIFO，LIFO 为当前线程执行的时候，FIFO 为别的线程窃取的时候调用，所以当前线程的任务执行不需要上锁，窃取的时候需要上锁。

