
title: runningdata改版记录
date: 2017-05-18 10:54:54
tags: [youdaonote]
---

抽象ETLObjRelated，其他对象继承之


- ETL的字段名tbl_name需要在数据库里修改为name
    - tasks里的exec_etl_sche任务会调用etl.name【老表的tbl_name会报错找不到】
- 执行metamap 0045的migrations，这个只是添加字段，不会造成额外影响
- 




sdf 

1. 确保新的save方法都已经注释掉

2. ./manage.py migrate metamap 0050 --settings=metamap.config

3. 开始清洗
- clean_etl
- clean_rel
- clean_m2h
- before_clean_blood
- clean_blood
- clean_h2m
- clean_jar
- clean_email
- clean_task[分别执行clean， type=4, type=1等等]
- clean_period

注意
---
1. H2M中的个别现象，多个周期的数据存放在同一个表中：
SELECT * from metamap_sqoophive2mysql where id in (71,72,73,74,75,76)
需要额外手动维护execblood
2. 放开那些注释
