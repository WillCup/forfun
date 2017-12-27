
title: presto手札
date: 2017-12-19 15:30:45
tags: [youdaonote]
---

#### outputFormat should not be accessed from a null StorageFormat

对于hive的一些外部表，presto不能支持，比如基于hbase、或者ES的hive外部表。

```sql
presto:default> select table_schema, table_name, column_name, is_nullable, data_type from information_schema.columns;

Query 20171219_073016_00015_4zvkw, FAILED, 2 nodes
Splits: 17 total, 0 done (0.00%)
0:00 [0 rows, 0B] [0 rows/s, 0B/s]

Query 20171219_073016_00015_4zvkw failed: outputFormat should not be accessed from a null StorageFormat

```


所以略过这种特殊表的才能成功：
```sql
select table_schema, table_name, column_name, is_nullable, data_type from information_schema.columns where table_name not in ('hive_table_based_on_hbase');

```
参考： https://github.com/prestodb/presto/issues/7721
