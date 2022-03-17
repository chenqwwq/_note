# ThreadLocal

---

[TOC]

---



## 思维导图

![ThreadLocal思维导图](../assets/ThreadLocal.png)

<br>



## 概述

ThreadLocal（线程局部变量），作用是**保存每个线程的私有变量**，以空间换时间的方式，为每一个线程保存一份**私有**变量，也就不存在所谓的并发问题。

> 真实的数据并不会存在 ThreadLocal 中。
>
> 实际上，数据都保存在 Thread 对象中 Thread#threadLocals 这个成员变量里，所以一定程度上 ThreadLocal 只是一个操作该集合的工具类。

<br>

以下就是 ThreadLocalMap 在 Thread 中的变量声明:

 ![ThreadLocalMap的变量声明](../assets/ThreadLocalMap的变量声明.png)

>threadLocals 是给 ThreadLocal 用的，该类只能访问当前线程中的数据。
>
>inheritableThreadLocal 是给 InheritableThreadLocal 用的，使用该类子线程可以访问到父线程的数据。

<br>





## ThreadLocal

ThreadLocal 的内部方法因为逻辑都不复杂，不需要单独出来看,就直接全放一块了。

### 数据获取

```java
   // 直接获取线程私有的数据
   public T get() {
        // 获取当前线程
        Thread t = Thread.currentThread();
        // getMap其实很简单就是获取`t`中的`threadLocals`,代码在`工具方法`中
        ThreadLocalMap map = getMap(t); 
        if (map != null) {										// 3.
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {							// 2.
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();  				// 1.
    }
	// 这个方法只有在上面`1.`处调用...不知道为什么map,thread不直接传参
	// 该方法的功能就是为`Thread`设置`threadLocals`的初始值
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        // map不为null表明是从上面的`2.`处进入该方法
        // 已经初始化`threadLocals`,但并未找到当前对应的`Entry`
        // 所以此时直接添加`Entry`就行
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
      // 初始值,`protected`方便子类继承,并定义自己的初始值.
      protected T initialValue() {
        return null;
      }

	// 创建并赋值`threadLocals`的方法
     void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

整个获取的过程：

1. 通过 Thread#currentThread 方法获取当前线程对象。
2. 首先通过 getMap 方法获取当前线程绑定的 threadLocals。
3. 不要为空时，以当前 ThreadLocal 对象为参数获取对应的Entry 对象，为空跳到第四步
4. 获取 Entry 对象中的 value 并返回。
5. 调用 setInitialValue方法初始化值并返回。

<br>

**这里可以很明显的看出来，数据其实还是保存在 Thread 对象里的，并且通过 getMap 方法确定目标 ThreadLocalMap。**

通过 setInitialValue 方法可以设定初始值。

> 例如，希望统计每个线程的某个操作计数，那么就可以用如下的方法：
>
> ```java
> ThreadLocal<Integer> counter = new ThreadLocal<Integer>() {
>         @Override
>         protected Integer initialValue() {
>             return 0;
>         }
> };
> ```
>
> 以 0 为初始值做统计。

<br>

<br>

### 数据存储 

```java
 public void set(T value) {
        // 获取当前线程
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);     // .1
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
 }
```

流程简述如下：

1. 获取当前线程,并以此获取线程绑定的 ThreadLocalMap 对象。
2. map 不为空时,直接set。
3. map 为空时需要先创建 Map 并赋值。

<br><br>

> **Thread 中存在两个 ThradLocalMap，在存取方法中都是通过 getMap 方法确定需要操作的集合。**

## ThreadLocalMap

ThreadLocalMap 类似于 HashMap ，也是使用 Hash 算法定位存取的数据结构，**以 ThreadLocal 对象为 Key，局部变量为 Value**

Hash 算法合理时 ThreadLocalMap 的存取操作近乎是 O(1) 的复杂度。

ThreadLocalMap 出人意料的并没有继承任何一个类或接口，是完全独立的类（以为会像 HashMap 一样继承 AbstractMap。

<br>

### 成员变量

  ```java
