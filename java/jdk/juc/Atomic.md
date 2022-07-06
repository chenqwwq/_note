# Atomic Class

---

[TOC]

---

## 概述

原子类是 JDK8 中推出的，基于 CAS 实现的线程安全的类型。

JUC 包中定义的原子类有如下几种类型：

基本类型： AtomicInteger、AtomicLong、AtomicBoolean

引用类型： AtomicReference、AtomicStampedReference、AtomicMarkableReference

对象属性类型：AtomicIntegerFieldUpdater、AtomicLongFieldUpdater、AtomicReferenceFieldUpdater

数组类型： AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray

累加器：DoubleAccumulator、DoubleAdder、LongAccumulator、LongAdder、Striped64





## Atomic 类（原子类

原子类最基础的实现 AtomicLong，AtomicBoolean 都类似，**使用 CAS 保证线程安全性。**

CAS 的实现就是 unsafe#compareAndSwap 方法，例如包括递增方法都是这样实现的。

```java
// AtomicInteger#incrementAndGet
public final int incrementAndGet() {
  // 直接调用了 unsafe#getAndAddInt 方法
  // 该方法获取的是原始值，所以此时还需要额外+1
  // unsafe.getAndAddInt(this, valueOffset, 1) 是直接在原值上递增
  return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```

（ Unsafe 相关的实现可以查看这篇 [Unsafe 实现](Unsafe.md)。

这是最基础的原子类的实现，核心还是 CAS 保证的线程安全，因为是简单的数据相加也不会出现 ABA 的问题。

在 JDK8 之后，根据 FunctionalInterface 又新增了几个方法：

| 方法名                                                       | 方法作用                                                   |
| :----------------------------------------------------------- | :--------------------------------------------------------- |
| getAndUpdate(IntUnaryOperator updateFunction)                | 调用 IntUnaryOperator#applyAsInt 计算当前值                |
| updateAndGet(IntUnaryOperator updateFunction)                | 和上述的区别就是该方法获取的是新值                         |
| int getAndAccumulate(int x,IntBinaryOperator accumulatorFunction) | 调用 IntBinaryOperator#applyAsInt，可以将当前值和x进行计算 |
| accumulateAndGet(int x,IntBinaryOperator accumulatorFunction) | 获取新值                                                   |

实现方法上类似，以 **getAndAccumulate** 为例：

```java
public final int getAndAccumulate(int x,IntBinaryOperator accumulatorFunction) {
    int prev, next;
    do {
        prev = get(); // 获取当前值
        next = accumulatorFunction.applyAsInt(prev, x); // 计算新值
    } while (!compareAndSet(prev, next)); // 替换
    return prev;
}
```

<br>

### 其他实现

实现上 AtomicLong、AtomicBoolean、AtomicReference 基本类似。

（还是 CAS + 替换，忽略 ABA 问题。

AtomicStampedReference 在 Reference 的基础上增加了 (int)Stamp 对象，相当于是增加了一个时间戳属性，避免了 ABA 问题。

```java
public boolean compareAndSet(V   expectedReference,  // 预期的原值
                             V   newReference, 			 // 新值
                             int expectedStamp,			 // 预期时间戳
                             int newStamp) {				 // 新时间戳，简单的还可以使用 AtomicInteger 自增作为 Stamp
    Pair<V> current = pair;
    return
      	// 对比一定要两个值都匹配才会替换 
        expectedReference == current.reference && 
        expectedStamp == current.stamp &&
        ((newReference == current.reference &&
          newStamp == current.stamp) ||
         // 使用封装的 Pair 进行统一替换
         casPair(current, Pair.of(newReference, newStamp)));
}
```

AtomicMarkableReference 则是增加了一个 (boolean)mark 对象来解决 ABA 问题。

（解决 ABA 问题的核心还是新增一个属性来表示当前对象的版本，如果对象不变可以使用 attemptStamp 单独更新 Mark 或者 Stamp。

<br>

AtomicIntegerFieldUpdater、AtomicLongFieldUpdater、AtomicReferenceFieldUpdater 属于是某个属性更新类。

属性更新类型可以通过 CAS 线程安全的更新类的某个属性，该类在创建的时候必须指定是某个 Class 类，以及类中的变量名。

（因为每个 Class 中变量的偏移量都不是相同的，所以针对不同 Class 类创建的对象不能共用。

数组相关的三个类型：AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray，在实现上也不净相同，只是更新的时候需要指定数组下标。

使用的也是 Unsafe 中数组相关的方法：

```java
public final int getAndAdd(int i, int delta) {
    // 不是原来的 CAS 方法了
    return unsafe.getAndAddInt(array, checkedByteOffset(i), delta);
}
```







## Accumulator / Adder 类（累加器类

该类是在累加的逻辑上对原子类的优化。

在多线程并发的情况下，LongAdder 性能远优于 AtomicLong，因为原子类在高并发的情况下，会有大量更新失败都需要通过长时间的自旋来保证更新到位，因此会有许多次无效更新。

（另外还有伪共享的问题，多线程修改同个变量的时候会导致缓存行的失效，导致每次都需要从主内存读取，但这不是主要问题。

<br>

**累加器类的主要优化就是分散冲突点**，这个逻辑在 Striped64 中实现，除了基础的 base 之外，还会创建一个 Cell 数组（一个 Cell 就是一个简单的 AtomicLong。

在累加 base 出现失败时候，会使用 ThreadLocalRandom 随机一个 Cell 进行累加，在统计的时候也是使用遍历累加统计。

<br>

详细的源码可以参考 [LongAdder](./LongAdder.md)

Adder 类专注于累加，完全忽略了操作的顺序性，