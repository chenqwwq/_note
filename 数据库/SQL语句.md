# SQL 语句

> 自认为 SQL 写的不太行，总结一下穿插 Leetcode 上的题目。



## SQL 基础

### group by 

分组函数，内容将根据 group by 指定的列进行分组。

SELECT 的内容必须为 group by 中声明的字段，或者为聚合函数输出。

例如：

```mysql
SELECT DEPT, sum( SALARY ) AS total
FROM STAFF
GROUP BY DEPT;
```

group  by 中指定了根据 DEPT 字段分组，所以在 SELECT 后面只能根据 DEPT 一个自然列，如果包含其他例如 NAME 等字段会抛出异常。

sum 属于内置的几种聚合函数（组函数）之一，会对每个分组的内容分别求值。

以下是常用的几种聚合函数：

- count() : 计算表中的记录数（行数）
- sum() : 计算表中数值列中数据的合计值
- avg() : 计算表中数值列中数据的平均值
- max() : 求出表中任意列中数据的最大值
- min() : 求出表中任意列中数据的最小值

组函数默认忽略 NULL 值，并且不允许出现嵌套（MAX(SUM(salary)) 此种形式会抛出异常）。



#### having 过滤

having 用于对分组的结果进行过滤。

因为 where 子句比 group by 先执行，而组函数必须在分完组之后才执行，且分完组后必须使用 having 子句才能进行结果集的过滤。

> where 和 having 的区别主要在于执行时间。

因此，where 中不能使用聚合函数（因为都还没有分组，但是 having 可以。



#### group_concat() 函数

用于将分组后的值进行连接，默认使用逗号。

```mysql
SELECT DEPT, group_concat( SALARY ) AS total
FROM STAFF
GROUP BY DEPT;
```

以上函数会返回所有 SALARY 使用逗号分割的内容。





### limit offset

MySQL 的分页关键字。

```mysql
SELECT * FROM Employee ORDER BY salary DESC LIMIT 1 OFFSET 4;

SELECT * FROM Employee ORDER BY salary DESC LIMIT 4,1;
```

以上两条相同意思都表示第五大的薪水。





### 联表语句

INNER JOIN，LEFT JOIN，RIGHT JOIN





### 联合函数

UNION （UNION DISTANT），UNION ALL

将两个 SELECT 的结果集进行合并，合并的两个查询在结果集的字段数量和类型上必须保持一致。





### 参考

- [MySQL最常用分组聚合函数](https://www.cnblogs.com/geaozhang/p/6745147.html)



## Leetcode 习题

### [175. 组合两个表](https://leetcode-cn.com/problems/combine-two-tables/)

简单的联表操作。

