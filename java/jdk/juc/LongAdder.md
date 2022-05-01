# LongAdder

---

[TOC]

---

## Introduce

LongAdder 是对 Atomic 的优化，更适用于高并发情况下的累加器实现，基本原理还是基于 CAS 实现并发安全的加减操作。

Atomic 使用 CAS + 自旋来保证累加的正确性，但是针对于同一个共享变量的频繁修改会有如下问题：

1. 大并发情况下，争用激烈会导致部分线程长时间自旋
2. 对单个共享变量的修改会导致缓存行频繁失效



LongAdder 继承了 Striped64 类，在 Striped64 中使用**数组分流**的方式解决了第一个问题（emm，缓存行的问题可能影响不大吧。



## Striped64

在该类中主要包含以下两种属性：

1. base - 基本的计数变量，在没有线程争用的情况下 base 就可以实现并发安全的自增
2. cells - 自增数组，每个单元都是一个 Cell 对象，会被分配给不同的线程各自递增

另外还有一个锁变量 cellsBusy（对于该变量的争用就是上锁。



Striped64 中另外提供了对 base 的 CAS 方法：

```java
/**
 * CASes the base field.
 */
final boolean casBase(long cmp, long val) {
  // 同样是采用 Unsafe 方法
  return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
}
```



### Cell 

Cell 就是基础的自增类。

```java
static final class Cell {
  // 自增变量
  volatile long value;
  // 初始化
  Cell(long x) { value = x; }
  // cas 自增
  final boolean cas(long cmp, long val) {
    return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
  }

  // Unsafe 相关
  private static final sun.misc.Unsafe UNSAFE;
  private static final long valueOffset;
  static {
    try {
      UNSAFE = sun.misc.Unsafe.getUnsafe();
      Class<?> ak = Cell.class;
      valueOffset = UNSAFE.objectFieldOffset
        (ak.getDeclaredField("value"));
    } catch (Exception e) {
      throw new Error(e);
    }
  }
}
```

## 增加计数



### LongAdder#add

```java
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    // cell 数组不为空或者 cas 替换 base 失败
    // 所以在 cells 数组不为空的时候就不会使用 base 基础计数
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        // 没有竞争
        boolean uncontended = true;
        // as 为空，说明是 cas 失败
        if (as == null 
            // as 长度为0
            || (m = as.length - 1) < 0 
            // getProbe 表示当前线程的 Hash 值，根据 Hash 计算对应的桶信息
            || (a = as[getProbe() & m]) == null 
            || !(uncontended = a.cas(v = a.value, v + x)))
            longAccumulate(x, null, uncontended);
    }
}
```

<br>



```java

final void longAccumulate(long x, LongBinaryOperator fn,
                          boolean wasUncontended) {
  int h;
  // 初始化 ThreadLocalRandom
  // getProbe 获取的是线程中的探测 Hash 值，如果等于0表示此事 ThreadLocalRandom 未初始化
  if ((h = getProbe()) == 0) {
    // 初始化 ThreadLocalRandom
    ThreadLocalRandom.current(); // force initialization
    h = getProbe();
    wasUncontended = true;
  }
  boolean collide = false;                // True if last slot nonempty
  for (;;) {
    Cell[] as; Cell a; int n; long v;
    // Cell 数组已经初始化
    if ((as = cells) != null && (n = as.length) > 0) {
      // 根据线程的 Probe 获取对应的 cell 下标
      // 为空表示没有初始化
      if ((a = as[(n - 1) & h]) == null) {
        if (cellsBusy == 0) {       // Try to attach new Cell
          Cell r = new Cell(x);   // Optimistically create
          // 通过 cas CellsBusy 上锁
          if (cellsBusy == 0 && casCellsBusy()) {
            boolean created = false;
            try {               // Recheck under lock
              Cell[] rs; int m, j;
              // 创建好的对象塞会 cells 对象
              if ((rs = cells) != null &&
                  (m = rs.length) > 0 &&
                  rs[j = (m - 1) & h] == null) {
                rs[j] = r;
                created = true;
              }
            } finally {
              // 解锁（这个解锁是真的简单
              cellsBusy = 0;
            }
            // 创建成功就退出了
            if (created)
              break;
            continue;           // Slot is now non-empty
          }
        }
        collide = false;
      }
      else if (!wasUncontended)       // CAS already known to fail
        wasUncontended = true;      // Continue after rehash
      else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                   fn.applyAsLong(v, x))))
        break;
      else if (n >= NCPU || cells != as)
        collide = false;            // At max size or stale
      else if (!collide)
        collide = true;
      else if (cellsBusy == 0 && casCellsBusy()) {
        // 扩容过程
        try {
          if (cells == as) {      // Expand table unless stale
            Cell[] rs = new Cell[n << 1];
            for (int i = 0; i < n; ++i)
              rs[i] = as[i];
            cells = rs;
          }
        } finally {
          cellsBusy = 0;
        }
        collide = false;
        continue;                   // Retry with expanded table
      }
      h = advanceProbe(h);
    } else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
      // 初始化为2
      boolean init = false;
      try {                           // Initialize table
        if (cells == as) {
          // 创建 cell，赋值并且将其塞入 cell
          Cell[] rs = new Cell[2];
          rs[h & 1] = new Cell(x);
          cells = rs;
          init = true;
        }
      } finally {
        cellsBusy = 0;
      }
      if (init)
        break;
    } else if (casBase(v = base, ((fn == null) ? v + x :
                                  fn.applyAsLong(v, x))))
      break;                          // Fall back on using base
  }


```





cells 数组长度初始为 2，每次递增为2次幂，使用 Thread#threadLocalRandomProbe 属性作为 Hash 值确定下标。



## 获取统计值

```java
public long sum() {
  Cell[] as = cells; Cell a;
  // 基础的 base 值
  long sum = base;
  if (as != null) {
    for (int i = 0; i < as.length; ++i) {
      if ((a = as[i]) != null)
        // 加上所有的 value
        sum += a.value;
    }
  }
  return sum;
}
```

LongAdder 的统计总和等于基础值 base 加上 cells 数组的各个值。