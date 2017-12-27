
title: hive中使用向量化查询引擎Vectorized-Query-Execution
date: 2017-04-25 19:59:52
tags: [youdaonote]
---

简介
---
向量化查询引擎可以在执行scan, filter, aggregate, join等操作的时候很大程度上降低CPU损耗。标准sql执行系统每次只处理一行数据，内部执行的时候会包括很长的代码路径和重要的metadata解析。向量化查询引擎简化了这个过程，一次读取1024行数据，在这个数据块中，每个字段都被存储成向量化的结构(一个原是数据类型的数组)。对于一些简单的算法可以通过快速迭代这个vector完成，只需要在循环过程中调用甚至不用调用几个函数。这些loop以一种流式方式编译，在固定时间内结束，为了提高效率，会使用processor pipline和内存缓存。[详细设计](https://issues.apache.org/jira/browse/HIVE-4160)

个人理解：就是每行都执行很小的计算，但是要处理N多次。改成一次处理多行，同时针对多行做计算处理，节省CPU时间。

使用
---
#### 启用
要使用向量化查询引擎，必须先把hive表存储成ORC格式的。
```
set hive.vectorized.execution.enabled = true;
```
默认是没有开启向量化查询引擎的，需要手动开启。


#### 支持的数据类型与操作
- tinyint
- smallint
- int
- bigint
- boolean
- float
- double
- decimal
- date
- timestamp (see Limitations below)
- string
其他的数据类型还是会按照传统方式一行行处理。

支持的表达式：
- 简单算法: +, -, *, /, %
- AND, OR, NOT
- 比较 <, >, <=, >=, =, !=, BETWEEN, IN ( list-of-constants ) as filters
- 布尔表达式 (non-filters) using AND, OR, NOT, <, >, <=, >=, =, !=
- IS [NOT] NULL
- 所有的匹配函数 (SIN, LOG, etc.)
- string相关函数 SUBSTR, CONCAT, TRIM, LTRIM, RTRIM, LOWER, UPPER, LENGTH
- 类型转换
- UDF
- 日期函数 (YEAR, MONTH, DAY, HOUR, MINUTE, SECOND, UNIX_TIMESTAMP)
- IF 条件表达式

UDF通过向后兼容的方式支持，不过执行的时候就是执行向量化查询引擎，但是没有内置的快。向量化的filter操作是从左到右执行，所以最好把UDF放在带有and的where语句的最后：
```sql
where column1 = 10 and myUDF(column2) = "x"
```

限制
---
Timestamps 只能是 1677-09-20 到2262-04-11. 
