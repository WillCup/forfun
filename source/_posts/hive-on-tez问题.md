
title: hive-on-tez问题
date: 2017-12-07 18:48:39
tags: [youdaonote]
---

在ambari 2.2.2.0中安装的tez和hive。

命令行使用hive的时候出现问题，但是在hue，也就是hiveserver2中使用tez作为引擎就不报错：
```
Previous writer likely failed to write hdfs://dd/tmp/hive/root/_tez_session_dir/219ebced-00b7-4c38-88d1-61572a7d61c3/will_custom_hive_udf.jar
. Failing because I am unlikely to write too.
```
