# FastThreadLocal及其相关类源码分析

> FastThreadLocal相关源码阅读   -   2020.11.22

[TOC]

## ThreadLocal 回顾

回顾下 ThreadLocal 的实现原理。

首先 ThreadLocal，中文可以叫做线程局部变量，**以空间换时间的方式，将变量和线程绑定，不同的线程使用不同的变量，以此来解决共享变量的争用问题。**

这里的绑定形式就是在 **Thread 类中增加可以一个 ThreadLocalMap 的数据结构。**

这个数据结构是懒加载的，只有在第一次使用的时候初始化，是一个 Map 的实现，**以 ThreadLocal 对象作为 Key，具体数据作为 Value，使用开放寻址法解决 Hash 冲突。**

调用 ThreadLocal.get() 会获取当前的 Thread 对象并获取 ThreadLocalMap，此种实现就**保证了不同的线程使用的是不同的对象**。

有一个问题，就是内存泄露的问题，ThreadLocal 的内存泄露主要还是 Thread 对象和 ThreadLocal 对象的生命周期的不同<font size=2>（个人理解）</font>，Thread 对象需要当前的调用链结束，或者在线程池中 Thread 对象被回收，但在这之前可能 ThreadLocal 的对象已经被回收了，此时按照正常的 get 方法， ThreadLocalMap 中的对应数据已经是不可见了。

ThreadLocal 解决内存泄漏的方式很暴力，就是增加检查次数，时不时就检查一下，但是这个只是治标不治本，ThreadLocal 的回收和 ThreadLocalMap 中对应的 Value 的回收还是会有差异。

<br>

FastThreadLcoal 是对 ThreadLocal 的优化，也兼容 ThreadLocal 的使用方法。

ThreadLocal 需要和 ThreadLocalMap 以及 Thread 对象搭配使用。

FastThreadLocal 也需要和 FastThreadLocalThread 以及 InternalThreadLocalMap 搭配使用才能达到最大的优化效果。



## 概述

FastThreadLocal 的结构如下图所示:

![FastThreadLoca 的结构](assets/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6-1605978194604.png)

看上去也蛮简单的。

## Get方法

![image-20201121225244191](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201121225244191.png)

以上就是 FastThreadLocal#get 方法的全部代码了。

可以粗略的分为以下几步:

### 一、获取Map结构

首先通过 InternalThreadLocalMap 的 get 方法获取到具体的 Map 对象，类似于 ThreadLocal 中的 ThreadLocalMap，是一个和线程绑定的数据结构。

![image-20201121225351351](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201121225351351.png)

获取 InternalThreadLocalMap 的方法会通过当前的 Thread 类型来选择不同的获取方式。

这里的 ThreadLocal.currentThread 方法会获取当前的Thread对象，native方法没有深入看源码，但只要改变创建的线程对象的类型，这里就会发生改变。

> **例如使用 new MyThread().start() 启动线程获取到的 class 类型就是 MyThread 类型的。**

<br>

**fastGet 方法直接就是 FastThreadLocalThread 中获取了 Map 对象，如果 Map 对象为空则进行初始化，然后返回。**

在 FastThreadLocalThread 的源码中直接可以看到 InternalThreadLocalMap 的变量声明。

![image-20201121230705274](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201121230705274.png)

slowGet 方法麻烦一点，会通过 UnpaddedInternalThreadLocalMap.slowThreadLocalMap 这个静态变量来获取存储在 Thread 的数据，这个数据就是 InternalThreadLocalMap 对象。

![image-20201122010644770](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122010644770.png)



**也就是说如果是 FastThreadLocalThread 的线程对象，则是直接保存 InternalThreadLocalMap 对象，而如果是原生的 Thread 对象，InternalThreadLocalMap 就是它 ThreadLocalMap 中的一个 Key-Value。**

<br>

以上就能有个大概的结构了：

1. FastThreadLocalThread 和 Thread 对象类似，保存了一个 InternalThreadLocalMap 对象。
2. 如果当前线程不是 FastThreadLocalThread，则以原生的 ThreadLocal 形式，将 Map 对象作为 Value 存储。

<br>

整个Map的获取流程就是根据Thread对象类型分别从以上的两个途径获取。



###  二、获取具体的数据

从上图可知，具体的数据获取就是 InternalThreadLocalMap#indexedVariable 方法。

![image-20201121230531314](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201121230531314.png)

**indeedVariables 就是 Map 中底层的数据结构，就是一个 Object 数组。**

从这里就能看到 FastThreadLocal 和 ThreadLocal 数据结构好像并没有实质上的不同，都是数组。

