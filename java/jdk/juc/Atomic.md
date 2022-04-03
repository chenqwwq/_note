# Atomic Class

---

[TOC]

---

## 概述

原子类是 JDK8 中推出的，基于 CAS 实现的线程安全的类型。

JUC 包中定义的原子类有如下几种类型：

基本类型： AtomicInteger、AtomicLong、AtomicBoolean

引用类型： AtomicReference、AtomicStampedReference、AtomicMarkableReference

数组类型： AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray

对象属性类型：AtomicIntegerFieldUpdater、AtomicLongFieldUpdater、AtomicReferenceFieldUpdater

累加器：DoubleAccumulator、DoubleAdder、LongAccumulator、LongAdder、Striped64





## AtomicInteger（原子 Integer 类型

AtomicLong，AtomicBoolean 都类似，使用 CAS 保证线程安全性。

CAS 的实现就是 unsafe#compareAndSwap 方法，例如包括递增方法都是这样实现的。

```java
// AtomicInteger#incrementAndGet
public final int incrementAndGet() {
  // 直接调用了 unsafe#getAndAddInt 方法
  // 该方法获取的是原始值，所以此时还需要额外+1
  return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```

可以在简单看一下 

```java
// Unsafe#getAndAddInt
public final int getAndAddInt(Object o, long offset, int delta) {
  int v;
  // 整体就是循环的 CAS 操作
  do {
    v = getIntVolatile(o, offset);
    // compareAndSwapInt 直接就是一个 native 方法
  } while (!compareAndSwapInt(o, offset, v, v + delta));
  return v;
}
```





## Striped64

该类也是