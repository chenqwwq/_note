# CyclicBarrier

---

[TOC]

---

## 概述

CyclicBarrier　是一个可以重用的同步辅助类，**允许一组线程相互等待**，在达到一个共同点在继续执行，并且可以置顶等待之后需要执行的程序。

使用示例如下：

```json
public class Main {
	public static void main(String[] args) throws InterruptedException {
		CyclicBarrier barrier = new CyclicBarrier(3, new Runnable() {
			@Override
			public void run() {
				System.out.println("start first command,[" + Thread.currentThread().getName() + "]");
			}
		});
		ExecutorService executorService = Executors.newFixedThreadPool(6);
		for (int i = 0; i < 6; i++) {
			executorService.submit(new Runnable() {
				@SneakyThrows
				@Override
				public void run() {
					System.out.println("start await,[" + Thread.currentThread().getName() + "]");
					barrier.await();
				}
			});
		}
		TimeUnit.SECONDS.sleep(10000);
	}
}
// 程序输出
//	start await,[pool-1-thread-1]
//	start await,[pool-1-thread-2]
//	start await,[pool-1-thread-3]
//	start first command,[pool-1-thread-3]
//	start await,[pool-1-thread-4]
//	start await,[pool-1-thread-5]
//	start await,[pool-1-thread-6]
//	start first command,[pool-1-thread-6]
```

**线程在调用 await 之后就会开始阻塞直到其他 n-1 个线程也调用该方法，并且在解除阻塞之后由最后一个 await 的调用线程执行指定的 barrierCommand。**

并且和 CountDownLatch 不同，CyclicBarrier 是可以重复使用的。

## 源码实现

CyclicBarrier 是基于 ReentrantLock 实现的。



### dowait 方法

```java

```