ThreadLocal是通过类似Hash的方式组织结构的，需要先通过 Key 的 HashCode 计算标记下标，而FastThreadLocal直接是通过下标来确定的并获取数据的。

如果没有数据，返回的是UNSET对象。



这个下标（index）则是每个FastThreadLocal在创建时就指定的。

![image-20201121233247383](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201121233247383.png)

以上就是FastThreadLocal的构造函数，简单调用了InternalThreadLocalMap#nextVariableIndex方法。

![image-20201121233257404](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201121233257404.png)

这个生成index的方法更干脆就是一个原子类型的自增返回。

还有需要注意的，就是这个nextIndex的声明位置，nextIndex是继承自UnpaddedInternalThreadLocalMap的AtomicInteger类型的静态变量。

**也就是说所有的FastThreadLocal都共享一个index的生成方法。**

**每多创建一个FastThreadLocal对象index就加一，以index确定这个数据在底层数组中的位置。**

比如我新建一个FastThreadLocal，它的index是1，那么这个FastThreadLocal对应的value全部保存在Map的1号位置，在新建一个FastThreadLocal，新的index就是2了。





在get方法中如果没有获取到具体对象，则会进入初始化的流程。

### 三、数据初始化

以下是FastThreadLocal#initialize的具体源码:

![image-20201121231119042](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201121231119042.png)

initialValue是FastThreadLocal中的protected方法，等待子类实现，这里是一个模板模式的实现。

**initialValue返回初始化的值**，然后通过setIndexedVariable方法将值塞到map中。

之后还有一步addToVariabklesToRemove方法:

![image-20201121231337236](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201121231337236.png)

该方法会将当前的FastThreadLocal添加到ThreadLocalMap的一个特殊地方，就是数组头(0号位置)。

![image-20201122004223637](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122004223637.png)

因为这个variablesToRemoveIndex是FastThreadLocal中定义的静态变量，所有FastThreadLocal的对象及其子类共享，在加载FastThreadLocal的时候就赋值了。

**所有的FastThreadLocal如果在Thread中有对应的数据存在，都会将Key备份一份到数组位置**





那为什么要把所有的Key都放在数组的另外一个位置呢？

其实还是为了删除的方便，嗯。。从名字也能看出来，这个下文再说。



到这里获取的逻辑就走完了，流程如下:

1. 根据当前的Thread类型，以不同方式获取Map结构，没有则初始化
2. 根据index获取对应下标的数据，获取到具体的就直接返回
3. 没有获取到具体数据走初始化流程，调用子类实现的初始化方法，并将当前Key存放在当前Thread中Map的首元素。





## Set方法

![image-20201122004447942](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122004447942.png)

Set方法直接就分为两个部分，如果set的值不是UNSET的话就是正常的set逻辑，添加或者覆盖数据，而如果是UNSET的话，则删除当前值。



InternalThreadLocalMap#get方法就不在赘述了。

直接来到setKnowNoteUnset方法，该方法两个入参，就是把后面的添加到前面的Map中。

![image-20201122004640451](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122004640451.png)

该方法调用了Map的set方法，将值value保存到了Map结构中，保存成功之后将Key重新保存一下，

![image-20201122004832819](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122004832819.png)

该类的Set方法直接判断数组和入参index的大小，如果入参小于数组大小说明当前数组已经满足存储需求，直接将旧值覆盖。

如果不满足则开启扩容的分支代码，以下就是全部的扩容内容了。

![image-20201122005014977](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122005014977.png)

相对于HashMap来说，这里的扩容简单很多的，就是**声明一个两倍大的数组，然后将老数组的元素复制到新数组就可以了。**

扩容完之后再将数据元素放进数组。



上述是value不为UNSET，所以是增加或者覆盖的情况。

下面就是另外一个分支，**value为UNSET时的情况**，需要将执行删除逻辑。



这样子，Set的逻辑也算走完了，整个流程就是向Map填充对象的过程，中间考虑需要扩容的情况。

另外SET一个UNSET对象，在FastThreadLocal中就相当于删除元素。





## remove方法

删除当前线程上绑定的FastThreadLocal对应的对象。

![image-20201122171949559](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122171949559.png)

InternalThreadLocalMap.getIfSet()就是获取当前InternalThreadLocalMap对象的方法。

和InternalThreadLocalMap.get()的方法的区别就是get方法**如果为空会进行初始化**，但是getIfSet不会进行初始化而是直接返回null。

![image-20201122172000475](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122172000475.png)

首先调用Map的删除方法，删除当前FastThreadLocal的Index对应的元素。

![image-20201122172055146](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122172055146.png)

