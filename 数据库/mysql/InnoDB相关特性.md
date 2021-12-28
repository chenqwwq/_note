# InnoDB 的相关特性





---

[TOC]

---

## 一、 Change Buffer（修改缓存

Change Buffer 的主要作用就是**缓存对二级非唯一索引的修改**（早期只在 Insert 中生效，称为 Insert Buffer。

Changer Buffer 属于日志的一种，在 InnoDB 底层的 Buffer Pool 中会占据一定的空间。

<br>

在一次数据更新中，如果没有 Change Buffer 会需要将数据所有的索引树获取到之后再在 Buffer Pool 里面做更新，而 Change Buffer 一定程度上减少了对应的操作。

Change Buffer 会在适当的时候进行 Merge，例如当索引页被加载到 Buffer Pool 的时候，或者服务空闲的时候，服务关闭之前等等。

<br>

可以和 redo log 做类比，redo log 减少了随机写的操作，而 Change Buffer 减少了随机读的操作（对于磁盘操作顺序操作比随机操作快了好几倍。

 

## 二、Double Write（两次写



## 三、Flush Neighbor Page（刷新邻接页

刷脏页的时候连带着将附近的一起刷了（处处透露着。





## InnoDB 的内存管理（LRU

InnoDB 底层会申请一片的 Buffer Pool 用于保存数据页以及其他数据。

数据页会根据 LRU 进行保存，InnoDB 的 LRU 经过一定的优化，链表将其前 3/8 的部分作为热数据区，后面的属于冷数据区，中间点可以称之为 midpoint。

读取的新数据并不是直接添加到链表的头部，而是添加到冷区头部，经过一定的时间时候才会进入到热区。

此类优化适合数据库查询相结合的，因为部分查询可能会大量查询到无用的数据页，如果一股脑全部填充到首部会将真实的热数据冲散。

