
title: hue中添加spark-shell的支持
date: 2017-11-07 16:56:49
tags: [youdaonote]
---

计划： livy + hue + spark

通过ambari已经部署完成了livy，并且通过了curl的spark测试。

参考： http://gethue.com/how-to-use-the-livy-spark-rest-job-server-for-interactive-spark-2-2/

#### 配置

```
# 修改spark配置

[spark]
  # Host address of the Livy Server.
  livy_server_host=servicenode02.will.com

  # Port of the Livy Server.
  livy_server_port=8998

  # Configure livy to start in local 'process' mode, or 'yarn' workers.
  ivy_server_session_kind=yarn

  # If livy should use proxy users when submitting a job.
  ## livy_impersonation_enabled=true

  # Host of the Sql Server
  ## sql_server_host=localhost

  # Port of the Sql Server
  ## sql_server_port=10000


[[[spark]]]
  name=Scala
  interface=livy

[[[pyspark]]]
  name=PySpark
  interface=livy

```


但是部署在hue中的时候，总是出现session问题, 在hue的example中下载了spark的测试notebook，执行pyspark的1+1+1时候出现提示

```
[07/Nov/2017 01:06:17 -0800] decorators   ERROR    error running <function execute at 0x7f0ca465ae60>
Traceback (most recent call last):
  File "/server/hue/desktop/libs/notebook/src/notebook/decorators.py", line 81, in decorator
    return func(*args, **kwargs)
  File "/server/hue/desktop/libs/notebook/src/notebook/api.py", line 109, in execute
    response['handle'] = get_api(request, snippet).execute(notebook, snippet)
  File "/server/hue/desktop/libs/notebook/src/notebook/connectors/spark_shell.py", line 194, in execute
    raise e
RestException: "Session '-1' not found." (error 404)
```


然后可以在livy server的rest api中看到session
```
{
    "from": 0,
    "total": 1,
    "sessions": [
        {
            "state": "idle",
            "proxyUser": null,
            "id": 0,
            "kind": "pyspark",
            "log": [
                "17/11/07 16:01:15 INFO SparkContext: Added file file:/usr/hdp/current/spark-client/python/lib/py4j-0.9-src.zip at file:/usr/hdp/current/spark-client/python/lib/py4j-0.9-src.zip with timestamp 1510041675612",
                "17/11/07 16:01:15 INFO Executor: Starting executor ID driver on host localhost",
                "17/11/07 16:01:15 INFO Utils: Successfully started service 'org.apache.spark.network.netty.NettyBlockTransferService' on port 42523.",
                "17/11/07 16:01:15 INFO NettyBlockTransferService: Server created on 42523",
                "17/11/07 16:01:15 INFO BlockManagerMaster: Trying to register BlockManager",
                "17/11/07 16:01:15 INFO BlockManagerMasterEndpoint: Registering block manager localhost:42523 with 511.1 MB RAM, BlockManagerId(driver, localhost, 42523)",
                "17/11/07 16:01:15 INFO BlockManagerMaster: Registered BlockManager",
                "17/11/07 16:01:16 WARN DomainSocketFactory: The short-circuit local reads feature cannot be used because libhadoop cannot be loaded.",
                "17/11/07 16:01:16 INFO EventLoggingListener: Logging events to hdfs:///spark-history/local-1510041675669",
                "17/11/07 16:01:18 INFO ScalatraBootstrap: Calling http://servicenode02.will.com:8998/sessions/0/callback..."
            ]
        }
    ]
}
```


点击recreate scala 的session后，发现scala代码是可以执行的。那么，我猜测，有可能pyspark也是一样的道理。

scala代码如下：
```scala
val data = Array(0,2,3,45,2);
val disData = sc.parallelize(data);
disData.map(s=>s+1).collect();
```

而且观察到此时livy的sessions接口里的session也有了变化
```
{
    "from": 0,
    "total": 1,
    "sessions": [
        {
            "state": "idle",
            "proxyUser": null,
            "id": 1,
            "kind": "spark",
            "log": [
                "17/11/07 17:43:10 INFO TaskSetManager: Finished task 0.0 in stage 0.0 (TID 0) in 2909 ms on localhost (5/8)",
                ............
                "17/11/07 17:43:21 INFO BlockManagerInfo: Removed broadcast_0_piece0 on localhost:36806 in memory (size: 1257.0 B, free: 511.1 MB)"
            ]
        }
    ]
}
```



到notebook的上面，点击pspark的recreate的地方，观察session变化：
```
{
    "from": 0,
    "total": 2,
    "sessions": [
        {
            "state": "idle",
            "proxyUser": null,
            "id": 2,
            "kind": "pyspark",
            "log": [
                "17/11/07 17:45:18 INFO SparkContext: Added file file:/usr/hdp/current/spark-client/python/lib/py4j-0.9-src.zip at file:/usr/hdp/current/spark-client/python/lib/py4j-0.9-src.zip with timestamp 1510047918034",
                .......
                "17/11/07 17:45:24 INFO ScalatraBootstrap: Calling http://servicenode02.will.com:8998/sessions/2/callback..."
            ]
        },
        {
            "state": "idle",
            "proxyUser": null,
            "id": 1,
            "kind": "spark",
            "log": [
                "17/11/07 17:43:10 INFO TaskSetManager: Finished task 0.0 in stage 0.0 (TID 0) in 2909 ms on localhost (5/8)",
                ......
                "17/11/07 17:43:21 INFO BlockManagerInfo: Removed broadcast_0_piece0 on localhost:36806 in memory (size: 1257.0 B, free: 511.1 MB)"
            ]
        }
    ]
}
```

