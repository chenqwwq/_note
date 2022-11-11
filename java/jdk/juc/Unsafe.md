# Unsafe 

---

[TOC]

---



## 概述

Unsafe 属于 sum.misc 包下的工具类，提供一些低级别，不安全的操作。

![img](assets/f182555953e29cec76497ebaec526fd1297846.png)

（感谢美团的图



Unsafe 中的一些方法涉及的底层知识太多，好像并不能很好的掌握，就先看个大概吧。

// TODO 方法详察



## 使用示例

Unsafe 在普通权限的类中是无法通过 Unsafe#getUnsafe 方法获取的，因为会有 Domain 的判断。

```java
@CallerSensitive
public static Unsafe getUnsafe() {
  // 获取调用方类型
  Class<?> caller = Reflection.getCallerClass();
  // 是否是系统域
  if (!VM.isSystemDomainLoader(caller.getClassLoader()))
    throw new SecurityException("Unsafe");
  return theUnsafe;
}
```

<br>

因此需要一些骚操作才可以使用，具体如下：

```java
static class Demo {
  private int val = 0;

  // static
  private static final Unsafe unsafe;
  private static final long valOffset;

  static {
    try {
      // 通过反射获取内部变量
      Field field = Unsafe.class.getDeclaredField("theUnsafe");
      field.setAccessible(true);
      unsafe = (Unsafe) field.get(null);
      // 获取偏移量
      valOffset = unsafe.objectFieldOffset(Demo.class.getDeclaredField("val"));
    } catch (Exception e) {
      throw new Error(e);
    }
  }

  public void putVal(int val) {
    unsafe.putInt(this, valOffset, val);
  }

  public int getVal() {
    return val;
  }
}
```





## CAS

| 方法名 | 方法作用 |
| :----- | :------- |
|        |          |
|        |          |

## 内存相关



## 数组相关

| 方法名                                | 方法作用                     |
| :------------------------------------ | :--------------------------- |
| arrayBaseOffset(Class<?> arrayClass)  | 获取数组第一个元素的偏移地址 |
| arrayIndexScale(Class<?> arrayClass); | 获取单个元素的偏移           |

（Unsafe 中另外可以使用 Unsafe.ARRAY_LONG_BASE_OFFSET 和 Unsafe.ARRAY_INT_INDEX_SCALE 保存各类型的偏移。

```java
int[] nums = new int[10];
// 这里是 Disruptor 的工具类，用来获取 Unsafe 的
Unsafe unsafe = Util.getUnsafe();
// 获取数组头节点的偏移量
int offset = Unsafe.ARRAY_LONG_BASE_OFFSET;
// 获取单个节点的偏移量
int scale = Unsafe.ARRAY_INT_INDEX_SCALE;
// 设置第三个元素为2
unsafe.putInt(nums, offset + 3L * scale, 2);
// print: 0 0 0 2 0 0 0 0 0 0 
```







## 线程调度相关



## Class 操作

| 方法名                           | 方法作用                                                     |
| :------------------------------- | :----------------------------------------------------------- |
| allocateInstance(Class<?> clazz) | 绕过构造函数创建对象（`<clinit> `还是会执行，`<init>` 不会执行 |
|                                  |                                                              |



## 对象操作

涉及的方法如下：

| 方法名                                             | 方法作用                                                     |
| :------------------------------------------------- | :----------------------------------------------------------- |
| objectFieldOffset(Field field)                     | 获取成员变量相对于对象内存的偏移量                           |
| getObject(Object o, Long offset)                   | 获取指定偏移量的值                                           |
| putObject(Object o, Long offset, Object x);        | 设置指定偏移量的值                                           |
| getObjectVolatile(Object o, long offset)           | 按照 volatile 的模式的加载                                   |
| putObjectVolatile(Object o, long offset, Object x) | 按照 volatile 的模式的设置                                   |
| putOrderedObject(Object o, long offset, Object x)  | 半 volatile 模式的写入，不保证可见性的设置，但保证不会被之前的写入操作覆盖 |
|                                                    |                                                              |

### 对 putOrdered 的理解

> 对JVM 定义的 Memory Barrier 的理解：
>
> 一共存在 StoreStore、StoreLoad、LoadLoad、LoadStore 四种 Barrier。
>
> StoreStore 表示前面的在执行 Store（写操作）的时候保证前面的 Store（写操作）都完成。
>
> StoreLoad 表示前面的在 Load 的时候前面的 Store 都已经完成。
>
> LoadLoad 表示在后续的 Load 之前，前面的 Load 被读取完毕（这里有点不懂，都是 Load 有啥关系。
>
> LoadStore 表示遇到 Store 的时候保证前面的 Load 操作都完成。

对于 volatile 的写操作会在前面添加 StoreStore 屏障，写操作后面添加 StoreLoad 屏障操作。

对于 volatile 的读操作会在后面添加 LoadLaod、LoadStore 屏障。

但是对于 putOrdered 的操作（包括 putOrderedObject、putOrderedInt、putOrderedLong），都只会在前面添加 StoreStore，但不会在后面添加 StoreLoad。

因此此次写操作保证不会因为重排序被前面的写操作覆盖，但是后续的读取并不保证是最新值。









## 内存屏障

涉及的方法如下：

| 方法名     | 方法作用                                                     |
| ---------- | ------------------------------------------------------------ |
| loadFence  | 相当于 LoadLoad、LoadStore 屏障，保证前面的 Load 操作都是完成的 |
| storeFence | 相当于 StoreLoad、StoreStore 屏障，保证前面的 Store 操作都是完成的 |
| fullFence  | 全屏障，相当于 volatile                                      |





## Reference

- [Java魔法类：Unsafe应用解析](https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html)



