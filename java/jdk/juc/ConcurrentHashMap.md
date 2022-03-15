# ConcurrentHashMap

>基于 JDK1.8 版本。 - chenqwwq 2020/03

---

[TOC]

---

## 概述

ConcurrentHashMap 就是线程安全的 HashMap。

在 1.8 以前的实现中，ConcurrentHashMap 以**分段锁**来实现并发安全，**而在 1.8 之后并发安全的实现依靠 CAS 以及 synchronize 关键字，并且上锁的粒度变成了单个桶，在写操作中只会对桶的头节点上锁**

> synchronized 关键字的性能在 1.8 之后得到了很大的提升（加入了偏向锁，轻量级锁等。





## 成员变量（关键的

![ConcurrentHashMap#成员变量表](../assets/image-20210911184252767.png)

以上是 ConcurrentHashMap 中大部分关键的成员列表，含义如下：

| 变量名        | 变量类型         | 变量含义                                                     |
| ------------- | ---------------- | ------------------------------------------------------------ |
| table         | Node 数组        | Hash 桶数组，保存真实的数据                                  |
| nextTable     | Node 数组        | 扩容时的过渡数组，扩容成功之后和 table 互换引用（类似 Redis 中 Dict 的实现 |
| baseCount     | Long             | 基础的元素个数（在没有任何 CAS 失败的情况下                  |
| counterCells  | CounterCell 数组 | **结合 baseCount 得到真实的元素个数，在 CAS 失败后，每个组的变动元素个数会保存在各自的 Cell 里** |
| transferIndex | int              | 扩容索引值,表示已经分配给扩容线程的table数组索引位置，主要用来协调多个线程间迁移任务的并发安全。 |
| sizeCtl       | int              | 表示  ConcurrentHashMap 中的状态变化。                       |









> sizeCtl 变量的作用？

sizeCtl 表示的是 ConcurrentHashMap 的状态，可以分为如下情况：

- sizeCtl > 0 时，表示 ConcurrentHashMap 的容量：
  1. 表示初始容量
  2. 表示初始之后的扩容阈值
- sizeCtl = -1 表示当前正处于初始化或者扩容阶段
- sizeCtl < -1 时，高16位表示扩容的标识符号，低16位表示参与的线程



## 内部类

（ ConcurrentHashMap 的内部类太多了，只看关键的吧。


### Node 

Node 表示一个 K/V 的数据对，并且也是桶中链表节点，具体源码如下：

![image-20210911192655248](../assets/image-20210911192655248.png)

常规的链表节点实现，但是它继承了 Map.Entry。

<br>

### ForwardingNode（转发节点

<br>

该类仅仅存活在扩容阶段,作为一个标记节点放在桶的首位,并且指向是 `nextTable`<font size=2>(扩容的中间数组)</font>

从构造函数可知,`ForwardingNode`  的`hash` 为 MOVED（常量 -1）,其他为空，是个完完全全的辅助类.

```java
static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable;
    
    	// 构造函数中默认以MOVED:-1为Hash,其它为空
        ForwardingNode(Node<K,V>[] tab) {
            super(MOVED, null, null, null);
            this.nextTable = tab;
        }
    	// 帮助扩容时的元素查找
        Node<K,V> find(int h, Object k) {
            // loop to avoid arbitrarily deep recursion on forwarding nodes
            outer: for (Node<K,V>[] tab = nextTable;;) {
                Node<K,V> e; int n;
                if (k == null || tab == null || (n = tab.length) == 0 ||
                    (e = tabAt(tab, (n - 1) & h)) == null)
                    return null;
                for (;;) {
                    int eh; K ek;
                    if ((eh = e.hash) == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                    if (eh < 0) {
                        if (e instanceof ForwardingNode) {
                            tab = ((ForwardingNode<K,V>)e).nextTable;
                            continue outer;
                        }
                        else
                            return e.find(h, k);
                    }
                    if ((e = e.next) == null)
                        return null;
                }
            }
        }
    }
```

- `find()` 方法实在扩容期间帮助 `get` 方法获取桶中元素.



## 元素新增方法

CHM#put 直接调用 putVal 方法。

```java
 	public V put(K key, V value) {
        return putVal(key, value, false);
    }
```

<br>

### putVal  - 新增元素

```java
  /**
	 * 方法参数:
	 * 1. key,value 自然不用说就是k/v的两个值
	 * 2. onlyIfAbsent 若为true,则仅仅在值为空时覆盖
	 * 返回值:
	 *  返回旧值,若是新增就为null.
	 */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        // CHM不支持NULL值的铁证
        if (key == null || value == null) throw new NullPointerException();
        // 获得key的Hash,spread可以称之为扰动函数
        int hash = spread(key.hashCode());
        int binCount = 0;
        // 无限循环
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            // 1. 在tab为空时负责初始化Table
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            // 2. 使用`(n-1)&hash`确定了元素的下标位置,获取对应节点
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 如果对应位置节点为空,直接以当前信息为桶的头节点
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // 3. 如果获取的桶的头结点的`Hash`为`MOVED`,表示该节点是`ForwardingNode`
            // 也就表示数组正在进行扩容
            else if ((fh = f.hash) == MOVED)
                // 帮助扩容
                tab = helpTransfer(tab, f);
            else {
              // 4. 桶存在并且当前处于正常状态
                V oldVal = null;
                // 上锁保证原子性,volatile仅能保证可见性
                // f为key获取到的节点元素,以此为锁对象
                synchronized (f) {
                    // f在上文就是根据`tabAt(tab,i)`获取的
                    // 此处是再次获取验证有没有被修改
                    if (tabAt(tab, i) == f) {
                        // 与else.if比较,得知
                        // fh >= 0表示当前节点为链表节点,即当前桶结构为链表 		  ？？？
                        if (fh >= 0) {
                            // 链表中的元素个数统计
                            binCount = 1;
                            // 循环遍历整个桶
                            // 跳出循环的两种情况:
                            // 1. 找到相同的值,binCount此时表示遍历的节点个数
                            // 2. 遍历到末尾,binCount就表示桶中的节点个数
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 源码中大量运用了表达式的短路特性,来展示判断的优先级
                                // 1. 若hash不相等,则直接跳过判断
                                // 2. hash相等之后,若key的地址相同,则直接进入if
                                // 3. 地址不同时在进入判断内容是否相等
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    // onlyIfAbsent为true,表示存在时不覆盖内容
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    // 已经找到确定的元素了,更新不更新都跳出
                                    break;
                                }
                                // 因为e就在同步代码块中,桶已经被上锁,不可能有别的线程改变
                                // 所以不需要重新获取
                                Node<K,V> pred = e;
                                // 1. 如果e为空,则直接将元素挂接到e后面,跳出循环
                                // 2. e不为空,继续遍历
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        // 类似HashMap,树节点独立操作.
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                // 表示进入了上面的同步表达式,对桶进行修改之后
                if (binCount != 0) {
                    // 如果binCount大于树的临界值,就将链表转化为红黑树
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    // 如果oldVal部位空,则返回
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // 添加元素计数,并在binCount大于0时检查是否需要扩容
        addCount(1L, binCount);
        return null;
    }
```



CHM 中使用桶头节点的 Hash 表示当前桶的状态。

```java
/*
 * Encodings for Node hash fields. See above for explanation.
 */
static final int MOVED     = -1; // hash for forwarding nodes
static final int TREEBIN   = -2; // hash for roots of trees
static final int RESERVED  = -3; // hash for transient reservations
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
```







#### 元素新增流程小结：

1. 判断并排除 key,value 为空的情况（CHM 不支持 key/value 为空。
2. 得到扰动后的 hash，获取对应下标的桶，若桶为空则进行初始化。
3. 通过 `(n - 1) & hash` 的公式获取桶的下标 ，若桶为空则直接 CAS 填充 key/value 为桶的头节点。
4. 判断桶的头节点 hash,若 hash == MOVED 表示**数组在扩容并帮助扩容。**
5. 进入`synchronize`的同步代码块,如果**桶的头节点的hash大于0表示桶的结构为链表**,接下去就是正常的链表遍历,新增或者覆盖.
6. 如果**桶的头节点是`TreeBin`类型表示桶的结构为红黑树**,按红黑树的操作进行遍历.
7. 退出同步代码块,判断在遍历期间统计的`binCount`是否需要转化为红黑树结构.
8. 判断`oldVal`是否为空,这步也挺关键的,如果不为空表示时覆盖操作,直接`return`就好.
9. 如果`oldVal`不为空调用`addCount`方法新增元素个数,并检测是否需要扩容.





## 元素获取方法

```java
   public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
       	// 获取hash，并进过扰动
        int h = spread(key.hashCode());
       	// 判断以进入获取方法
       	// 1. 数组不为空 & 数组长度大于0
        // 2. 获取的桶不为空
        if ((tab = table) != null && (n = tab.length) > 0 &&
            // 获取桶下标的公式都是通用的 `(n -1) & h`
            (e = tabAt(tab, (n - 1) & h)) != null)
        {// 对于桶中头节点的hash，对比成功就不需要遍历整个列表了
            if ((eh = e.hash) == h) {
                // 返回匹配的元素value
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            // 元素hash < 0的情况有以下三种:
            // 1. 数组正在扩容，Node的实际类型是ForwardingNode
            // 2. 节点为树的root节点，TreeNode
            // 3. 暂时保留的Hash, Node
            // 不同的Node都会调用各自的find()方法
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            // 如果头节点不是所需节点,且Map此时并未扩容
        	// 直接遍历桶中元素查找
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

##### 完整的获取流程如下:

1. 经过扰动函数获取`key`的hash,在获取之前会先判断tab是否为空以及长度
2. 通过`(n -1)& hash`获取的桶下表获取桶.
3. 判断`key`的hash和桶的头节点是否相等,相等则直接返回.
4. 若获得的桶头节点的`hash < 0`,表示**处于以下三种状态,则是通过调用各自实际节点类型的`find`方法获取元素.**
   1. 数组正在扩容，Node的实际类型是`ForwardingNode`
   2. 节点为树的root节点,节点类型为`TreeNode`
   3. 暂时保留的Hash, Node
5. 如果hash不相等,且头节点hash正常,之后**就是普通的链表遍历查找操作.**





## 扩容机制

- 不得不说,扩容部分的代码绝对是超一流的大师手笔!!!

##### addCount  扩容的监测

- `addCount`的作用:
  1.   增加`ConcurrentHashMap`的元素计数
  2.   前驱检测是否需要扩容,

```java
/**
 *   参数: 
 * 	 x -> 具体增加的元素个数
 *   check -> 如果check<0不检查时都需要扩容,
 */
private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
     	// 1. counterCells不为空
      	// 2. CAS修改baseCount属性成功
        if ((as = counterCells) != null ||
            // CAS增加baseCOunt
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            // 线程争用的状态标记
            boolean uncontended = true;
            // 1. 计数cell为null,或长度小于1
            // 2. 随机去一个数组位置为为空
            // 3. CAS替换CounterCell的value失败
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            // CAS增加CounterCell的value值失败会调用fullAddCount方法
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
    	// 根据`check >= 0`判断是否需要检查扩容
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            // 1. 如果元素总数大于sizeCtl,表示达到了扩容阈值
            // 2. tab数组不能为空,已经初始化
            // 3. table.length小于最大容,有扩容空间
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                // 根据数组长度获取一个扩容标志
                int rs = resizeStamp(n);
                if (sc < 0) {
                    // 如果sc的低16位不等于rs,表示标识符已经改变.				// 待补充
                    // 如果nextTable为空,表示扩容已经结束
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    // CAS替换sc值为sc+1,成功则开始扩容
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        //	调用transfer开始扩容,此时nextTable已经指定
                        transfer(tab, nt);
                }
                // `sc > 0`表示数组此时并不在扩容阶段,更新sizeCtl并开始扩容
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    // 调用transfer,nextTable待生成
                    transfer(tab, null);
                s = sumCount();
            }
        }
    }
