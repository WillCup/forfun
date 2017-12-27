
title: hive2---轻松更新hive表1
date: 2017-11-22 17:07:16
tags: [youdaonote]
---

首先是merge、insert、update、delete。

以前，让hive里的数据持续更新是需要很复杂的成本的，很难去维护与执行。HDP2.6借助hive里的merge语法彻底地简化了数据维护成本，完成了INSERT, UPDATE, DELETE能力。

这个blog会说明怎样解释以下三种问题：
- hive update，从RDBMS同步数据到hive
- 更新hive里数据的分区
- 选择性地mask或者purge数据

#### 基础操作：SQL MERGE, UPDATE AND DELETE
MERGE是SQL 2008标准里的，是一个强大的SQL语句，它可以在同一个statement中inert，update，delete数据。MERGE让两个系统一致性工作变得简单。咱们看一下MERGE的语法：
```
MERGE INTO <target table>
 USING <table reference>
ON <search condition>
 <merge when clause>...
WHEN MATCHED [ AND <search condition> ]
THEN <merge update or delete specification>
WHEN NOT MATCHED [ AND <search condition> ]
THEN <merge insert specification>
```
WHEN MATCHED/WHEN NOT MATCHED语句可以无限量的。

我们也会使用到比较熟悉的UPDATE，语法
```
UPDATE <target table>
SET <set clause list>
[ WHERE <search condition> ]
```


当没必要把insert 和update的数据在一个sql statement里进行合并的时候，就可以使用update了。

#### 保持数据fresh

在HDP2.6中，需要先做两个工作：
- 开启Hive transaction。
- 我们table必须是一个transactional table。就是说这个table必须是clustered的，必须是ORCFile存储格式，而且有一个table属性：transactional=true。下面是一个例子：
```
create table customer_partitioned
 (id int, name string, email string, state string)
 partitioned by (signup date)
 clustered by (id) into 2 buckets stored as orc
 tblproperties("transactional"="true");
```


#### 例1：HIVE UPSERT
假设我们有一个源数据库，想要load进hadoop来运行ing大批量的分析。这个RDBMS中的数据不断的被添加和修改，而且并没有log告诉你哪些数据有变化【就是说没有binlog】。最简单的处理就是每24小时完整的copy一下这个RDBMS的数据镜像。

![](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2017/08/RDBMS-Source.png)

下面我们创建table：
```
create table customer_partitioned
(id int, name string, email string, state string)
 partitioned by (signup date)
 clustered by (id) into 2 buckets stored as orc
 tblproperties("transactional"="true");
```


假设我们的数据在Time = 1的时候是这样的：
![](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2017/08/Time-equals-1.png)

在Time = 2的时候是这样的
![](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2017/08/Time-equals-2.png)

Upsert操作是吧update和insert放在一个操作里，这样我们就不用关心这些数据是不是原来就已经存在于目标table中。MERGE就是用来做这个事情的：
```
merge into customer_partitioned
 using all_updates on customer_partitioned.id = all_updates.id
 when matched then update set
   email=all_updates.email,
   state=all_updates.state
 when not matched then insert
   values(all_updates.id, all_updates.name, all_updates.email,
   all_updates.state, all_updates.signup);
```

注意我们的两个when条件语句是用来管理update或者insert的。在merge过后，这个managed table就与Time=2的staged Table完全一样了，而且所有数据也都在其对应的分区中。

#### 例2：更新hive partition
hive中很多会用日期作为partition策略。这样可以简化数据加载，提升性能。只是有时我们偶尔会出现数据进入错误的分区的情形。例如，假设用户数据是由一个第三方提供的，里面包含一个用户的singup日期。如果这个第三方数据提供者提供的数据开始有问题，后面又修正了，那么前面的在错误分区里的数据就应该被清除了。

![初始数据](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2017/08/update-hive-partitions.png)


![修复后的数据](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2017/08/second-load.png)

注意到ID为2的数据第一次跟第二次的signup日期是不一样的，这就需要更新2017-01-08分区，从里面把它删除，然后把它加入到2017-01-10里面去。

在MERGE出现之前，基本不可能管理这些分区裱花的。Hive的MERGE statement不是原生支持更新partition key，但是有一个小的技巧。 We introduce a delete marker which we set any time the partition keys and UNION this with a second query that produces an extra row on-the-fly for each of these non-matching records. 
```
merge into customer_partitioned
 using (
-- Updates with matching partitions or net new records.
-- 更新符合分区的或者
select
  case
       when all_updates.signup <> customer_partitioned.signup then 1
       else 0
     end as delete_flag,
     all_updates.id as match_key,
     all_updates.* from
    all_updates left join customer_partitioned
   on all_updates.id = customer_partitioned.id
      union all

 -- Produce new records when partitions don’t match.
 -- 分区不匹配的时候，生成新的record
     select 0, null, all_updates.*
     from all_updates, customer_partitioned where
     all_updates.id = customer_partitioned.id
     and all_updates.signup <> customer_partitioned.signup
 ) sub
on customer_partitioned.id = sub.match_key
 when matched and delete_flag=1 then delete
 when matched and delete_flag=0 then
   update set email=sub.email, state=sub.state
 when not matched then
   insert values(sub.id, sub.name, sub.email, sub.state, sub.signup);
```
在MERGE处理过这个managed table之后，它就跟源数据表完全一致了。虽然过程中有分区的修改，但是这是一个操作，是原子、且独立的操作。

#### 例3：mask或者purge hive的数据
假设有一天你们公司的安全部门过来，让我们把某个用户的所有数据进行mask或者purge操作。那么我们就需要花费很长的时间对很多收到影响的分区进行数据重写。

假设有一个contact table
![](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2017/08/contracts-table.png)

我们对应的hive表是：
```
create table contacts
 (id int, name string, customer string, phone string)
 clustered by (id) into 2 buckets stored as orc 
tblproperties("transactional"="true");
```

安全部门提出以下要求：
##### 把MaxLeads的所有的电话号码mask掉
我们可以使用hive内置的mask方法
```
update contacts set phone = mask(phone) where customer = 'MaxLeads';
```

##### 把所有LeadMax的记录都purge掉
```
delete from contacts where customer = 'LeadMax';
```

##### 把给定id列表的所有记录都删掉
```
delete from contacts where id in ( select id from purge_list );
```

#### 结论
hive的MERGE和ACID事务让hive里的数据管理工作变得简单、强大、而且兼容于现有的EDW平台。


参考：https://zh.hortonworks.com/blog/update-hive-tables-easy-way/
