
title: HIVE创建bucketed表
date: 2017-04-20 18:54:03
tags: [youdaonote]
---

建表语句
```
CREATE TABLE user_info_bucketed(user_id BIGINT, firstname STRING, lastname STRING)
COMMENT 'A bucketed copy of user_info'
PARTITIONED BY(ds STRING)
CLUSTERED BY(user_id) INTO 256 BUCKETS;
```
基于字段user_id分桶的。

使用
```sql
set hive.enforce.bucketing = true;  -- (Note: Not needed in Hive 2.x onward)
FROM user_id
INSERT OVERWRITE TABLE user_info_bucketed
PARTITION (ds='2009-02-25')
SELECT userid, firstname, lastname WHERE ds='2009-02-25';
```

hive怎样把row打散到不同的bucket中去的呢？一般是决定于hash函数，使用什么hash函数又取决于分桶字段的类型。比如整型字段就使用hash_int函数。

注意，如果分桶字段类型不同于insert进来的数据类型会出错的，或者手动使用不同类型的数据执行分桶操作也会出错。只要设置了`set hive.enforce.bucketing=true`，就能够正确的把数据发布到对应的地方。


再来一个例子
Bucketed Sorted Table
```sql
CREATE TABLE page_view(viewTime INT, userid BIGINT,
     page_url STRING, referrer_url STRING,
     ip STRING COMMENT 'IP Address of the User')
 COMMENT 'This is the page view table'
 PARTITIONED BY(dt STRING, country STRING)
 CLUSTERED BY(userid) SORTED BY(viewTime) INTO 32 BUCKETS
 ROW FORMAT DELIMITED
   FIELDS TERMINATED BY '\001'
   COLLECTION ITEMS TERMINATED BY '\002'
   MAP KEYS TERMINATED BY '\003'
 STORED AS SEQUENCEFILE;
```
这个表根据userid分桶，每个bucket中按照viewTime升序排列。这种组织结果允许用户更高效的抽取clustered的字段，这里就是userid字段了。排序属性让内部操作能够快速处理估算查询的任务。MAP KEYS和COLLECTION ITEMS关键词可以用来处理字段是list或者map的情况。

CLUSTERED BY 和 SORTED BY并不影响insert数据。用户需要注意reducer数必须要跟bucket的数量一致，在query语句中还要使用CLUSTER BY 和SORT BY语句。

还有一种倾斜表。适用于某些表中确认包含倾斜数据的情况。通过指定倾斜key，hive会把他们打散到不同的文件中。

```sql
CREATE TABLE list_bucket_single (key STRING, value STRING)
  SKEWED BY (key) ON (1,5,6) [STORED AS DIRECTORIES];
```
下面这个是有两个倾斜key
```
CREATE TABLE list_bucket_multiple (col1 STRING, col2 int, col3 STRING)
  SKEWED BY (col1, col2) ON (('s1',1), ('s3',3), ('s13',13), ('s78',78)) [STORED AS DIRECTORIES];
```


参考：
- https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL+BucketedTables
- https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL#LanguageManualDDL-BucketedSortedTables
