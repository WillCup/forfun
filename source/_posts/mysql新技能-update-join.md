title: mysql新技能-update join
date: 2015-12-08 10:56:30
tags: [mysql]
---

上面说到了insert overwrite或者insert update的思想。这里说另一种场景，就是我们需要使用a表的直接或者间接数据来更新b表里的数据内容。比如a是所有订单，而b是所有商家订单总金额统计，业务就自然是定时计算a里所有订单的金额汇总，然后udpate到b表里面去。
*这里其实也有insert update的需要，比如判断某商家是否存在，但是我们这里只关注update*

```sql
update
    b 
join
(
	select 
		poi_id,
		sum(fee) total_act_cost,
		sum(fee - poi_charge_fee) act_cost
	from
		a
	where 
		ctime between #{start_time} and #{end_time}
		and valid=1
		and status=9
	group by 
		poi_id
) temp 
on 
	a.poi_id = b.poi_id
set 
	b.total_act_cost = temp.total_act_cost,
	b.act_cost = temp.act_cost,
	b.act_flag=1
where 
	b.order_time between #{start_time} and #{end_time}
```
计算a里的指标结果，然后update到b的对应记录里面。
