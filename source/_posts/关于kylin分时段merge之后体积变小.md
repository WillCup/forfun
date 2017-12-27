
title: 关于kylin分时段merge之后体积变小
date: 2017-06-26 19:01:24
tags: [youdaonote]
---

我们设置了一个cube，指定的mandatory dimension为create_year，问了一下，说是因为有按照年去查询指标的需求。

配置
---
mandatory dimension: create_year

自动merge : 7天、28天


现象
---

merge之前单天的数据大小平均为3G，而单周的数据大小平均为11G。

就是说： **merge完之后数据变小了**！

那么为什么呢？


几个人想了好半天，终于有个同事想到了....就是因为对于不包含merge时间【一般是天】的所有其他维度的组合，都减少了6倍的大小。




那么，由此延伸，我们是可以将create_day设置为mandatory dimension的，这样每天的cube就会减少很多大小，merge以后也不会有对应的压缩。这样对于单天记录的体积会有很大压缩提升。。。

那么对于年指标的查询应该怎样处理呢？由于这种比较少，建议可以前端解决。