// 默认的初始容量 一定要是二的次幂
private static final int INITIAL_CAPACITY = 16;
// 元素数组/条目数组
private Entry[] table;
// 大小,用于记录数组中实际存在的Entry数目
private int size = 0;
// 阈值
private int threshold; // Default to 0 构造方法
  ```

> **ThreadLocalMap 的底层数据结构是 Entry 的数组，**并且默认容量为16。

<br>

以下为 Entry 对象的声明形式：

 ![ThreadLocalMap$Entry](../../pic/image-20210221154222208.png)

> **WeakReference 声明了 Entry 对象中对于对于 ThreadLocal 对象的引用是弱引用。**
>
> **弱引用消除了 ThreadLocalMap 的引用对 ThreadLocal 的 GC 的影响，**这是 ThreadLocal 避免内存泄漏的核心逻辑。

<br>

### 元素获取

#### getEntry(ThreadLocal<?> key) 

该方法就是通过 ThreadLocal 对象获取对应的数据。

```java
private Entry getEntry(ThreadLocal<?> key) {
        // 和HashMap中一样的下标计算方式
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        // 获取到对应的Entry之后就分两步
        if (e != null && e.get() == key)
            // 1. e不为空且threadLocal相等
            return e;		
        else														
            // 2. e为空或者threadLocal不相等				
            return getEntryAfterMiss(key, i, e);
}
```

起手就是一个 HashCode & (len - 1)，和 HashMap 类似，但ThreadLocal 的 HashCode 和 HashMap 中的直接调用 hashCode() 方法不同。

ThreadLocal 是采用递增的形式，而非直接计算对象的 HashCode。

<br>

 ```java
 private final int threadLocalHashCode = nextHashCode();
 private static AtomicInteger nextHashCode = new AtomicInteger();  
 private static int nextHashCode() {
 	return nextHashCode.getAndAdd(HASH_INCREMENT);
 }
 ```
 以上就是 HashCode 的获取方式，**是以类变量的方式递增获取**，相对于直接调用 hashCode() 可以更好的减少 hash 冲突。

> 每次创建一个 ThreadLocal，hashCode 都会+1，所以能使数据更加均匀的散布在数组中，更好的减少 hash 冲突。

<br>

如果hash计算出来的下标存在想要的元素就直接返回，如果获取元素为空还会再调用 `getEntryAfterMiss` 做冲突查询的后续处理.

<br>		<br>

#### getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e)

该方法是在直接按照 `Hash` 计算下标后，没获取到对应的 `Entry` 对象的时候调用，**下标处不是想要的元素就说明出现了 Hash 冲突。**

以下为方法源码：

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
        Entry[] tab = table;
        int len = tab.length;
        // 此时注意如果从上面情况`2.`进来时,
        // e为空则直接返回null,不会进入while循环
        // 只有e不为空且e.get() != key时才会进while循环
        while (e != null) {
            ThreadLocal<?> k = e.get();
            // 找到相同的k,返回得到的Entry,get操作结束
            if (k == key)
                return e;
            // 若此时的k为空,那么e则被标记为`Stale`需要被`expunge`
            if (k == null)
                expungeStaleEntry(i);
            else	// 下面两个都是遍历的相关操作
                // nextIndex就是+1判断是否越界
                i = nextIndex(i, len);
            e = tab[i];
        }
        return null;
}
```

> **在判断出现 hash 冲突之后，直接往后线性查找之后的数组元素。**

<br>

<br>

#### expungeStaleEntry(int staleSlot)

该方法用来清除 `staleSlot` 位置的 Entry 对象,并且会**清理当前节点到下一个 `null` 节点中间的过期 `Entry`.**

```java
   /** 
     * 清空旧的Entry对象
     * @param staleSlot: 清理的起始位置
     * @param return: 返回的是第一个为空的Entry下标
     */
    private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
        	// 清空`staleSlot`位置的Entry
        	// value引用置为空之后,对象被标记为不可达,下次GC就会被回收.
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;
            Entry e;
            int i;
        	// 通过nextIndex从`staleSlot`的下一个开始向后遍历Entry数组,直到e不为空
         	// e赋值为当前的Entry对象
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;		
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                // 当k为空的时候清空节点信息
                if (k == null) {							
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {	// 以下为k存在的情况
                    int h = k.threadLocalHashCode & (len - 1);
                    // 元素下标和key计算的不一样，表明是出现`Hash碰撞`之后调整的位置
                    // 将当前的元素移动到下一个null位置
                    if (h != i) {					
                        tab[i] = null;
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        } 
```