```



##### helpTransfer  帮助扩容

```java
 /**
  * 参数：
  * tab -> 扩容的数组，一般为table
  * f -> 线程持有的锁对应的桶的头节点
  * 调用地方:
  * 1. `putVal`检测到头节点Hash为MOVED
  */
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        // 1.参数数组不能为空 
		// 2.参数f必须为ForwardingNode类型
        // 3.f.nextTab不能为空
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            // resizeStamp一顿位操作打的我头昏脑涨
            // 获取扩容的标识
            int rs = resizeStamp(tab.length);
            // Map仍处在扩容状态的判断
            // 1. 判断节点f的nextTable是否和成员变量的nextTable相同
            // 2. 判断传入的tab和成员变量的table是否相同
            // 3. sizeCtl是否小于0
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                // 两种不同的情况判断                
                // 一. 不需要帮助扩容的情况
                // 1. sc的高16位不等于rs
                // 2. sc等于rs+1
                // 3. sc等于rs+MAX_RESIZERS
                // 4. transferIndex <= 0, 这个好理解因为扩容时会分配并减去transferIndex,
                // 小于0时表示数组的区域已分配完毕
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                // 二. CAS `sc+1`并调用transfer帮助扩容.
                // 线程在帮助扩容时会对sizeCtl+1,完成时-1,表示标记
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
```



##### transfer 扩容的核心方法,负责迁移桶中元素

```java
  private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
      	// stride为此次需要迁移的桶的数目
      	// NCPU为当前主机CPU数目
      	// MIN_TRANSFER_STRIDE为每个线程最小处理的组数目
      	// 1. 在多核中stride为当前容量的1/8对CPU数目取整,例如容量为16时,CPU为2时结果是1
      	// 2. 在单核中stride为n就为当前数组容量
 		// !!! stride最小为16,被限定死.
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
      	// nextTab是扩容的过渡对象,所以必须要先初始化
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                // !!! 重点就在这 扩容后的大小为当前的两倍 --> n << 1
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
            	// 扩容失败,直接填充int的最大值
                sizeCtl = Integer.MAX_VALUE;
                // 直接退出
                return;	
           }
            // 更新成员变量
            nextTable = nextTab;
            // transferIndex为数组长度
            transferIndex = n;
        }
      	// 记录过渡数组的长度
        int nextn = nextTab.length;
    	// 此处新建了一个ForwardingNode用于后续占位
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        /**
          * 以上为数据准备部分,初始化过渡数组,记录长度,创建填充节点等操作
          * 以下时真正扩容的主要逻辑
          */
 		// 该变量控制迁移的进行,     
        boolean advance = true;
        boolean finishing = false; 			// 两个变量作用未知 finishing可能是此次扩容标记
  // 扩容的for循环里面可以分为两部分
 // 一. while循环里面确定需要迁移的桶的区域,以及本次需要迁移的桶的下标
      	// 这个i就是需要迁移的桶的下标
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
          	// 该while代码块根据if的顺序功能分别是
            // --i: 负责迁移区域的向前推荐,i为桶下标
            // nextIndex: 在没有获取负责区域时,检查是否还需要扩容
            // CAS: 负责获取此次for循环的区域,每次都为stride个桶
            while (advance) {
                int nextIndex, nextBound;
                // 这个`--i`每次都会进行,每次都会向前推进一个位置
                if (--i >= bound || finishing)
                    advance = false;
                // 因此如果当transferIndex<=0时,表示扩容的区域分配完
                else if ((nextIndex = transferIndex) <= 0) {
            		i = -1;
                    advance = false;
                // CAS替换transferIndex的值,新值为旧值减去分到的stride
                // stride就表示此次的迁移区域,nextIndex就代表了下次起点
                // 从这里可以看出扩容是从数组末尾开始向前推进的
                }else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    // bount为此次扩容的推进终点,下次起点
                    bound = nextBound;
                    // i此次扩容开始的桶下表
                    i = nextIndex - 1;
                    advance = false;
                }
            }
 // 二. 扩容的逻辑代码
        // 1. 此if判定扩容的结果,中间是三种异常值
              // 1). i < 0的情况时上面第二个if跳出的线程
          	  // 2). i > 旧数组的长度
           	  // 3). i+n大于新数组的长度
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                // 此阶段扩容结束后的操作
                // 1. 将nextTable置空,
                // 2. 将中间过渡的数组赋值给table
                // 3. sizeCtl变为1.5倍(2n-0.5n)
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    // 分别使用有符号左移,无符号右移
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                // CAS替换`sizeCtl-1`,表示本线程的扩容任务已经完成
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    //	表达式成立表示还有别的线程在执行扩容,直接退出
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    // 表达式成立,表示已经全部扩容完成.
                    finishing = advance = true;
                    // 提交前重新检查
                    i = n; 
                }
            }
       // 2. 扩容时发现负责的区域有空的桶直接使用ForwardingNode填充
            // ForwardingNode持有nextTable的引用
            else if ((f = tabAt(tab, i)) == null)
                // CAS替换
                advance = casTabAt(tab, i, null, fwd);
       // 3. 表示处理完毕
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
      // 4. 迁移桶的操作
            else {
                // sync保证原子性和可见性
                synchronized (f) {                
                    // 获取数组中的第i个桶的头节点
                    // 进入synchronized之后重新判断,保证数据的正确性没有在中间被修改
                    if (tabAt(tab, i) == f) {
                        // 此处扩容和HashMap有点像,分为了lowNode和highNode两个头结点
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                           	// true的话会重新
                            advance = true;
                        }
                        // 树的桶迁移操作
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }

```

##### 扩容触发线程逻辑

1. 在`addCount`方法中间检查元素个数是否达到扩容阈值(0.75 * table.length),超过则触发扩容,调用方法`transfer`.

   - 注意此时`sizeCtl`会被`CAS`替换为`(resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2`

2. **接下来就是`teansfer`的代码**:

   1. 根据`CPU`和当前容量算出每次扩容该分配的区域大小,最小为16,表示为`stride`.

   2. 若过渡数组`nextTab`未初始化,则先初始化数组.并使用`transferIndex`记录下旧数组长度,作为扩容长度.

   3. 以上扩容需要的数据准备完全开始具体的扩容操作:

   4. 在一个`while`循环中获取本次扩容包含的桶的范围,即`[transferIndex,transferIndex-stride]`的范围,`i`表示当前扩容的桶的下标.

   5. 三个判断,四段代码分别完成不同情况下的操作

      1. `i`数值异常,`< 0 || >= n || + n > nextn`,表示扩容已完成,且在`while`循环中没有分配的扩容任务.

         - 如果此时`finishing`参数为`true`表示整体扩容完成,且完成结束前的检查.

         - 如果`finishing`为`false`,则**`CAS`替换sizeCtl为sizeCtl-1**,表示一个线程完成扩容任务并需要退出.

           替换成功之后还会检查`sc`是否等于`addCount`进来时的值,不相等就直接`return`,表示还有线程未完成扩容任务.

      2.  `i`对应的桶为空,直接使用`ForwardingNode`填充头节点,表示此处正在扩容.并设`advance`为`true`

      3.   如果检查到节点`hash`为`Moved`表示当前节点为`ForwardingNode`,`advance`为`true`.

      4.   排除了上面三种情况,就是对应的桶的迁移工作,和`HashMap`有点像.结束后设置`advance`为`true`

   6. 之后会再回到第4步.

##### 扩容从属线程逻辑

1. 在`putVal`等元素操作方法中,发现获取的桶头节点为`ForwdingNode`就表示`ConcurrentHashMap`当前正在扩容,会马上调用`helpTransfer`帮助扩容.
2. `helpTransfer`中会有各种正确性判断,只有在以下三个条件都都满足时才会帮助扩容.
   1. `tab`是否不空
   2. 头节点是否为`ForwardingNode`
   3. 过渡数组`nextTable`是否初始化.
3. while循环中有以下两种判断
   1. 判断扩容过程是否需要帮助,有以下五种情况不需要帮助
      1. `sc >> 16 != rs ` - 标识符已经改变. 
      2. `sc == rs+1` - 触发扩容的线程已退出,扩容已经完成
      3. `sc == rs+MAX_RESIZER` - 参与扩容的线程达最大值
      4. `transferIndex <= 0 `  - 扩容区域已经分配完
   2. 排除以上不需要帮助的情况,就会调用`transfer`帮助扩容.



##### 扩容过程中sizeCtl的变化

1. `addCount` -> `sizeCtl = (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2`

   - 此处有个我很久才想通的点: 为什么在`helpTransfer`中会有判断`sizeCtl`高16位的操作,
   - **在此处赋值的时候就相当于将`resizeStamp(n)`的值推高16位,赋值给`sizeCtl`,而低16位则保存了这个2.也就是说在扩容的时候`sizeCtl`的高16为保存了标识符,而低16位保存了参与线程数目.**
   - 真他娘的是个天才.

   ![](https://chenbxxx.oss-cn-beijing.aliyuncs.com/%E6%89%A9%E5%AE%B9%E6%97%B6%E7%9A%84sizeCtl.png)

2.  有线程参与扩容 -> `sizeCtl = sizeCtl - 1`

3.  线程退出扩容 -> `sizeCtl = sizeCtl + 1`

4.  扩容完成 -> `sizeCtl = nextTab.length * 0.75` 

## 初始化方法

- 和`HashMap`一样,`ConcurrentHashMap`并不是在构造函数中就直接初始化底层的数组,而是在`put`等存方法中,判断是否需要扩容.

###### initTable 数组初始化函数

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // `sizeCtl`表示有别的数组正在初始化,让出CPU时间
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // CAS操作,以-1置换`sizeCtl`的值
        // 可以看出 `sizeCtl==-1`时,表示数组正在某个线程初始化
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                // 置换之后需要重新检测数组是否未初始化
                if ((tab = table) == null || tab.length == 0) {
                    // sc就是置换之前的sizeCtl.
                    // 此时sizeCtl作为初始容量.
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 初始化结束之后sc变为0.75n,是扩容阈值
                    sc = n - (n >>> 2);
                }
            } finally {
                // 为避免异常退出导致sizeCtl永久为-1,此处强制赋值.
                sizeCtl = sc;
            }
            break;
        }
    }
    // 返回了新建的数组地址
    return tab;
}
```

- `initTable`方法时在`putVal`而非构造函数,也算是`CHM`中的一种懒加载机制.
- **初始化的流程:**
  1. 检查是否有别的线程正在初始化,有就让出时间片,没有则进行下一步.
  2. 初始化之前,先将`sizeCtl`通过`CAS`置换为-1,表示正在初始化
  3. 以`sizeCtl`之前的值为初始容量,`sizeCtl`<=0时使用默认容量16
  4. 初始化结束,**将`sizeCtl`赋值为0.75*数组容量**<font size="2">(sizeCtl贯穿全篇,真的很重要)</font>

## 通用工具方法


###### 1. resizeStamp  获取扩容时的一个标记

```java
    static final int resizeStamp(int n) {
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }
```

- `Integer.numberOfLeadingZeros(n)` **返回的是n的32位二进制形式前面的0的个数**,例如值位16的`int(32位)`类型二进制表示为`000000...0010000`,1前面的就有27个0,返回就是27.
- `|`操作现在此处可以简单理解为加法。
- 整合起来作用就是:**获取n的有效位之前的0的个数加上1的15次方**.
- 暂时不清楚为什么要获取一个标Stamp

###### 2. spread 扰动函数

```java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

- 扰动函数,和`HashMap`中的`hash()`方法功能类似.
- `CHM`中的扰动函数除了将高16位于低16位异或之外又与上HASH_BITS,**可以有效降低哈希冲突的概率,使元素分散更加均匀.**

## 元素个数统计方法

ConcurrentHashMap 中的元素统计采用了特殊的方式。

使用了上文说到了 baseCount 和 ConuterCell 两个成员变量，统计的逻辑如下：

> **baseCount 表示的是基础的元素个数，而 CounterCell 数组中保存的是对应的各个桶中的变化的元素个数。**
>
> 所以统计的时候，需要从 CounterCell 数组中统计所有的个数加上 baseCount。



### 元素个数新增

```java
private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
                if (sc < 0) {
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount();
            }
        }
}


```



## 



#### Node数组的元素访问方法

###### 1. tabAt  以Volatile方式获取数组元素

```java
    @SuppressWarnings("unchecked")
	// tab: 数组   i : 下标
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }
```

###### 2. casTabAt 以`CAS`形式替换数组元素

```java
// tab: 原始数组  i:下标 c:对比元素 v:替换元素   
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
```

###### 3. setTabAt  以volatile方式更新数组元素

```java
// tab:原始数组 i:下标 v:替换元素
	static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```



#### Unsafe 静态块

- `Unsafe`是一块Java开发人员都很少接触的区域,但这里还是简单了解一下

```java
    private static final sun.misc.Unsafe U;
	// sizeCtl属性的偏移地址
    private static final long SIZECTL;
	// transferIndex属性的偏移地址
    private static final long TRANSFERINDEX;
	// baseCount的偏移地址
    private static final long BASECOUNT;
	// cellsBusy的偏移地址
    private static final long CELLSBUSY;
	// CounterCell类中value的偏移地址
    private static final long CELLVALUE;
	// Node数组第一个元素的偏移地址
    private static final long ABASE;
	// Node数组中元素的增量地址,与ABASE配合使用能访问到数组的各元素
    private static final int ASHIFT;

    static {
        try {
            U = sun.misc.Unsafe.getUnsafe();
            Class<?> k = ConcurrentHashMap.class;
            // 先通过反射获取到对应的属性值,再通过Unsafe类获取属性的偏移地址
            SIZECTL = U.objectFieldOffset
                (k.getDeclaredField("sizeCtl"));
            TRANSFERINDEX = U.objectFieldOffset
                (k.getDeclaredField("transferIndex"));
            BASECOUNT = U.objectFieldOffset
                (k.getDeclaredField("baseCount"));
            CELLSBUSY = U.objectFieldOffset
                (k.getDeclaredField("cellsBusy"));
            Class<?> ck = CounterCell.class;
            CELLVALUE = U.objectFieldOffset
                (ck.getDeclaredField("value"));
            Class<?> ak = Node[].class;
            // 获取数组中第一个元素的偏移地址
            ABASE = U.arrayBaseOffset(ak);
            // 获取数组的增量地址
            int scale = U.arrayIndexScale(ak);
            if ((scale & (scale - 1)) != 0)
                throw new Error("data type scale not a power of two");
            ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);
        } catch (Exception e) {
            throw new Error(e);
        }
    }
```