Map的删除逻辑非常简单，index小于我数组大小的就删，大于的就不删。

![image-20201122172154299](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122172154299.png)

除了删除Map中具体的数据，还需要删除removeSet中Key的缓存。

最后是调用FastThreadLocal的onRemoval方法。



到这里就结束了，总结来说就是分别执行以下三步:

1. 删除Map中的具体的值
2. 删除RemoveSet的Key
3. 调用FastThreadLocal的onRemoval方法。



除了删除单个的方法，FastThreadLocal还提供了清空的removeAll方法。

![image-20201122172808015](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122172808015.png)

首先明确这个方法的作用，并不是删除当前FastThreadLocal在不同线程的数据，而是删除当前线程上所有的FastThreadLocal的对象。

简单来看就是遍历RemoveSet，逐个执行上述的remove方法，最后执行InternalThreadLocalMap#remove方法。

![image-20201122172959637](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122172959637.png)

如果是FastThreadLocalThread则直接断开其上绑定的IniternalThreadLocalMap的引用。

如果是Thread对象，则调用ThreadLocal的remove方法。





删除方法里面有一个很重要的部分就是RemoveSet，也就是底层数组的0号位置。

FastThreadLocal中每次Set元素都会将对应的FastThreadLocal对象添加到这个RemoveSet。

这样以来虽然废了数组的一个位置，但是再删除的时候不需要在遍历整个数组了，而且如果直接遍历数组也就没办法在调用到不同FastThreadLocal#onRemoval方法了。





## 关于内存泄露问题

综合上述所讲，每个FastThreadLocal会携带一个index，然后根据index确定自己在数组中的下标记。

那么它也会出现内存泄露的问题，就是在FastThreadLocal这个Key已经被回收之后，数组中的value还未回收，但是按照正常情况又已经获取不到这个value了。

FastThreadLocal好像并没有提供太好的可以帮助开发者解决内存泄露问题的方法，所以这个问题只能靠良好的编程习惯解决。

**就是在使用完之后remove一下，其他我也不知道咋办了**



## 总结

在我的理解中，FastThreadLocal对原生的ThreadLocal的优化可能就在于以下几点:

1. **最大优化可能就在于取值方法。**

原始的ThreadLocal需要先进行一次Hash计算，时间复杂度最优情况下为O(1)，但是如果出现Hash冲突，那么最坏也就需要遍历整个数组。

但在FastThreadLocal中，直接使用index取数组中的值，时间复杂度永远都是O(1)。

减少了计算的过程，也就加快了寻址的速度，提升了性能。



2. 对象的清理

ThreadLocal类在原生的Thread类被回收之后，ThreadLocalMap结构需要等待GC清除，开发者基本没有参与空间，但是FastThreadLocal可以。

以下FastThreadLocalThread的其中几种构造函数:

![image-20201122211105436](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122211105436.png)

在接收到Runnable的参数时，都会先将其包装为FastThreadLocalRunnable，然后继续别的操作。

再来看看FastThreadLocalRunnable。

![image-20201122211237185](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20201122211237185.png)

这里就很明显了，FastThreadLocalRunnable相当于对Runnable的又一层包装，包装了run方法执行完后的清理过程。





## FastThreadLocal的缺点

1. index只增不减

每一个FastThreadLocal的index都是由InternalThreadLocalMap#nextVariableIndex方法下发的，从下发的方法可以之后就是对AtomicInteger的+1并获取，这就导致了index的大小问题。

InternalThreadLocalMap的数组原始大小为32，如果系统中存在很多的FastThreadLocal，即使他们的生命周期很短，但是造成的index+1影响一直都在，会导致数组频繁被扩容。

这个缺点导致的就是FastThreadLocal的使用量的问题，过多的FastThreadLocal会导致index的大小异常。

2. 对空间的浪费问题

由index确定的元素的存储位置，但是index确实全局下发的。

这就有一个问题，我定义了一个FastThreadLocal，但是并没有很多线程上有用，那没有使用的线程也必须将这个index的位置空着，这也就造成了浪费。

常见的比如项目中往往会定义多个线程池作为不同的作用，一些FastThreadLocal可能只在某个线程池中使用，但是别的线程池却要为这个FastThreadLocal留个空缺。

3. 内存泄露问题

如上所说的，FastThreadLocal并没有解决内存泄露的问题，诚然它比ThreadLocal更进一步，在Runnable方法执行完毕之后会清理Map中的数据，但在执行期间对并没有什么作为。

对于线程池来说，有些Thread的生命周期基本可以和系统等同，如果一个FastThreadLocal在中途被回收掉了，那对应的数据却还是保留在数组中的。

