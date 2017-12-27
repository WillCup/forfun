
title: django中使用视图
date: 2016-12-05 17:06:49
tags: [youdaonote]
---



最近在做数据地图之类的东西，要对hive的meta库进行一些查询与分析之类的。但是hive的字段表、tbls表、dbs表设计的有些鸡肋。中间隔了sds、cds表，所以查询非常费劲，就考虑搞一个视图，然后利用django的models直接进行orm的只读操作。


```sql
CREATE OR REPLACE VIEW col_tbl_db_view AS 
SELECT
    1 as id,
    db.DB_ID as db_id,
    db.`NAME` as db_name,
    a.TBL_ID as tbl_id,
    a.TBL_NAME as tbl_name,
    a.TBL_TYPE as tbl_type,
    d.TYPE_NAME as col_type_name,
    d.`COMMENT` as col_comment,
    d.COLUMN_NAME as col_name
FROM
    TBLS a
LEFT JOIN SDS b ON a.SD_ID = b.SD_ID
LEFT JOIN COLUMNS_V2 d ON b.CD_ID = d.CD_ID
LEFT JOIN DBS db ON a.DB_ID = db.DB_ID
```




model代码
```python
# !/usr/bin/env python
# -*- coding: utf-8 -*
'''
created by will 
'''
from django.db import models
import datetime
from django.utils import timezone
class DB(models.Model):
    db_id = models.BigIntegerField(max_length=20, primary_key=True)
    desc = models.CharField(max_length=4000)
    db_location_uri = models.CharField(max_length=4000)
    name = models.CharField(max_length=128)
    owner_name = models.CharField(max_length=128)
    owner_type = models.CharField(max_length=10)
    class Meta:
        managed = False
        db_table = 'DBS'

class TBL(models.Model):
    tbl_id = models.BigIntegerField(max_length=20, primary_key=True)
    create_time = models.DateTimeField
    owner = models.CharField(max_length=767)
    tbl_name = models.CharField(max_length=128)
    class Meta:
        managed = False
        db_table = 'TBLS'

class ColMeta(models.Model):
    '''
       字段对应meta
    '''
    id = models.IntegerField(primary_key=True)
    db = models.ForeignKey(DB, on_delete=models.DO_NOTHING)
    tbl = models.ForeignKey(TBL, on_delete=models.DO_NOTHING)
    tbl_type = models.CharField(max_length=30, null=True)
    col_type_name = models.CharField(max_length=300, null=True)
    col_comment = models.CharField(max_length=300, null=True)
    col_name = models.CharField(max_length=300, null=True)

    class Meta:
        managed = False
        db_table = 'col_tbl_db_view'
```
注意几点：
- 由于是只读的model，所以设置了managed=false，这样就不会在migration中对数据库进行任何操作了。
- 对于hive的meta库我们只做查询不做修改，也是出于上面的原因，我们把这些代码放在read_models.py里面，这个文件里放置的都是对既有数据库的只读操作的model。
- 使用view的时候注意几点
    - django的模板遍历结果的时候会使用到model的主键，当model里面没有指定某个字段是主键的时候，会自动生成一个id字段作为主键。这时，view里并没有id这个字段就会报错了。所以我们需要一个row_number()之类的东西，但是mysql不支持此种东西。所以尝试了一下把所有的id都弄成1，然后在model里面指定primary_key为id，结果成功。以此，推测，django template在遍历context里的变量的时候并不会判断主键是否唯一，也不是以主键为索引去抽取数据。
    - 在foreignkey的地方，一定要加上on_delete=models.DO_NOTHING。防止对表里的数据有任何的级联影响
    - 指定db_table为view的名字
    - 虽然明知会报错，但是不要调用save或者update之类的方法
    - 在运行makemigrations的时候，并不会包含创建视图的sql语句，因为managed=false。我们需要手动加到0001_initial.py里面去。
```python
...
migrations.RunSQL(
    """
    CREATE OR REPLACE VIEW col_tbl_db_view AS 
    SELECT
        1 as id,
        db.DB_ID as db_id,
        db.`NAME` as db_name,
        a.TBL_ID as tbl_id,
        a.TBL_NAME as tbl_name,
        a.TBL_TYPE as tbl_type,
        d.TYPE_NAME as col_type_name,
        d.`COMMENT` as col_comment,
        d.COLUMN_NAME as col_name
    FROM
        TBLS a
    LEFT JOIN SDS b ON a.SD_ID = b.SD_ID
    LEFT JOIN COLUMNS_V2 d ON b.CD_ID = d.CD_ID
    LEFT JOIN DBS db ON a.DB_ID = db.DB_ID
    """
),
...
```


参考： https://blog.rescale.com/using-database-views-in-django-orm/
