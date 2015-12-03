title: mysql新技能-ON DUPLICATE KEY UPDATE
date: 2015-12-08 10:30:42
tags: [mysql, sql]
---

向数据库插入数据的时候，我们会遇到一种情形：如果数据库中没有此条记录，就直接插入；如果已经存在此条记录，就更新其部分的字段值。这里定位“此条记录”是指在数据库中是唯一的，确定某个标识可以在数据库中唯一定位某一条记录。
玩儿hive的同学请注意意思，虽R是insert overwrite的大概思想，但是不会覆盖整个分区数据，而仅仅针对“此条记录”。


ON DUPLICATE KEY不会指定具体的唯一key是哪个,因为并没有必要，这个语句要求table里有unique key,或者primary key, 它会自动比较这个key是不是有重复，如果有重复就update，没有的话就直接insert。

```sql
INSERT INTO table (a,b,c) VALUES (1,2,3)
  ON DUPLICATE KEY UPDATE c=c+1;

UPDATE table SET c=c+1 WHERE a=1;
```
如果a=1这条记录存在，而且a是unique key的话，那么这两个句子是等同的。
这个句子还支持一下方式，就是可以同时更新多个字段的值，而且可以相互引用：
```sql
INSERT INTO table (a,b,c) VALUES (1,2,3),(4,5,6)
  ON DUPLICATE KEY UPDATE c=VALUES(a)+VALUES(b), b=2;
```



