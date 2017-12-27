
title: hive-llap启动
date: 2017-12-07 17:22:12
tags: [youdaonote]
---

#### 前提

- tez
- slider


#### 配置

##### conf/hive-env.sh

```
# 到tez的配置目录里找tez-site.xml
export TEZ_HOME=/usr/hdp/current/tez-client
export TEZ_CONF_DIR=$TEZ_HOME/conf

HADOOP_HOME=/usr/hdp/current/hadoop-client/
```


##### hive-site.xml
```xml
 <property>
    <name>hive.execution.mode</name>
    <value>llap</value>
    <description>
      Expects one of [container, llap].
      Chooses whether query fragments will run in container or in llap
    </description>
  </property>

```


##### tez-site.xml
```xml
   <property>
      <name>tez.lib.uris</name>
      <value>/hdp/apps/${hdp.version}/tez/apache-tez-0.8.5-bin.tar.gz</value>
    </property>
```


#### slider查看

```
[root@servicenode04 apache-hive-2.3.2-bin]# slider list
2017-12-07 18:10:01,421 [main] INFO  impl.TimelineClientImpl - Timeline service address: http://datanode04.will.com:8188/ws/v1/timeline/
2017-12-07 18:10:02,387 [main] WARN  shortcircuit.DomainSocketFactory - The short-circuit local reads feature cannot be used because libhadoop cannot be loaded.
2017-12-07 18:10:02,403 [main] INFO  client.RMProxy - Connecting to ResourceManager at datanode02.will.com/10.2.19.83:8050
llap_service                       RUNNING  application_1511064572746_27005             http://datanode02.will.com:8088/proxy/application_1511064572746_27005/

```


#### 报错

##### container大小

```
[root@servicenode04 apache-hive-2.3.2-bin]# ./bin/hive --service llap --instances 5 
llap
INFO cli.LlapServiceDriver: LLAP service driver invoked with arguments=-directory
INFO conf.HiveConf: Found configuration file file:/server/apache-hive-2.3.2-bin/conf/hive-site.xml
Failed: Container size (-1B) should be greater than minimum allocation(2.00GB)
java.lang.IllegalArgumentException: Container size (-1B) should be greater than minimum allocation(2.00GB)
	at com.google.common.base.Preconditions.checkArgument(Preconditions.java:92)
	at org.apache.hadoop.hive.llap.cli.LlapServiceDriver.run(LlapServiceDriver.java:313)
	at org.apache.hadoop.hive.llap.cli.LlapServiceDriver.main(LlapServiceDriver.java:116)
INFO cli.LlapServiceDriver: LLAP service driver finished

```

提示是container大小还没有最小的大。加个参数就行了
```
./bin/hive --service llap --size 2147483648 --instances 3
```

##### 配置漏项

生成slider项目目录之后，启动llap失败。到yarn的日志里翻看，发现异常：
```
 yarn logs --applicationId application_1511064572746_25101 |less
```

```
017-12-06T11:56:39,771 WARN  [main ()] org.apache.hadoop.hive.llap.daemon.impl.LlapDaemon: Failed to start LLAP Daemon with exception
java.lang.NullPointerException
        at org.apache.hadoop.hive.llap.daemon.impl.LlapDaemon.<init>(LlapDaemon.java:139) ~[hive-llap-server-2.3.2.jar:2.3.2]
        at org.apache.hadoop.hive.llap.daemon.impl.LlapDaemon.main(LlapDaemon.java:521) [hive-llap-server-2.3.2.jar:2.3.2]
End of LogType:llap-daemon-root-datanode02.will.com.log

```

找到对应版本的源码里，发现是少了配置项`String hosts = HiveConf.getTrimmedVar(daemonConf, ConfVars.LLAP_DAEMON_SERVICE_HOSTS);`,也就是`hive.llap.daemon.service.hosts`，应该配置为@llap_service.

##### tez版本问题

