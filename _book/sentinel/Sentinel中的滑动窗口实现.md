# Sentinel中的滑动窗口实现



## 概述

Sentinel 中的统计使用滑动窗口来保证其平滑，最基础的实现就是 LeapArray（以分析 LeapArray 的实现实现为主。





## 窗口结构

Sentinel 中实现的 LeapArray 作为滑动窗口的基本实现（保存单窗口数组。

![image-20211008235941152](assets/image-20211008235941152.png)<br>

直接从初始化开始看，LeapArray 的构造函数如下：

![image-20211009001413559](assets/image-20211009001413559.png)

参数含义如下：

| 参数             | 含义                        |
| ---------------- | --------------------------- |
| windowLengthInMs | 每个桶代表的时间            |
| sampleCount      | 桶的数目                    |
| intervalInMs     | 整个 LeapArray 代表时间跨度 |
| array            | 整个窗口数组                |

参数中指定了窗口数和整个数组代表的毫秒数，所以直接使用 intervalInMs / sampleCount 计算出了每个窗口的时间。

另外 LeapArray 中的数组实现使用了 AtomicReferenceArray，并且以 WindowWrap 作为桶。

以下是 WindowWrap 的变量定义：

<img src="assets/image-20211009002124134.png" alt="image-20211009002124134" style="zoom:80%;" />

| 参数             | 含义             |
| ---------------- | ---------------- |
| windowLengthInMs | 窗口代表的毫秒数 |
| windowStart      | 窗口开始时间     |
| value            | 窗口中的数据     |



> **整个的滑动窗口结构就是以 WindowWrap 作为桶（单个窗口）的 AtomicReferenceArray 数组。**

## 获取当前窗口

根据当前的时间戳获取对应的窗口对象，直接调用的就是 LeapArray#currentWindow 方法：

![image-20211009001047575](assets/image-20211009001047575.png)

（TimeUtil.currentTimeMillis 好像也有蛮复杂的实现，但是此处可以简单当做 System。

之后的完整逻辑如下：

```java
public WindowWrap<T> currentWindow(long timeMillis) {
   if (timeMillis < 0) {
      return null;
   }
   // 求当前时间戳对应的下标
   // 当前时间除以窗口时间，并对窗口个数取模
   int idx = calculateTimeIdx(timeMillis);
   // 计算窗口开始时间
    // 对于窗口时间取整，例如 10410 对于 100 取整为 10400
   // important: 其实就是窗口时间的规整，按照当前的窗口大小来
   long windowStart = calculateWindowStart(timeMillis);

   /*
    * Get bucket item at given time from the array.
    *
    * (1) Bucket is absent, then just create a new bucket and CAS update to circular array.
    * 窗口不存在
    * (2) Bucket is up-to-date, then just return the bucket.
    * 窗口存在，且时间正确
    * (3) Bucket is deprecated, then reset current bucket and clean all deprecated buckets.
    * 窗口存在，但开始时间以过期
    */
   while (true) {
       // 通过下标获取窗口
      WindowWrap<T> old = array.get(idx);
      if (old == null) {
         // 旧窗口为空
         /*
          *     B0       B1      B2    NULL      B4
          * ||_______|_______|_______|_______|_______||___
          * 200     400     600     800     1000    1200  timestamp
          *                             ^
          *                          time=888
          *            bucket is empty, so create new and update
          *
          * If the old bucket is absent, then we create a new bucket at {@code windowStart},
          * then try to update circular array via a CAS operation. Only one thread can
          * succeed to update, while other threads yield its time slice.
          */
          // 新建窗口
         WindowWrap<T> window = new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
         if (array.compareAndSet(idx, null, window)) {
            // Successfully updated, return the created bucket.
            return window;
         } else {
            // Contention failed, the thread will yield its time slice to wait for bucket available.
            Thread.yield();
         }
      } else if (windowStart == old.windowStart()) {
         // 窗口可以费用
         /*
          *     B0       B1      B2     B3      B4
          * ||_______|_______|_______|_______|_______||___
          * 200     400     600     800     1000    1200  timestamp
          *                             ^
          *                          time=888
          *            startTime of Bucket 3: 800, so it's up-to-date
          *
          * If current {@code windowStart} is equal to the start timestamp of old bucket,
          * that means the time is within the bucket, so directly return the bucket.
          */
         return old;
      } else if (windowStart > old.windowStart()) {
         // 窗口过期
         /*
          *   (old)
          *             B0       B1      B2    NULL      B4
          * |_______||_______|_______|_______|_______|_______||___
          * ...    1200     1400    1600    1800    2000    2200  timestamp
          *                              ^
          *                           time=1676
          *          startTime of Bucket 2: 400, deprecated, should be reset
          *
          * If the start timestamp of old bucket is behind provided time, that means
          * the bucket is deprecated. We have to reset the bucket to current {@code windowStart}.
          * Note that the reset and clean-up operations are hard to be atomic,
          * so we need a update lock to guarantee the correctness of bucket update.
          *
          * The update lock is conditional (tiny scope) and will take effect only when
          * bucket is deprecated, so in most cases it won't lead to performance loss.
          */
         if (updateLock.tryLock()) {
            try {
               // Successfully get the update lock, now we reset the bucket.
                // 重置窗口
               return resetWindowTo(old, windowStart);
            } finally {
               updateLock.unlock();
            }
         } else {
            // Contention failed, the thread will yield its time slice to wait for bucket available.
            Thread.yield();
         }
      } else if (windowStart < old.windowStart()) {
         // Should not go through here, as the provided time is already behind.
         return new WindowWrap<T>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
      }
   }
}
```

