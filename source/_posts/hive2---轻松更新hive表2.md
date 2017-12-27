
title: hive2---轻松更新hive表2
date: 2017-11-22 17:58:13
tags: [youdaonote]
---

前面讲了使用MERGE,UPDATE,DELETE更新hive数据。现在我们进一步谈一下hive管理slowly-changing dimensions(SCDs，就是缓慢更新的维度表)的策略。在数据仓库中，SCDs更新数据是无规律的。对应于不同的业务需求，有不同的策略。假如你要一个用户维度表的所有历史，以跟踪某个用户随时间的变化情况。还有些情况，我们只关心最新的维度状态 。

下面是三种SCD更新策略：
- 使用新数据覆盖旧数据。很简单，如果只是要同步最新状态的话，就用这个，但是会丢失历史维度值。
- 添加带有version的新数据行。可以追踪到所有历史。但是随着时间的退役，可能会特别大，还有就是查询的时候需要只看最新版本的维度值。
- 添加新数据行，管理有限版本的历史。有一些历史数据，然是控制在一定范围内。
- 

这个blog是讲一下怎样使用hive的MERGE来管理SCD。所有的例子都可以在[这里](https://github.com/cartershanklin/hive-scd-examples)找到。管理SCD是很麻烦的事情，所以最好能够用一些工作，比如[数仓工具](https://www.amazon.com/Data-Warehouse-Toolkit-Complete-Dimensional/dp/0471200247)

#### SCD管理策略一览

![](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2017/08/1Slowly-changing-dimensions-1024x695.png)


#### 基础
所有的是例子都是从一个外部表，copy到hive的managed table，这个managed table就是merge target。第二个外部表，代表第二次从某个系统全量dump出来的数据。这两个外部表是一样的csv文件，包含字段:ID,Name, Email,State。初始化的数据有1000条，第二次数据有1100条，其中包含100个新纪录和93个需要更新项。

#### 第一种策略

有就直接替换，没有就添加。

```
merge into
 contacts_target
using
 contacts_update_stage as stage
on
 stage.id = contacts_target.id
when matched then
 update set name = stage.name, email = stage.email, state = stage.state
when not matched then
 insert values (stage.id, stage.name, stage.email, stage.state);
```

值得注意的是，上面的操作也是单独一个，是原子且独立的，如果出现错误会正确rollback。在SQL-on-Hasdoop方式里提供这些特性是很困难的，但是hive的MERGE操作就实现了。

#### 第二种策略

保留所有的历史版本，提供单独的版本相关字段:ValidFrom, ValidTo。

![](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2017/08/2Hive-Type-2_1-1024x319.png)

我们可以使用这个策略来满足并发用户对于正在更新的数据的数据读取。

```
merge into contacts_target
using (
 — The base staging data.
 select
contacts_update_stage.id as join_key,
contacts_update_stage.* from contacts_update_stage
 union all
— Generate an extra row for changed records.
 — The null join_key forces records down the insert path.
 select
   null, contacts_update_stage.*
 from
   contacts_update_stage join contacts_target
   on contacts_update_stage.id = contacts_target.id
 where
   ( contacts_update_stage.email <> contacts_target.email
     or contacts_update_stage.state <> contacts_target.state )
   and contacts_target.valid_to is null
) sub
on sub.join_key = contacts_target.id
when matched
 and sub.email <> contacts_target.email or sub.state <> contacts_target.state
 then update set valid_to = current_date()
when not matched
 then insert
 values (sub.id, sub.name, sub.email, sub.state, current_date(), null);
```
需要注意的是，using语句中对于每个更新的row会输出2个record。这些record会有一个null join key(就会成为一个insert了), 还会有一个valid jonk key(这是一个update)。如果都过去i安眠文章的话，其实有类似于在分区之间移动数据，只不过是使用update而不是delete。

看下93条记录的情况。
![](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2017/08/2Hive-Type-2_2-1024x304.png)

#### 第三种类型

第二种类型其实挺强大的了，不过比较复杂，而且维度表会无限增长下去。第三种策略中维度表基本跟数据源大小差不多，但是只提供部分历史。

下面我们就只保存上一个版本的纬度值

![](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2017/08/3Hive-Type-3_1.png)


当update的时候，我们任务就是把当前的版本放到last的值里。

```sql
merge into
 contacts_target
using
 contacts_update_stage as stage
on stage.id = contacts_target.id
when matched and
 contacts_target.email <> stage.email
 or contacts_target.state <> stage.state — change detection
 then update set
 last_email = contacts_target.email, email = stage.email, — email history
 last_state = contacts_target.state, state = stage.state  — state history
when not matched then insert
 values (stage.id, stage.name, stage.email, stage.email,
 stage.state, stage.state);
```
我们看到相比第二种策略，这种就简单多了。

![](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2017/08/3Hive-Type-3_2.png)

#### 一个更简单的变化追踪方法

如果有很多字段需要比较，那么对于变化的探测逻辑会比较笨重。幸运的是，hive引入了一个hash UDF让这个变得简单，可以接收任意数量的参数，然会一个checksum。


```
merge into
 contacts_target
using
 contacts_update_stage as stage
on stage.id = contacts_target.id
when matched and
 hash(contacts_target.email, contacts_target.state) <>
   hash(stage.email, stage.state)
 then update set
 last_email = contacts_target.email, email = stage.email, — email history
 last_state = contacts_target.state, state = stage.state  — state history
when not matched then insert
 values (stage.id, stage.name, stage.email, stage.email,
 stage.state, stage.state);
```
好处就是，不管有多少个字段要比较，我们对于代码的修改可以几乎没有。


#### 结论
SCD管理是数据仓库机器重要的概念，是一个有很多策略和方法实现的子模块。有了ACID MERGE，hive让我们在hadoop上管理SCD变的简单。



参考：https://zh.hortonworks.com/blog/update-hive-tables-easy-way-2/


