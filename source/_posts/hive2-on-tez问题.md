
title: hive2-on-tez问题
date: 2017-12-11 17:16:51
tags: [youdaonote]
---

hive cli端
```
hive> set hive.execution.engine;
hive.execution.engine=tez
hive> select count(1) as num, year(create_time) as time from money_record group by year(create_time);
Query ID = root_20171211171347_57cea87e-f54a-457d-93cf-b312063a822e
Total jobs = 1
Launching Job 1 out of 1
FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.tez.TezTask

```



yarn app的log信息
```
............
............
# container文件内容

7749233 1292 -r-xr-xr-x   1 yarn     hadoop    1319613 Mar  8  2017 ./tezlib/apache-tez-0.8.5-bin/tez-dag-0.8.5.jar
37749246    8 -r-xr-xr-x   1 yarn     hadoop       7734 Mar  8  2017 ./tezlib/apache-tez-0.8.5-bin/tez-yarn-timeline-history-with-acls-0.8.5.jar
37749242  144 -r-xr-xr-x   1 yarn     hadoop     146572 Mar  8  2017 ./tezlib/apache-tez-0.8.5-bin/tez-tests-0.8.5.jar
37749230    4 drwxr-xr-x   2 yarn     hadoop       4096 Dec 11 17:13 ./tezlib/apache-tez-0.8.5-bin/share
37749247 41916 -r-xr-xr-x   1 yarn     hadoop   42921455 Mar  8  2017 ./tezlib/apache-tez-0.8.5-bin/share/tez.tar.gz

......
# 启动脚本
LogType:launch_container.sh
Log Upload Time:Mon Dec 11 17:13:49 +0800 2017
LogLength:5084
Log Contents:
#!/bin/bash
....
....

# 指定类路径
export CLASSPATH="/usr/hdp/2.4.2.0-258/hadoop/lib/hadoop-lzo-0.6.0.2.4.2.0-258.jar:/etc/hadoop/conf/secure:$PWD:$PWD/*:$PWD/tezlib/*:$PWD/tezlib/lib/*:$HADOOP_CONF_DIR:"
........

LogType:stderr
Log Upload Time:Mon Dec 11 17:13:50 +0800 2017
LogLength:77
Log Contents:
Error: Could not find or load main class org.apache.tez.dag.app.DAGAppMaster
End of LogType:stderr

```


可以看到报错信息是找不到tez的主类，这个一般就是类路径的问题了。

找一个hive 1.2.1的tez的正常的程序，看一下container文件内容：
```
....
12322304  164 -r-xr-xr-x   1 yarn     hadoop     164267 Apr 25  2016 ./tezlib/tez-runtime-internals-0.7.0.2.4.2.0-258.jar
12322432   64 -r-xr-xr-x   1 yarn     hadoop      62943 Apr 25  2016 ./tezlib/tez-history-parser-0.7.0.2.4.2.0-258.jar
12322283   68 -r-xr-xr-x   1 yarn     hadoop      67195 Apr 25  2016 ./tezlib/tez-common-0.7.0.2.4.2.0-258.jar
12322430   28 -r-xr-xr-x   1 yarn     hadoop      25622 Apr 25  2016 ./tezlib/tez-yarn-timeline-history-0.7.0.2.4.2.0-258.jar
12322306  532 -r-xr-xr-x   1 yarn     hadoop     542434 Apr 25  2016 ./tezlib/tez-runtime-library-0.7.0.2.4.2.0-258.jar
....

LogType:launch_container.sh
Log Upload Time:Mon Dec 11 15:47:42 +0800 2017
LogLength:5740
Log Contents:
#!/bin/bash

....
export CLASSPATH="/usr/hdp/2.4.2.0-258/hadoop/lib/hadoop-lzo-0.6.0.2.4.2.0-258.jar:/etc/hadoop/conf/secure:$PWD:$PWD/*:$PWD/tezlib/*:$PWD/tezlib/lib/*:$HADOOP_CONF_DIR:"

....
```

可以看到都是用的默认的`$PWD:$PWD/*:$PWD/tezlib/*:$PWD/tezlib/lib/*`, 上面的执行错误的container里的tez文件多一个文件夹路径。

对于`tez.lib.uris`的注释里也说到：会解压开，然后直接作为classpath。


tez对于classpath[配置参考](https://tez.apache.org/releases/0.8.5/tez-api-javadocs/configs/TezConfiguration.html).


我们目前的配置是
```xml
  
    <property>
      <name>tez.lib.uris</name>
      <value>/hdp/apps/${hdp.version}/tez/apache-tez-0.8.5-bin.tar.gz</value>
    </property>

```


可以看出是这个包的问题，那么就有两个方式解决了。

- 把apache-tez-0.8.5-bin.tar.gz自己处理一下，把多余的文件夹去掉，重新上传
- 添加个classpath的前缀，也就是把文件夹名称加上。
```xml
<property>
  <name>tez.lib.uris.classpath</name>
  <value>....</value>
</property>
```


看一下tez的目录，发现share目录下有个tez.tar.gz, 这个文件就是专门做这个事情的。


