# Stream

## Stream 的作用

Stream 是 Java8 提供的一种流式操作 API，除了结合 Lambda 表达式可以提供良好的阅读性之外，Stream 还为集合操作提供了另外一种形式。

首先要明确的是 Stream 并不是层层的遍历，可以借助以下代码理解：

```java
int[] nums = new int[]{9, -7, 1, -4, 2, -5, 12, 6};

// 针对 nums，取绝对值，过滤并且排序
final int[] afterOP = Arrays.stream(nums)
    .map(Math::abs)
    .filter(a -> a > 3)
    .sorted()
    .toArray()
    
 // 直接逐层操作遍历
for (int i = 0; i < nums.length; i++) {
     nums[i] = Math.abs(nums[i]);
}
List<Integer> afterFilter = new ArrayList<>();
for (int num : nums) {
    if (num > 3) {
        afterFilter.add(num);
    }
}
Collections.sort(afterFilter);
int[] ans = new int[afterFilter.size()];
for (int i = 0; i < afterFilter.size(); i++) {
    ans[i] = afterFilter.get(i);
}
// 逐层遍历会有很多的多余操作，并且中间的数据存储也会非常麻烦。

// 整合中间操作
// 总的来说，首段的 Stream 代码其实相当于以下逻辑：
List<Integer> afterFilter = new ArrayList<>();
for (int num : nums) {
    num = Math.abs(num);
    if (num > 3) {
        afterFilter.add(num);
    }
}
Collections.sort(afterFilter);
int[] ans = new int[afterFilter.size()];
for (int i = 0; i < afterFilter.size(); i++) {
    ans[i] = afterFilter.get(i);
}
```

Stream 的使用随着中间操作的变多以及数据量地增加会有明显的性能优势。

<br>

> 针对以上的代码实现就会出现一些问题：
>
> 1. Stream 如何组织源数据
> 2. Stream 采用流式操作如何连接各个中间操作
> 3. Stream 如何将数据汇总到

## Stream 相关操作 API

Stream 中的所有操作分为**中间操作和终止操作**。

中间操作可以细分为**有状态和无状态的**，终止操作可以分为**短路和非短路的**。

细分如下表：

| Stream操作分类                    | 分类                       |                                                              |
| --------------------------------- | -------------------------- | ------------------------------------------------------------ |
| 中间操作(Intermediate operations) | 无状态(Stateless)          | unordered() filter() map() mapToInt() mapToLong() mapToDouble() flatMap() flatMapToInt() flatMapToLong() flatMapToDouble() peek() |
|                                   | 有状态(Stateful)           | distinct() sorted() sorted() limit() skip()                  |
| 结束操作(Terminal operations)     | 非短路操作                 | forEach() forEachOrdered() toArray() reduce() collect() max() min() count() |
|                                   | 短路操作(short-circuiting) | anyMatch() allMatch() noneMatch() findFirst() findAny()      |

> 以上方法都能在 Stream 接口中找到，并在 ReferencePipeline 中提供了基本实现，另外的 AbstractPipeline 定义了每个流水线节点的基本结构。

从最开始的生成 Stream，添加中间操作都不会触发操作的执行，**只有终止操作才会触发执行所有的操作**，这也是 Stream 的**延迟执行**特性。

<br>



## Sink 

Stream 中的每一个操作都会被封装为一个 Sink。

Sink 继承于 Consumer，并且定义了几个用于上下 Stage 串联的方法。

| 方法名                          | 作用                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| void begin(long size)           | 开始遍历元素之前调用该方法，通知Sink做好准备，例如 sort 会向下传递元素的个数，而 map 则是个空方法，filter 不确定数据大小所以传递的是个 -1。 |
| void end()                      | 所有元素遍历完成之后调用，通知Sink没有更多的元素了。         |
| boolean cancellationRequested() | 是否可以结束操作，可以让短路操作尽早结束。                   |
| void accept(T t)                | 遍历元素时调用，接受一个待处理元素，并对元素进行处理。Stage把自己包含的操作和回调方法封装到该方法里，前一个Stage只需要调用当前Stage.accept(T t)方法就行了。 |

**Stream 就是以 Sink 的四个方法串联起整个执行链路。**

参考 Sink 的实现可能更好的理解：

```java
/**
  * {@link Sink} for implementing sort on SIZED reference streams.
   */
private static final class SizedRefSortingSink<T> extends AbstractRefSortingSink<T> {
    private T[] array;
    private int offset;

    SizedRefSortingSink(Sink<? super T> sink, Comparator<? super T> comparator) {
        super(sink, comparator);
    }

    @Override
    @SuppressWarnings("unchecked")
    public void begin(long size) {
        if (size >= Nodes.MAX_ARRAY_SIZE)
            throw new IllegalArgumentException(Nodes.BAD_SIZE);
        array = (T[]) new Object[(int) size];
    }

    @Override
    public void end() {
        Arrays.sort(array, 0, offset, comparator);
        downstream.begin(offset);
        if (!cancellationWasRequested) {
            for (int i = 0; i < offset; i++)
                downstream.accept(array[i]);
        }
        else {
            for (int i = 0; i < offset && !downstream.cancellationRequested(); i++)
                downstream.accept(array[i]);
        }
        downstream.end();
        array = null;
    }

    @Override
    public void accept(T t) {
        array[offset++] = t;
    }
}
```

ReferencePipeline 中声明了 Head 类，表示的就是整个流水线的头节点，存放了原始数据。

另外还有 StatelessOp 以及 StatefulOp 类，分别表示无状态的操作以及有状态的操作。

而结束操作一般直接定义在 ReferencePipeline 内部方法里。

