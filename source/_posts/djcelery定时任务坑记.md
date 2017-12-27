
title: djcelery定时任务坑记
date: 2016-12-05 17:05:44
tags: [youdaonote]
---

celery实现定时任务，原生是通过后台的配置文件settings里配置添加的。形如：
```py
CELERYBEAT_SCHEDULE = {
    'add-every-30-seconds': {
        'task': 'tasks.add',
        'schedule': timedelta(seconds=30),
        'args': (16, 16)
    },
    'add-every-30-seconds': {
        'task': 'tasks.add',
        'schedule': timedelta(seconds=30),
        'args': (16, 16)
    },
}

```

每增加或修改或删除都要重新修改这个文件，并且重启beat才能生效。不友好

djcelery结合mysql，将定时任务的配置放进mysql，然后自己实现了一个beat服务，每次定时扫描mysql里最新的定时任务信息，来调度任务，默认扫描间隔是5s。

挺好，它还提供了一个admin界面来添加定时任务。

我们需要的是一个自由增删改查任务的功能，暴露给编写任务并设定任务执行周期的数据分析人员。所以需要在djcelery的基础之上再稍微加一些自定义的实现：为了与现有任务融合，在periodTask模型上添加了一个我们自定义任务WillTask的外键。

中间遇到问题：
```
定义一个定时任务periodTask1，然后修改只修改cron到10分钟以后，结果到了时间竟然没有调度。

到admin界面同样操作了一下，能够触发最新配置的调度。

```

观察beat的日志，发现它总是查询djcelery_periodictasks里的内容：
```

```

然后就注意观察了一下，发现在admin里操作更改任务的cron的时候，djcelery_periodictasks里的last_update字段会更新。而我自己修改的话就不会更新。

修改逻辑，每次对task进行修改都去更新djcelery_periodictasks里的last_update字段为当前时间。 —— 成功。


找网友@周星星对了一下，他是使用的djcelery的master 3.2.0a分支，不存在这个问题。

他的代码：
```py
target_task = PeriodicTask.objects.get(name='test_1338')
crontab = CrontabSchedule.objects.create(minute='*/2')
crontab.save()
target_task.crontab = crontab
target_task.args = '[1, 888888888, "1??99999???"]'
result = target_task.save()
return HttpResponse(result, content_type='text/json')
```
可以看到他是每次都新建的，中间我也尝试了新建cron，但是结果还是同上。鉴于程序严密性，又改回了修改cron。

我当前的代码：
```py
task = WillDependencyTask.objects.get(pk=pk)
httputils.post2obj(task, request.POST, 'id')
task.save()

cron_task = PeriodicTask.objects.get(willtask_id=pk)
cron_task.name = task.name
cron_task.save()

cron = DjceleryCrontabschedule.objects.get(pk=cron_task.crontab_id)
cron.minute, cron.hour, cron.day_of_month, cron.month_of_year, cron.day_of_week = cronhelper.cron_from_str(
    request.POST['cronexp'])
cron.save()

# 这里是多出来的地方
tasks = DjceleryPeriodictasks.objects.get(ident=1)
tasks.last_update = timezone.now()
tasks.save()

```
所以初步确认为版本问题。我使用的是djcelery3.1.17。



参考：
- http://docs.celeryproject.org/en/latest/userguide/periodic-tasks.html
- https://github.com/celery/django-celery
- 


传送长数据串出错
----

```python

@transaction.atomic
def exec_job(request, sqoopid):
    sqoop = SqoopHive2Mysql.objects.get(id=sqoopid)
    location = AZKABAN_SCRIPT_LOCATION + dateutils.now_datetime() + '-sqoop-' + sqoop.name + '.log'
    command = etlhelper.generate_sqoop_hive2mysql(sqoop)
    execution = SqoopHive2MysqlExecutions(logLocation=location, job_id=sqoopid, status=0)
    execution.save()
    from metamap import tasks
    tasks.exec_sqoop.delay(command, location)
    return redirect('metamap:sqoop_execlog', execid=execution.id)
```

command再这里打印是：
```
sqoop export  --connect jdbc:mysql://120.55.176.18:5306/product?useCursorFetch=true&dontTrackOpenResources=true&defaultFetchSize=2000 --driver com.mysql.jdbc.Driver --username xmanread --password LtLUGkNbr84UWXglBFYe4GuMX8EJXeIG  --input-fields-terminated-by "\t"    --update-key  create_date,period,type,channel_id,bank_name,platform,status_id,return_status,channel_id,bank_name,platform,status_id,return_status,number_order,amount_order  --update-mode  allowinsert  --columns  create_date,end_date,period,type,channel_id,bank_name,platform,status_id,return_status,number_order,amount_order  --export-dir  hdfs://namenode01.will.com:8020/apps/hive/warehouse/dim_payment.db/order_detail/log_type=1/log_period=1/log_create_date=2016-12-06  --table  JLC_ORDER_DETAIL_APP  --verbose
```

在task执行的时候打印出来是：
```
sqoop export  --connect jdbc:mysql://120.55.176.18:5306/product?useCursorFetch=true&dontTrackOpenResources=true&defaultFetchSize=2000 --driver com.my xmanread --password LtLUGkNbr84UWXglBFYe4GuMX8EJXeIG  --input-fields-terminated-by "\t"    --update-key  create_date,period,type,channel_id,bank_name,platform,status_id,return_status,channel_id,bank_name,platform,status_id,return_status,number_order,amount_order  --update-mode  allowinsert  --columns  create_date,end_date,period,type,channel_id,bank_name,platform,status_id,return_status,number_order,amount_order  --export-dir  hdfs://namenode01.will.com:8020/apps/hive/warehouse/dim_payment.db/order_detail/log_type=1/log_period=1/log_create_date=2016-12-06  --table  JLC_ORDER_DETAIL_APP  --verbose , location is /var/azkaban-metamap/20161208062420-sqoop-export_JLC_ORDER_DETAIL_APP.log

```

driver那里莫名就消失了。

后来发现是由于windows里的换行符导致的。


celery beat不触发定时任务
---
**注意时区，妈蛋！！**


