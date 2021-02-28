# ThreadLocal



> 在`趣链`的面试中被问到`ThreadLocal`的相关问题，被问的一脸懵*,所以有次总结.
>
> 2020/2/21 - 整理相关问题

---

[TOC]



## 概述

ThreadLocal（线程局部变量），作用是**为每个线程对象中保存该线程的私有变量**。

> 真实的数据并不会存在ThreadLocal中，而是在Thread对象中Thread#threadLocals这个成员变量中，所以一定程度上ThreadLocal只是一个操作该集合的工具类。

以下就是ThreadLocalMap在Thread中的声明:

 ![image-20210221153908595](https://chenqwwq-img.oss-cn-beijing.aliyuncs.com/img/image-20210221153908595.png)

>threadLocals是给ThreadLocal用的，该类只能访问当前线程中的数据。
>
>inheritableThreadLocal是给InheritableThreadLocal用的，子线程可以访问到父线程的数据。



## ThreadLocalMap

ThreadLocalMap类似于HashMap是使用Hash算法定位存取的数据结构，ThreadLocal作为Map中的Key。

> 只要有Key就能获取数据，因为数据都保存在Thread.currentThread()#threadLocals中。

`ThreadLocalMap`出人意料的并没有继承任何一个类或接口，是完全独立的类，作为一个Map内部存放的是K/V形式的数据，以ThreadLocal对象为Key。



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

> ThreadLocalMap的底层数据结构是Entry的数组。

以下为Entry对象的声明形式：

 ![image-20210221154222208](/home/chen/github/_note/pic/image-20210221154222208.png)

> WeakReference就是Java中的弱引用，以ThreadLocal作为弱引用对象。
>
> 对象在被GC扫描到之后，发现该对象仅仅只有弱引用指向它，那么它就会被回收。



### 元素获取

#### getEntry(ThreadLocal<?> key) 

通过ThreadLocal对象获取对应的数据。

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

因为`ThreadLocalMap`底层也是使用数组作为数据结构，所以该方法也**借鉴了`HashMap`中求元素下标的方式**.

> ThreadLocal的HashCode和HashMap中的直接调用hashCode()方法不同，ThreadLocal是采用递增的形式。
>
> ```java
> private final int threadLocalHashCode = nextHashCode();
> private static AtomicInteger nextHashCode = new AtomicInteger();  
> private static int nextHashCode() {
>  return nextHashCode.getAndAdd(HASH_INCREMENT);
> }
> ```
> 以上就是Key的获取方式，Key是以类变量的方式递增获取，相对于直接调用hashCode()可以更好的减少hash冲突，也有hash冲突的解决方式的不同。



如果hash计算出来的下标存在想要的元素就直接返回，如果获取元素为空还会再调用`getEntryAfterMiss`做后续处理.

#### getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e)

- 该方法是在直接按照`Hash`计算下标后，没获取到对应的`Entry`对象的时候调用。

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

> 在发生Hash冲突导致Key并不在计算出来的下标之后，直接采用的遍历数组的形式查找所有的Key，从预算的下标开始找到第一个空缺的位置。



#### expungeStaleEntry(int staleSlot)

- 该方法用来清除`staleSlot`位置的Entry对象,并且会**清理当前节点到下一个`null`节点中间的过期`Entry`.**

> 如果将ThreadLocal的内存泄露问题分成两个部分来看，一个是Key，另外一个就是Value。
>
> Key的部分依靠弱引用清除，而Value的部分则靠该方法反复清除。

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



### 元素添加

#### set(ThreadLocal<?> key, Object value)

- 该方法就是添加元素的方法。
- 从下方源码也可以看出来,`Entry`再确定数组位置之后直接就开始了遍历,如果`key`不匹配就往后遍历找到`key`匹配的元素覆盖,或者`key == null`的替换.

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

#### replaceStaleEntry

- 源码中只有从上面`1.`处进入该方法,用于**替换`key`为空的`Entry`节点,顺带清除数组中的过期节点.**

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
                    // e就是tab[i]，所以下三行代码的功能就是替换Entry
                    // 新的Entry实际还是在staleSlot下标的位置
                    e.value = value;
                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;
                    // 因为接下来要进行清理操作,所以此处需要重新确定清理的起点.
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
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);
			// set后的清理
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }

