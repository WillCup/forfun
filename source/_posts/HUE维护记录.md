
title: HUE维护记录
date: 2016-12-05 17:04:58
tags: [youdaonote]
---

1. 应用管理工具
tools/app_reg/app_reg.py

```
A tool to manage Hue applications. This does not stop/restart a
running Hue instance.

Usage:
    ./tools/app_reg/app_reg.py [flags] --install <path_to_app> [<path_to_app> ...] [--relative-paths]
        To register and install new application(s).
        Add '--relative-paths' to the end of the args list to force the app manager to register the new application using its path relative to the hue root.

    ./tools/app_reg/app_reg.py [flags] --remove <application_name>
        To unregister and remove an installed application.

    ./tools/app_reg/app_reg.py [flags] --list
        To list all registered applications.

    ./tools/app_reg/app_reg.py [flags] --sync
        Synchronize all registered applications with the Hue environment.
        Useful after a `make clean'.

Optional flags:
    --debug             Turns on debugging output
```


脚本
---

clean.sh
定时关闭hue的session和查询
```
export HIVE_CONF_DIR=/usr/hdp/2.4.2.0-258/hive/conf/
cd /server/hue/hue
build/env/bin/hue close_queries 3
build/env/bin/hue close_sessions 1
```
restart_hue.sh
定时重启hue
```
num=`ps -ef | grep hue | wc -l`
if [ $num -lt 2 ];then
	echo "restart hue at `date +%Y%m%d%H%M`"
	/server/hue/hue/build/env/bin/supervisor &
fi
```

指定yarn队列
---
beeline可以这样
```
jdbc:hive2://10.0.1.84:10000?mapreduce.job.queuename=root.default
```
但是hue没有提供额外设置hiveserver连接参数的地方。看来只能重新指定hive配置目录了。
