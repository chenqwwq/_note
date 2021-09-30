# ConcurrentSkipListMap 

---

[TOC]

---



## 概述

ConcurrentSkipListMap 是 JUC 中提供的对**跳表**的并发安全的实现，跳表是**基于二分查找的原理，采用多级索引的形式加快查询的有序列表**。

> Redis 中的 ZSet 也使用了跳表作为其实现。







## 源码实现



```java
// ConcurrentSkipListMap#findPredecessor
private Node<K,V> findPredecessor(Object key, Comparator<? super K> cmp) {
    if (key == null)
        throw new NullPointerException(); // don't postpone errors
    for (;;) {
        // head 是顶层的索引节点
        // r 是同层的索引
        for (Index<K,V> q = head, r = q.right, d;;) {
            // 存在同层索引
            if (r != null) {
                Node<K,V> n = r.node;
                K k = n.key;
                // 节点被删的情况
                if (n.value == null) {
                    if (!q.unlink(r))
                        break;           // restart
                    r = q.right;         // reread r
                    continue;
                }
                // 和右侧节点比，如果 key 大，则继续向右
                if (cpr(cmp, key, k) > 0) {
                    q = r;
                    r = r.right;
                    continue;
                }
            }
            // 两种情况
            // 1. 没有同层的索引可
            // 2. 找到了比 key 大的同层索引节点
            // 此时开始下沉，注意是 q 也就是比 key 大的节点的前一个节点
            if ((d = q.down) == null)
                // 无法下沉说明已经到基础数据这一层了，直接返回
                return q.node;
            // 可以下沉,则继续遍历
            q = d;
            r = d.right;
        }
    }
}

```

> 跳表中寻找节点，都是**从左上角的顶层索引**开始：
>
> 1. 查找同层大于 Key 的节点
> 2. 索引下沉
> 3. 继续第 1 步，直到无法下沉，返回最终数据