```

#### cleanSomeSlots

- 该方法的功能是就是清除数组中的过期`Entry`
- 首次清除从`i`向后开始遍历`log2(n)`次,如果之间发现过期`Entry`会直接将`n`扩充到`len`可以说全数组范围的遍历.发现过期`Entry`就调用`expungeStaleEntry`清除直到未发现`Entry`为止.

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



### 扩容调整方法、

#### rehash

- 容量调整的先驱方法,先清理过期`Entry`,并做是否需要`resize`的判断
- 调整的条件是**当前size大于阈值的3/4**就进行扩容

```java
 private void rehash() {
     		// 清理过期Entry
            expungeStaleEntries();
     		// 初始阈值threshold为10
            if (size >= threshold - threshold / 4)
                resize();
        }
```

#### resize

- 扩容的实际方法.

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





---

- `ThreadLocal`的内部方法因为逻辑都不复杂,不需要单独出来看,就直接全放一块了.

#### Get方法

- 整个获取的过程其实并不难,所以我说`ThreadLocal`的精华只要还是在`TheradLocalMap`和这种**空间换时间**的结构.
  1. 首先通过`getMap`方法获取当前线程绑定的`threadLocals`
  2. 不要为空时,以当前`ThreadLocal`对象为参数获取对应的`Entry`对象.为空跳到第四步
  3. 获取`Entry`对象中的`value`,并返回
  4. 调用`setInitialValue`方法,

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



#### Set方法

1. 获取当前线程,并以此获取线程绑定的`ThreadLocalMap`对象.
2. `map`不为空时,直接set就好
3.  `map`为空时需要先创建并赋值.

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

## 相关问题

### 为什么Entry要使用弱引用

弱引用的特点就是如果一个对象**只存在弱引用**，那么在被GC扫描到之后就会被回收。

在ThreadLocalMap中效果就是这样，因为Thread对象和ThreadLocalMap中K/V数据的生命周期可能不同，往往是Thread对象还在但是Map中数据已经无用。

但这时候Map中却还保存在ThreadLocal对象的强引用，因此就使得GC无法回收无用数据，造成内存泄露。

> 使用弱引用的原因在我总结来看就是避免ThreadLocalMap中强引用干扰到ThreadLocal对象的回收。

在ThreadLocal对象被回收之后，通过判断Key是否还存在也可以进一步清除无用的Value。



### ThreadLocal如何解决内存泄漏

首先内存泄露的原因就是上述的原因:

> Key/Value数据的生命周期和Thread对象的生命周期之间的差异，在Thread对象未被回收前，如果没有弱引用ThreadLocalMap就一直持有着Key，Value的引用，Key/Value的生命周期被迫和Thread对象一致。

解决的方法就是采用弱引用。

另外的在每次获取或者添加数据的时候都会判断Key是否被回收，如果Key被回收就表明该数据的生命周期已经过了，所以会连带清理Value对象。





### ThreadLocalMap如何解决Hash冲突

> Hash冲突是使用Hash类算法都会遇到的问题，不同的Key计算得到一样的数值。

HashMap解决Hash冲突的方法就是**拉链法**，底层的数组中保存的不是单一的数据，而是一个集合(链表/红黑树)，冲突之后下挂。

采用拉链法的结果就是在Hash冲突严重时会严重影响时间复杂度，因为就算是红黑树查询的事件复杂度都是O(Log2n)。

ThreadLocalMap并没有采用这种方法，而是使用的**开放寻址法**，如果已经有数据存在冲突点，就在数组中往下遍历找到第一个空着的位置。



### ThreadLocalMap和HashMap的异同

两个都是采用Hash定位的数据结构，底层都是以数组的形式。

但是HashCode的获取方式不同，HashMap调用对象的hashCode()方法，而ThreadLocalMap中的Key就是ThreadLocal，ThreadLocal的HashCode是递增分配的。

另外处理Hash冲突的方式不同，ThreadLocalMap采用的开放寻址法，而HashMap采用的是拉链法。