多了一个。好，那么再次去执行我们的代码。

```python
file = sc.textFile("/apps/hive/warehouse/sample_08/sample_08")

file = file.flatMap(lambda line: line.split("\t")).map(lambda word: (word, 1)).reduceByKey(lambda a, b: a + b)

for row in file.collect()[:5]:
  print row
```

然后就可以了。哈哈


查看一下livy server的log
```
17/11/07 17:44:50 INFO SparkProcessBuilder: Running sh -c /usr/hdp/current/spark-client/bin/spark-submit
--name Livy 
--jars /usr/hdp/current/livy-server/repl-jars/mime-util-2.1.3.jar,/usr/hdp/current/livy-server/repl-jars/jetty-io-9.2.10.v20150310.jar,/usr/hdp/current/livy-server/repl-jars/jetty-security-9.2.10.v20150310.jar,/usr/hdp/current/livy-server/repl-jars/jetty-util-9.2.10.v20150310.jar,/usr/hdp/current/livy-server/repl-jars/livy-api-0.2.0.2.4.2.0-258.jar,/usr/hdp/current/livy-server/repl-jars/livy-core-0.2.0.2.4.2.0-258.jar,/usr/hdp/current/livy-server/repl-jars/rl_2.10-0.4.10.jar,/usr/hdp/current/livy-server/repl-jars/async-http-client-1.9.33.jar,/usr/hdp/current/livy-server/repl-jars/jetty-server-9.2.10.v20150310.jar,/usr/hdp/current/livy-server/repl-jars/commons-codec-1.9.jar,/usr/hdp/current/livy-server/repl-jars/joda-convert-1.6.jar,/usr/hdp/current/livy-server/repl-jars/scalatra-json_2.10-2.3.0.jar,/usr/hdp/current/livy-server/repl-jars/kryo-2.22.jar,/usr/hdp/current/livy-server/repl-jars/joda-time-2.3.jar,/usr/hdp/current/livy-server/repl-jars/grizzled-slf4j_2.10-1.0.2.jar,/usr/hdp/current/livy-server/repl-jars/scalatra-common_2.10-2.3.0.jar,/usr/hdp/current/livy-server/repl-jars/livy-client-common-0.2.0.2.4.2.0-258.jar,/usr/hdp/current/livy-server/repl-jars/livy-repl-0.2.0.2.4.2.0-258.jar,/usr/hdp/current/livy-server/repl-jars/jetty-http-9.2.10.v20150310.jar,/usr/hdp/current/livy-server/repl-jars/jetty-servlet-9.2.10.v20150310.jar,/usr/hdp/current/livy-server/repl-jars/scalatra_2.10-2.3.0.jar,/usr/hdp/current/livy-server/repl-jars/juniversalchardet-1.0.3.jar,/usr/hdp/current/livy-server/repl-jars/javax.servlet-api-3.1.0.jar,/usr/hdp/current/livy-server/repl-jars/netty-3.10.5.Final.jar 
--files /usr/hdp/current/spark-client/python/lib/pyspark.zip,/usr/hdp/current/spark-client/python/lib/py4j-0.9-src.zip 
--class com.cloudera.livy.repl.Main --conf spark.executor.memory=1G --conf spark.driver.memory=1G 
--conf spark.submit.pyFiles=/usr/hdp/current/spark-client/python/lib/pyspark.zip,/usr/hdp/current/spark-client/python/lib/py4j-0.9-src.zip 
--conf spark.driver.cores=1 
--conf spark.livy.callbackUrl=http://servicenode02.will.com:8998/sessions/2/callback 
--conf spark.livy.port=0 
--conf spark.yarn.isPython=true 
--conf spark.executor.cores=1 
--queue default 
spark-internal pyspark
17/11/07 17:44:50 INFO SessionManager: Registering new session 2

```

竟然没有提交到yarn上执行！！上面的参数我们明明配置了yarn啊。再查一下livy server的[相关配置信息](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.6.2/bk_command-line-installation/content/configure_livy2.html)。

到ambari添加配置
```
livy.spark.master=yarn-cluster
```

重启。并没有生效。


看下配置，竟然是这样的
```
livy.environment production
livy.impersonation.enabled false
livy.server.port 8998
livy.server.session.timeout 3600000
livy.spark.master yarn-cluster

```
中间的等号都没有了。

手动修改，暂时不用ambari了。
```
livy.environment =production
livy.impersonation.enabled= false
livy.server.port= 8998
livy.server.session.timeout =3600000
livy.spark.master= yarn-cluster

```

而且通过`ps -ef | grep livy`发现，ambari 2.2.2.0的livy对于stop并不完整，会有泄露问题存在。虽然livy server关掉了，但是它打开的spark进程却仍然存在。




