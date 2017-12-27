
title: hive杂记
date: 2016-12-05 17:05:00
tags: [youdaonote]
---

Execution failed with exit status: 3
---
使用mapjoin的时候出现map端内存溢出。通常有两个原因：
 - 当面对压缩文件的时候，mapjoin基于的数据大小参数未必准确。或许解压开的数据会大得多。可以降低`hive.smalltable.filesize`来调整，或者增大`hive.mapred.local.mem`来让map任务获取更大一些的内存。
 - `hive.mapred.local.mem`没起作用。参考：https://issues.apache.org/jira/browse/HADOOP-10245
 

MapJoin的过程：

- 本地
    - 从数据源读取数据
    - 在内存构建hashtable
    - 把hashtable写到本地磁盘
    - 上传hashtable到hdfs
    - 把hashtable添加到DistributeCache
- Map任务中
    - 从DistributeCache读取到内存中
    - 基于内存中的hashtable匹配每条记录的key
    - join到合适的，写到output中
- 无reduce任务


参考：
- https://cwiki.apache.org/confluence/display/Hive/LanguageManual+JoinOptimization
- http://blog.csdn.net/yhao2014/article/details/42675011



org.apache.hadoop.hive.serde2.lazy.objectinspector.LazyListObjectInspector cannot be cast to org.apache.hadoop.hive.serde2.objectinspector.PrimitiveObjectInspector
=
load data时确定了data的数据类型，load完成后如果改变hive表中列的类型后select会出错



grouping set 聚合函数多变量问题
=

```
select 
nvl(a.loan_day,'-') loan_day, 
nvl(b.is_newuser, '-') is_new, 
sum(case when to_date(a.create_time) <=a.dt then a.apply_amount else 0 end) loan_apply_amount_all,	
count(case when to_date(a.create_time) <=a.dt then a.id else null end) loan_apply_count_all,	
sum(case when to_date(a.loan_time) <=a.dt then a.apply_amount else 0 end) loan_suc_amount_all,	
count(case when to_date(a.loan_time) <=a.dt then a.id else null end) loan_suc_count_all,	
sum(case when to_date(a.create_time) =a.dt then a.apply_amount else 0 end) loan_apply_amount_inc,	
count(case when to_date(a.create_time) =a.dt then a.id else null end) loan_apply_count_inc,	
sum(case when to_date(a.loan_time) =a.dt then a.apply_amount else 0 end) loan_suc_amount_inc,	
count(case when to_date(a.loan_time) =a.dt then a.id else null end) loan_suc_count_inc, 
a.dt statistics_date,	
'weibo' channel ,
a.dt dt
from fact_level.fact_loan_info a 
left join ( 
select id,case when to_date(new_user_date)=dt then 'newuser' else 'olduser' end is_newuser 
from fact_level.fact_user_info 
where dt between '2017-02-13' and '2017-03-27' and is_new=1 
)b 
on a.user_id=b.id 
where a.dt between '2017-02-13' and '2017-03-27' and a.loan_unit = '002007001' and to_date(a.create_time)<=a.dt and to_date(a.create_time)>='2017-02-13' 
group by a.dt,a.loan_day,b.is_newuser 
grouping sets (a.dt,(a.dt,a.loan_day,b.is_newuser),(a.dt,a.loan_day),(a.dt,b.is_newuser)) 

```

报错：
```
Error while compiling statement: FAILED: SemanticException [Error 10210]: Grouping sets aggregations (with rollups or cubes) are not allowed if aggregation function parameters overlap with the aggregation functions columns
```

字面意思上看就是grouping set的聚合函数里不能出现聚合函数的参数超过聚合函数的字段数。 对应上面，就是那些case when里都是传入了两个参数create_time和dt。所以不支持。下面的sql就能够正常运行
```
select 
nvl(a.loan_day,'-') loan_day, 
nvl(b.is_newuser, '-') is_new, 
sum(case when to_date(a.create_time) <='2017-03-27' then a.apply_amount else 0 end) loan_apply_amount_all,	
count(case when to_date(a.create_time) <='2017-03-27' then a.id else null end) loan_apply_count_all,	
sum(case when to_date(a.loan_time) <='2017-03-27' then a.apply_amount else 0 end) loan_suc_amount_all,	
count(case when to_date(a.loan_time) <='2017-03-27' then a.id else null end) loan_suc_count_all,	
sum(case when to_date(a.create_time) ='2017-03-27' then a.apply_amount else 0 end) loan_apply_amount_inc,	
count(case when to_date(a.create_time) ='2017-03-27' then a.id else null end) loan_apply_count_inc,	
sum(case when to_date(a.loan_time) ='2017-03-27' then a.apply_amount else 0 end) loan_suc_amount_inc,	
count(case when to_date(a.loan_time) ='2017-03-27' then a.id else null end) loan_suc_count_inc, 
a.dt statistics_date,	
'weibo' channel ,
a.dt dt
from fact_level.fact_loan_info a 
left join ( 
select id,case when to_date(new_user_date)=dt then 'newuser' else 'olduser' end is_newuser 
from fact_level.fact_user_info 
where dt between '2017-02-13' and '2017-03-27' and is_new=1 
)b 
on a.user_id=b.id 
where a.dt between '2017-02-13' and '2017-03-27' and a.loan_unit = '002007001' and to_date(a.create_time)<=a.dt and to_date(a.create_time)>='2017-02-13' 
group by a.dt,a.loan_day,b.is_newuser 
grouping sets (a.dt,(a.dt,a.loan_day,b.is_newuser),(a.dt,a.loan_day),(a.dt,b.is_newuser)) 
```

因为上面这个sql就只有一个参数了`create_time`或者`loan_time`。然而我们是不想写死的，我们要的日期是要在where条件里面限制。
那就通过嵌套的方式进行实现，只需要绕开grouping set不能支持聚合函数太多参数这个约束就行了。

```
select
xx.loan_day, 
xx.is_new, 
sum(xx.loan_apply_amoun) loan_apply_amoun_all,	
xx.dt
from 
(
select 
nvl(a.loan_day,'-') loan_day, 
nvl(b.is_newuser, '-') is_new, 
case when to_date(a.create_time) <=a.dt then a.apply_amount else 0 end loan_apply_amoun,	
a.dt statistics_date,	
'weibo' channel ,
a.dt dt
from fact_level.fact_loan_info a 
left join ( 
select id,case when to_date(new_user_date)=dt then 'newuser' else 'olduser' end is_newuser 
from fact_level.fact_user_info 
where dt between '2017-02-13' and '2017-03-27' and is_new=1 
)b 
on a.user_id=b.id 
where a.dt between '2017-02-13' and '2017-03-27' and a.loan_unit = '002007001' and to_date(a.create_time)<=a.dt and to_date(a.create_time)>='2017-02-13' 
)xx
group by xx.dt,xx.loan_day,xx.is_new 
grouping sets (xx.dt,(xx.dt,xx.loan_day,xx.is_new),(xx.dt,xx.loan_day),(xx.dt,xx.is_new)) 
```

PS: 并不明白为什么参数不能过多，理论上是可以支持的。有兴趣的同学可以参考一下hive 源码咯。

我写个邮件问问hive用户组的人。

