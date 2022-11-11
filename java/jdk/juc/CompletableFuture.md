# CompletableFuture

---

[TOC]

---

## Introduction

CompletableFuture 是对 Future 接口的扩展（这非常重要，曾经有人让我对比过这两个东西，gtnd。

Future 在 JDK 中表示一个异步任务的结果，通过 Future 可以阻塞获取异步返回值或者取消异步任务，例如 FutureTask 的实现。

但是 Future 存在以下几种问题：

- get 方式只有有阻塞/超时阻塞两种
- 只能表示单个任务无法编排
- 缺少回调钩子（ Netty 的 Future 实现了 Future 的钩子方法的执行



## Usage Case（Method List

相关方法和作用如下：

| 静态方法名                                                   | 作用                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| allOf(CompletableFuture<?>... cfs)                           | 编排多个 CompletableFuture，若掉用 get/join 任务全部完成之后才会返回，无返回值 |
| anyOf(CompletableFuture<?>... cfs)                           | 任意任务完成就会返回，并返回对应的返回值                     |
| supplyAsync(Supplier<U> supplier)<br />supplyAsync(Supplier<U> supplier,Executor executor) | 异步执行 Supplier，返回一个包含返回值的 CompletableFuture<U> |
| runAsync(Runnable runnable)<br />runAsync(Runnable runnable, Executor executor) | 异步执行的任务                                               |
| completedFuture(U value)                                     | 返回一个带指定返回值的 CompletableFuture                     |

**以上方法都是静态的可以用于初始化一个 CompletableFuture **

（Consumer 和 Function 都是单入参的，但是 Function 包含返回值

```java
// 等待所有任务执行完毕
ExecutorService executorService = Executors.newFixedThreadPool(10);
CompletableFuture<Void> future = CompletableFuture.allOf(CompletableFuture.runAsync(() -> {
  // DO SOMETHING
}), CompletableFuture.runAsync(() -> {
  // DO OTHER THINGS
}, executorService));
future.get();
```

**以下方法是实例方法，是对于一个已经存在的 CompletableFuture 添加钩子方法（主体任务执行到一定阶段才会执行的**

| 实例方法名                                                   | 作用                                                |
| ------------------------------------------------------------ | --------------------------------------------------- |
| thenApply(     Function<? super T,? extends U> fn)           | 添加同步执行的  Function 方法                       |
| thenApplyAsync(Function<? super T,? extends U> fn)<br />thenApplyAsync(Function<? super T,? extends U> fn, Executor executor) | 添加异步执行的  Function 方法，可以指定执行的线程池 |
| thenAccept(Consumer<? super T> action)                       | 添加 Consumer 方法                                  |
| thenAcceptAsync(Consumer<? super T> action)  <br />thenAcceptAsync(Consumer<? super T> action, Executor executor) | 添加异步执行的 Consumer 方法，可以进一步指定线程池  |
| thenRunnabl                                                  |                                                     |
|                                                              |                                                     |



**获取结果的方法：**



| 方法名                                                      | 作用                                                         |
| ----------------------------------------------------------- | ------------------------------------------------------------ |
| get() <br />get(long timeout, TimeUnit unit)                | 获取或者超时获取，成功前阻塞                                 |
| join()                                                      | 和 get 类似，不同的是 join 不响应中断                        |
| getNow(T valueIfAbsent)                                     | 获取当前结果，没有可以填充结果，为 null 就是不塞             |
| complete(T value) <br />completeExceptionally(Throwable ex) | 任务尚未完成时，填充值或者异常，另外的线程可以在 get 方法中获取到填充的 |

## CompletionStage

该接口表示异步计算中的一个步骤，在一个步骤完成后可以触发另外一个步骤，所以该接口就是用来编排各个异步的流程都（以流式调用的形式。







## Reference

- [CompletableFuture原理与实践-外卖商家端API的异步化](https://mp.weixin.qq.com/s/GQGidprakfticYnbVYVYGQ)