该方法是对内存泄露的进一步处理。

**如果将 ThreadLocal 的内存泄露问题分成两个部分来看，一个是 Key，另外一个就是 Value。**

**Key 的部分依靠弱引用清除，如果外部的强引用断开之后，也就是没有地方在使用到该 Key 之后，Key 会被 GC 回收，所以引用就为 null。**

从而判断 Key 为 null 的 Value 就是 Stale 的对象，则靠该方法清除。

> **ThreadLocal 靠弱引用清除的只有 Key 对象，还有 Value 对象则需要靠扫描，所以内存泄露的情况并不是能够完全避免的。**

<br>

<br>

### 元素添加

#### set(ThreadLocal<?> key, Object value)

该方法就是添加元素的方法。

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    // 整个循环的功能就是找到相同的key覆盖value
    // 或者找到key为null的节点覆盖节点信息
    // 只有在e==null的时候跳出循环执行下面的代码
    for (Entry e = tab[i];
         e != null;	
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        // 找到相等的k,则直接替换value,set操作结束
        if (k == key) {
            e.value = value;
            return;
        }
        // k为空表示该节点过期,直接替换该节点
        if (k == null) {					       // 1.
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // 走到这一步就是找到了e为空的位置，不然在上面两个判断里都return了
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

通过 hashCode 确定下标后，如果 Key 相等则直接覆盖原数据，如果 Key 不相等则往后线性查找元素，找到为 null 的元素直接覆盖，或者找到空余的位置赋值。

<br>

最后会清理旧的元素，并且判断 threshold，决定是否需要扩容。

> **ThreadLocalMap 处理 Hash 冲突的方法叫做 线性寻址法，在冲突之后往后搜索，找到第一个为空的下标并保存元素。**
>
> 线性寻址法在出现 Hash 冲突的时候处理的复杂度基本会变成 O(n)，并不能直接找一个 null 点就存储，因为数组中可能还有相同的 Key 在后面。

<br>

<br>

#### replaceStaleEntry

源码中只有从上面 `1.` 处进入该方法,用于**替换  `key`  为空的 `Entry` 节点,顺带清除数组中的过期节点.**

往后搜索的是第一个为空或者 Key 相等，如果先找到 Key 为空的并不能保证后续的节点没有 Key 相等的，所以在 replaceStaleEntry 方法中可能还需要处理另外一个 Key 相同的节点。

```java
/**
 *	从`set.1.`处进入,key是插入元素ThreadLocal的hash,staleSlot为key为空的数组节点下标
 */
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;
        Entry e;
        int slotToExpunge = staleSlot;
        // 从传入位置,即插入时发现k为null的位置开始,向前遍历,直到数组元素为空
        // 找到最前面一个key为null的值.	
        // 这里要吐槽一下源代码...大括号都不加 习惯真差
        for (int i = prevIndex(staleSlot, len);
             (e = tab[i]) != null;		
             i = prevIndex(i, len)){
            // 向前获取到第一个 Key 为空的对象
            if (e.get() == null)
                // 因为是环状遍历所以此时slotToExpunge是可能等于staleSlot的
                slotToExpunge = i;
        }
        // 该段循环的功能就是向后遍历找到`key`相等的节点并替换
        // 并对之后的元素进行清理
        for (int i = nextIndex(staleSlot, len);
             (e = tab[i]) != null;
             i = nextIndex(i, len)) {
            ThreadLocal<?> k = e.get();
            if (k == key) {	
                // 替换 e 的 value
                e.value = value;
                // staleSlot 是因为 key 为 null 才进来的
                // 所以 tab[i] 也是需要清理的节点
                tab[i] = tab[staleSlot];
                tab[staleSlot] = e;
                if (slotToExpunge == staleSlot)
                    slotToExpunge = i;
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                return;
            }
            // 其实我对这个`slotToExpunge == staleSlot`的判断一直挺疑惑的,为什么需要这个判断?
            if (k == null && slotToExpunge == staleSlot)
                slotToExpunge = i;
        }
        // e==null时跳到下面代码运行
        // 清空并重新赋值
        // 断开 Entry 对应的数据的强引用
        tab[staleSlot].value = null;
        tab[staleSlot] = new Entry(key, value);
        // set后的清理
        if (slotToExpunge != staleSlot)
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}

```

**如上所说，再出现 Hash 冲突的时候，往后搜索的是第一个为空的节点，并不能直接赋值，因为在后续的数组中可能还存在相同的 Key 的节点。**

替换元素之前会先向前搜索找到一个 Key 为 null 的节点。

<br>

<br>



#### cleanSomeSlots

该方法的功能是就是清除数组中的过期 Entry

```java
/**
  * @param i 清除的起始节点位置
  * @param n 遍历控制,每次扫描都是log2(n)次,一般取当前数组的`size`或`len`
  */
private boolean cleanSomeSlots(int i, int n) {
    		// 是否有清除的标记
            boolean removed = false;
    		// 获取底层数组的数据信息
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) {
                    // 当发现有过期`Entry`时,n变为len
                    // 即扩大范围,全数组范围在遍历一次
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }	
                // 无符号右移一位相当于n = n /2
                // 所以在第一次会遍历`log2(n)`次
            } while ( (n >>>= 1) != 0);
    		// 遍历过程中没出现过期`Entry`的情况下会返回是否有清理的标记.
            return removed;
        }
```

首次清除从 i 向后开始遍历 log2(n) 次,如果之间发现过期 Entry 会直接将 n 扩充到 len 可以说全数组范围的遍历，发现过期 Entry 就调用expungeStaleEntry 清除直到未发现 Entry 为止。

<br>

<br>

### 扩容调整方法

#### rehash

容量调整的先驱方法,先清理过期 Entry ，并做是否需要 resize 的判断。

调整的条件是**当前size大于阈值的3/4**就进行扩容

```java
 private void rehash() {
     // 清理过期Entry
     expungeStaleEntries();
     // 初始阈值threshold为10
     if (size >= threshold - threshold / 4)
         resize();
 }
```

<br>

<br>

#### resize

扩容的实际方法.

```java
private void resize() {
        // 获取旧数组并记录就数组大小
        Entry[] oldTab = table;
        int oldLen = oldTab.length;
        // 新数组大小为旧数组的两倍
        int newLen = oldLen * 2;
        Entry[] newTab = new Entry[newLen];
        int count = 0;
        // 遍历整个旧数组,并迁移元素到新数组
        for (int j = 0; j < oldLen; ++j) {
            Entry e = oldTab[j];
            // 判断是否为空,空的话就算了
            if (e != null) {
                ThreadLocal<?> k = e.get();
                // k为空即表示为过期节点,当即清理了.
                if (k == null) {
                    e.value = null; 
                } else {
                    // 重新计算数组下标,如果数组对应位置已存在元素
                    // 则环状遍历整个数组找个空位置赋值
                    int h = k.threadLocalHashCode & (newLen - 1);
                    while (newTab[h] != null)
                        h = nextIndex(h, newLen);
                    newTab[h] = e;
                    count++;
                }
            }
        }
        // 设置新属性	
        setThreshold(newLen);
        size = count;
        table = newTab;
}
```

<br>

<br>

## InheritableThreadLocal 

ThreadLocal 是对单线程而言的，如果出现线程切换会丢失 ThreadLocal 的数据，而 InheritableThreadLocal 可以实现子线程对父线程 ThreadLocalMap 的继承。

**继承的 ThreadLocalMap 就是 Thread 对象中的 inheritableThreadLocals。**

<br>

InheritableThreadLocal 的主要实现如下：

![InheritableThreadLocal#getMap和createMap](../assets/image-20210902161137264.png)

基本逻辑和 ThreadLocal 并无不同，只是切换了操作的 ThreadLocalMap 对象。

**因此只要在创建自线程的时候完全继承父线程的 inheritableThreadLocals 对象就可以实现变量的继承。**

继承的实现逻辑在 Thread 的构造函数中，以下是 Thread#init 方法的片段：

![image-20210902161430450](../assets/image-20210902161430450.png)

方法会获取 parent 线程的 inheritableThreadLocals，并复制到当前的 Thread 对象。

<br>

## Q & A

> Q: ThreadLocal 为何会出现内存泄露？

**ThreadLocal 会出现内存泄露的主要原因是如果是强引用，那么在 ThreadLocal 类不再使用之后，ThreadLocalMap 中无法清除相关的 Entry 对象。**

在 ThreadLocal 不再使用之后，ThreadLocalMap 中指向 ThreadLocal 的强引用也会导致 ThreadLocal 无法被 GC 回收，同理 Value 对象也被保留了下来。

**也就出现了所谓的内存泄露，无用的数据无法被 GC 有效的清除。**

<br>

<br>





>  Q: ThreadLocal 如何解决内存泄漏?

ThreadLocal 的内存泄露可以分为 Key（也就是 ThreadLocal），以及 Value。

**解决 Key 的内存泄露的方法就是采用弱引用，弱引用消除了 ThreadLocalMap 对 ThreadLocal 对象的 GC 的影响。**

另外的在每次获取或者添加数据的时候都会判断 Key 是否被回收，如果 Key 已经被回收会连带清理 Value 对象，这也就顺带解决了 Value 的泄露问题。

<br>

<br>



>  Q: ThreadLocalMap 如何解决Hash冲突？

Hash 冲突就是指通过 Hash 计算的下标值一致，两个元素的定位一致。

HashMap 解决 Hash 冲突的方法就是**拉链法**，底层的数组中保存的不是单一的数据，而是一个集合(链表/红黑树)，冲突之后下挂。

采用拉链法的结果就是在Hash冲突严重时会严重影响时间复杂度，因为就算是红黑树查询的事件复杂度都是 O(Log2n)。

ThreadLocalMap 并没有采用这种方法，而是使用的**开放寻址法**，如果已经有数据存在冲突点，就在数组中往下遍历找到第一个空着的位置。

> 需要注意的是，并不是找到空的位置就可以直接替换，还是需要遍历整个数组确保没有重复的 Key。

<br>

<br>



>  Q: ThreadLocalMap 和 HashMap 的异同

两个都是采用 Hash 定位的数据结构，底层都是以数组的形式。

但是 HashCode 的获取方式不同，HashMap 调用对象的 hashCode() 方法，而  ThreadLocalMap 中的 Key 就是 ThreadLocal，ThreadLocal 的 HashCode 是递增分配的。

另外处理 Hash 冲突的方式不同，ThreadLocalMap 采用的开放寻址法，而 HashMap 采用的是拉链法。

<br>



> Q: ThreadLocal 在线程池中的使用

ThreadLocal 在线程池中的使用，因为 Thread 对象的复用，所以会导致 ThreadLocalMap 变量的使用混乱。

因此需要在任务完成之后清理公用的 ThreadLocal 对象，可以对 execute 的 Runnable 变量进行二次封装，参照 Netty 的 FastThreadLocalRunnable 的实现:

```java
  class ThreadLocalSafeRunnable implements Runnable {
        Runnable runnable;
        List<ThreadLocal<?>> threadLocals;

        public ThreadLocalSafeRunnable(Runnable runnable, ThreadLocal<?>... ths) {
            this.runnable = runnable;
            if (ths.length > 0) {
                this.threadLocals = new ArrayList<>();
                Collections.addAll(threadLocals, ths);
            }
        }

        @Override
        public void run() {
            if (runnable == null) {
                	return;
            }
            try {
                	run();
            } finally {
                    if (threadLocals != null) {
                        for (ThreadLocal<?> threadLocal : threadLocals) {
                            threadLocal.remove();
                        }
                    }
            }
        }
    }
```



> Q: ThreadLocal 在线程间的传递

基本使用 InheritableThreadLocal 就可以。

如果希望特殊实现，也可以直接继承 Thread，定义自己的 ThreadLocalMap。