重新生产slider项目，再次，运行，还是异常退出了，继续看yarn的log。
```
ue
2017-12-07T17:12:20,582 WARN  [main ()] org.apache.hadoop.hive.llap.daemon.impl.LlapDaemon: Failed to start LLAP Daemon with exception
java.lang.NoClassDefFoundError: org/apache/tez/hadoop/shim/HadoopShimsLoader
        at org.apache.hadoop.hive.llap.daemon.impl.ContainerRunnerImpl.<init>(ContainerRunnerImpl.java:157) ~[hive-llap-server-2.3.2.jar:2.3.2]
        at org.apache.hadoop.hive.llap.daemon.impl.LlapDaemon.<init>(LlapDaemon.java:283) ~[hive-llap-server-2.3.2.jar:2.3.2]
        at org.apache.hadoop.hive.llap.daemon.impl.LlapDaemon.main(LlapDaemon.java:521) [hive-llap-server-2.3.2.jar:2.3.2]
Caused by: java.lang.ClassNotFoundException: org.apache.tez.hadoop.shim.HadoopShimsLoader
        at java.net.URLClassLoader.findClass(URLClassLoader.java:381) ~[?:1.8.0_60]
        at java.lang.ClassLoader.loadClass(ClassLoader.java:424) ~[?:1.8.0_60]
        at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:331) ~[?:1.8.0_60]
        at java.lang.ClassLoader.loadClass(ClassLoader.java:357) ~[?:1.8.0_60]
        ... 3 more
End of LogType:llap-daemon-root-datanode05.will.com.log

```

查了一下，这个类是tez的。我当前使用的是0.7版本的，到github上试了一下，这个版本还没有这个类。那就自己下载个有这个类的tez，上传到hdfs，重新修改tez-site.xml，指定下位置。

##### log4j版本问题？

```
Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/commons/logging/LogFactory
        at org.apache.hadoop.service.AbstractService.<clinit>(AbstractService.java:43)
Caused by: java.lang.ClassNotFoundException: org.apache.commons.logging.LogFactory
        at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
        at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:331)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
        ... 1 more
End of LogType:llap-daemon-root-datanode11.will.com.out

```

这次是hadoop的AbstractService初始化的时候，找不到commons-logging的类了...好累

看这行日志上面输出了类路径之类的东西：
```
+ exec /server/java/jdk1.8.0_60/bin/java -Dproc_llapdaemon -Xms4096m -Xmx4096m -XX:+UseG1GC -XX:+ResizeTLAB -XX:+UseNUMA -XX:-ResizePLAB -server -Djava.net.p
referIPv4Stack=true -XX:NewRatio=8 -XX:+UseNUMA -XX:+PrintGCDetails -verbose:gc -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=4 -XX:GCLogFileSize=100M -XX
:+PrintGCDateStamps -Xloggc:/server/hadoop/yarn/log/application_1511064572746_27003/container_e51_1511064572746_27003_01_000006//gc.log -Djava.io.tmpdir=/ser
ver/hadoop/yarn/local/usercache/root/appcache/application_1511064572746_27003/container_e51_1511064572746_27003_01_000006/tmp/ -Dlog4j.configurationFile=llap
-daemon-log4j2.properties -Dllap.daemon.log.dir=/server/hadoop/yarn/log/application_1511064572746_27003/container_e51_1511064572746_27003_01_000006/ -Dllap.d
aemon.log.file=llap-daemon-root-datanode02.will.com.log -Dllap.daemon.root.logger=query-routing -Dllap.daemon.log.level=INFO -classpath '/server/hadoop/yar
n/local/usercache/root/appcache/application_1511064572746_27003/container_e51_1511064572746_27003_01_000006/app/install//conf/:/server/hadoop/yarn/local/user
cache/root/appcache/application_1511064572746_27003/container_e51_1511064572746_27003_01_000006/app/install//lib/*:/server/hadoop/yarn/local/usercache/root/a
ppcache/application_1511064572746_27003/container_e51_1511064572746_27003_01_000006/app/install//lib/tez/*:/server/hadoop/yarn/local/usercache/root/appcache/
application_1511064572746_27003/container_e51_1511064572746_27003_01_000006/app/install//lib/udfs/*:.:'
```
明显发现这个classpath是不包含hadoop相关的类的。一番查找，找到hive根目录下的`scripts/llap/bin/runLlapDaemon.sh`文件，添加一下：
```
CLASSPATH=${LLAP_DAEMON_CONF_DIR}:${LLAP_DAEMON_HOME}/lib/*:${LLAP_DAEMON_HOME}/lib/tez/*:${LLAP_DAEMON_HOME}/lib/udfs/*:.`hadoop classpath`
```

再重新生成slider项目。




参考：

http://housong.github.io/2017/hive-llap/

http://blog.csdn.net/qingzhenli/article/details/72723018
