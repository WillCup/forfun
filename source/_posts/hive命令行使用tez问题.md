
title: hive命令行使用tez问题
date: 2017-12-11 15:48:25
tags: [youdaonote]
---

#### 背景

- 使用ambari2.2.2.0搭建的集群环境
- 使用了自定义的hive UDF包


#### 问题
通过hue在hiveserver2上使用tez进行查询，正常运行。


使用hive cli进行tez查询的时候，报错：
```
2017-12-11 15:06:30,745 INFO  [main]: ql.Context (Context.java:getMRScratchDir(330)) - New scratch dir is hdfs://dd/tmp/hive/root/c6354673-afc9-4515-8c44-ff7c7f75edcb/hive_2017-12-11_15-06-30_517_4074051706046309823-1
2017-12-11 15:06:30,750 INFO  [main]: exec.Task (TezTask.java:updateSession(270)) - Tez session hasn't been created yet. Opening session
2017-12-11 15:06:30,750 INFO  [main]: tez.TezSessionState (TezSessionState.java:open(130)) - Opening the session with id 97240ae8-cce2-4000-83d6-f6729146ebab for thread main log trace id -  query id - root_20171211150630_4fbc2d95-5613-438c-9499-126da1ce1f49
2017-12-11 15:06:30,750 INFO  [main]: tez.TezSessionState (TezSessionState.java:open(146)) - User of session id 97240ae8-cce2-4000-83d6-f6729146ebab is root
2017-12-11 15:06:30,763 INFO  [main]: tez.DagUtils (DagUtils.java:createLocalResource(720)) - Resource modification time: 1512974530270
2017-12-11 15:06:30,766 INFO  [main]: tez.DagUtils (DagUtils.java:createLocalResource(720)) - Resource modification time: 1512974530408
2017-12-11 15:06:30,770 INFO  [main]: tez.DagUtils (DagUtils.java:createLocalResource(720)) - Resource modification time: 1512974530453
2017-12-11 15:06:30,773 INFO  [main]: tez.DagUtils (DagUtils.java:createLocalResource(720)) - Resource modification time: 1512974530487
2017-12-11 15:06:30,775 INFO  [main]: tez.DagUtils (DagUtils.java:createLocalResource(720)) - Resource modification time: 1512974530408
2017-12-11 15:06:30,777 INFO  [main]: tez.DagUtils (DagUtils.java:localizeResource(954)) - Localizing resource because it does not exist: file:/server/app/hive/will_hive_udf.jar to dest: hdfs://dd/tmp/hive/root/_tez_session_dir/97240ae8-cce2-4000-83d6-f6729146ebab/will_hive_udf.jar
2017-12-11 15:06:30,780 INFO  [main]: tez.DagUtils (DagUtils.java:localizeResource(958)) - Looks like another thread is writing the same file will wait.
2017-12-11 15:06:30,780 INFO  [main]: tez.DagUtils (DagUtils.java:localizeResource(965)) - Number of wait attempts: 5. Wait interval: 5000
2017-12-11 15:06:55,790 ERROR [main]: tez.DagUtils (DagUtils.java:localizeResource(981)) - Could not find the jar that was being uploaded
2017-12-11 15:06:55,790 ERROR [main]: exec.Task (TezTask.java:execute(212)) - Failed to execute tez graph.
java.io.IOException: Previous writer likely failed to write hdfs://dd/tmp/hive/root/_tez_session_dir/97240ae8-cce2-4000-83d6-f6729146ebab/will_hive_udf.jar. Failing because I am unlikely to write too.
        at org.apache.hadoop.hive.ql.exec.tez.DagUtils.localizeResource(DagUtils.java:982)
        at org.apache.hadoop.hive.ql.exec.tez.DagUtils.addTempResources(DagUtils.java:862)

```

#### 问题定位
上传udf到hdfs中tez session目录的时候，出现错误。

#### 排查过程

一遍遍地检查HDFS上的目录，发现尽管报错，也已经有了这个UDF jar文件了。
```
[root@servicenode02 livy]# hdfs dfs -ls /tmp/hive/root/_tez_session_dir/97240ae8-cce2-4000-83d6-f6729146ebab
Found 4 items
-rw-r--r--   3 root hdfs     258861 2017-12-11 15:09 /tmp/hive/root/_tez_session_dir/97240ae8-cce2-4000-83d6-f6729146ebab/hive-hcatalog-core.jar
-rw-r--r--   3 root hdfs       3809 2017-12-11 15:09 /tmp/hive/root/_tez_session_dir/97240ae8-cce2-4000-83d6-f6729146ebab/tinyv_hive_udf.jar
-rw-r--r--   3 root hdfs      83977 2017-12-11 15:09 /tmp/hive/root/_tez_session_dir/97240ae8-cce2-4000-83d6-f6729146ebab/tinyv_json_serde_1.3.8.jar
-rw-r--r--   3 root hdfs      24630 2017-12-11 15:09 /tmp/hive/root/_tez_session_dir/97240ae8-cce2-4000-83d6-f6729146ebab/will_hive_udf.jar

```

来来回回看了八百遍，到源码里也看了一下
```java
try {
    // 尝试将本地文件copy到远程的目录中
    destFS.copyFromLocalFile(false, false, src, dest);
} catch (IOException e) {
  // 如果copy失败，一般就是文件已经存在了，或者有其他人在进行同样的操作
  LOG.info("Looks like another thread is writing the same file will wait.");
  int waitAttempts =
      conf.getInt(HiveConf.ConfVars.HIVE_LOCALIZE_RESOURCE_NUM_WAIT_ATTEMPTS.varname,
          HiveConf.ConfVars.HIVE_LOCALIZE_RESOURCE_NUM_WAIT_ATTEMPTS.defaultIntVal);
  long sleepInterval = HiveConf.getTimeVar(
      conf, HiveConf.ConfVars.HIVE_LOCALIZE_RESOURCE_WAIT_INTERVAL,
      TimeUnit.MILLISECONDS);
  LOG.info("Number of wait attempts: " + waitAttempts + ". Wait interval: "
      + sleepInterval);
      
  // 隔一段时间检查一下，这个文件是不是已经有了
  boolean found = false;
  for (int i = 0; i < waitAttempts; i++) {
    if (!checkPreExisting(src, dest, conf)) {
      try {
        Thread.sleep(sleepInterval);
      } catch (InterruptedException interruptedException) {
        throw new IOException(interruptedException);
      }
    } else {
      found = true;
      break;
    }
  }
  // 如果一直都么有，就报上面的错误了
  if (!found) {
    LOG.error("Could not find the jar that was being uploaded");
    throw new IOException("Previous writer likely failed to write " + dest +
        ". Failing because I am unlikely to write too.");
  }
}

```

对于`checkPreExisting`方法，也要看一下：
```java
// 检查目标目录下是不是已经有了同名文件，
// 如果文件已经有了，那么要检查文件是否一致，这里简单用len验证
private boolean checkPreExisting(Path src, Path dest, Configuration conf)
    throws IOException {
    FileSystem destFS = dest.getFileSystem(conf);
    FileSystem sourceFS = src.getFileSystem(conf);
    FileStatus destStatus = FileUtils.getFileStatusOrNull(destFS, dest);
    if (destStatus != null) {
      // 我挂到了这一步，就是udf的版本不一致
      return (sourceFS.getFileStatus(src).getLen() == destStatus.getLen());
    }
    return false;
}

```


好了。那么看一下我的问题。为了使用udf，我们的数据研发在hive cli的初始化文件`bin/.hiverc`中添加了下面的命令：
```
add jar /server/app/hive/will_hive_udf.jar;
add jar /usr/hdp/current/hive-client/auxlib/tinyv_hive_udf.jar;
create temporary function  strstart as 'com.will.common.hive.StrStart';  
create temporary function  strDateFormat as 'com.will.common.hive.StrDateFormat';
create temporary function  strContain as 'com.will.common.hive.StrContain'; 
create temporary function  parse_uri as 'com.will.common.hive.ParseUri';
create temporary function  strEnd as 'com.will.common.hive.StrEnd';
create temporary function  split_by_index as 'com.will.common.hive.SplitByIndex';
create temporary function birthday as 'com.will.common.hive.BirthdayByIdcard';
create temporary function gender as 'com.will.common.hive.GenderByIdcard';  
create temporary function usernamesen as 'com.will.common.hive.UserNameSen';
create temporary function strTrim as 'com.will.common.hive.StrTrim';
create temporary function channel as 'com.will.common.hive.Channel';
create temporary function iptonum as 'com.will.common.hive.IpToNumber';
create temporary function geturl as 'com.will.common.hive.ParseUrls';
create temporary function money_record_type as 'com.will.common.hive.MoneyRecordType';
create temporary function xv_encode as 'com.will.tinyv.Encode';
create temporary function xv_decode as 'com.will.tinyv.Decode';
set hive.mapred.mode=nostrict;

```


重新启动hive cli，报错的日志，发现上传了来自两个不同地方的will_hive_udf.jar文件：一个来自`hive/auxlib`目录，另一个来自`/server/app`.....而**只有第一次报错的时候才能看到第一个udf的上传过程，因为同一个session中，已经上传过的，验证没问题就跳过了。。。。只打印了一个时间戳，也没有指明是哪个文件**.....导致定位问题的时候，有些恍惚，还以为是tez与UDF功能之间的问题
```
2017-12-11 15:09:30,604 INFO  [main]: ql.Context (Context.java:getMRScratchDir(330)) - New scratch dir is hdfs://dd/tmp/hive/root/c6354673-afc9-4515-8c44-ff7
c7f75edcb/hive_2017-12-11_15-09-30_406_4902171368286077085-1
2017-12-11 15:09:30,608 INFO  [main]: exec.Task (TezTask.java:updateSession(270)) - Tez session hasn't been created yet. Opening session
2017-12-11 15:09:30,608 INFO  [main]: tez.TezSessionState (TezSessionState.java:open(130)) - Opening the session with id 97240ae8-cce2-4000-83d6-f6729146ebab
 for thread main log trace id -  query id - root_20171211150930_d928c00e-62da-4c0e-95c2-23a28594a7c0
2017-12-11 15:09:30,608 INFO  [main]: tez.TezSessionState (TezSessionState.java:open(146)) - User of session id 97240ae8-cce2-4000-83d6-f6729146ebab is root
2017-12-11 15:09:30,613 INFO  [main]: tez.DagUtils (DagUtils.java:localizeResource(954)) - Localizing resource because it does not exist: file:/usr/hdp/curre
nt/hive-webhcat/share/hcatalog/hive-hcatalog-core.jar to dest: hdfs://dd/tmp/hive/root/_tez_session_dir/97240ae8-cce2-4000-83d6-f6729146ebab/hive-hcatalog-co
re.jar
2017-12-11 15:09:30,655 INFO  [main]: tez.DagUtils (DagUtils.java:createLocalResource(720)) - Resource modification time: 1512976170667
2017-12-11 15:09:30,656 INFO  [main]: tez.DagUtils (DagUtils.java:localizeResource(954)) - Localizing resource because it does not exist: file:/usr/hdp/2.4.2
.0-258/hive/auxlib/tinyv_hive_udf.jar to dest: hdfs://dd/tmp/hive/root/_tez_session_dir/97240ae8-cce2-4000-83d6-f6729146ebab/tinyv_hive_udf.jar
2017-12-11 15:09:30,683 INFO  [main]: tez.DagUtils (DagUtils.java:createLocalResource(720)) - Resource modification time: 1512976170691
2017-12-11 15:09:30,684 INFO  [main]: tez.DagUtils (DagUtils.java:localizeResource(954)) - Localizing resource because it does not exist: file:/usr/hdp/2.4.2
.0-258/hive/auxlib/tinyv_json_serde_1.3.8.jar to dest: hdfs://dd/tmp/hive/root/_tez_session_dir/97240ae8-cce2-4000-83d6-f6729146ebab/tinyv_json_serde_1.3.8.j
ar
2017-12-11 15:09:30,707 INFO  [main]: tez.DagUtils (DagUtils.java:createLocalResource(720)) - Resource modification time: 1512976170720
2017-12-11 15:09:30,709 INFO  [main]: tez.DagUtils (DagUtils.java:localizeResource(954)) - Localizing resource because it does not exist: file:/usr/hdp/2.4.2.0-258/hive/auxlib/will_hive_udf.jar to dest: hdfs://dd/tmp/hive/root/_tez_session_dir/97240ae8-cce2-4000-83d6-f6729146ebab/will_hive_udf.jar
2017-12-11 15:09:30,729 INFO  [main]: tez.DagUtils (DagUtils.java:createLocalResource(720)) - Resource modification time: 1512976170742
2017-12-11 15:09:30,732 INFO  [main]: tez.DagUtils (DagUtils.java:createLocalResource(720)) - Resource modification time: 1512976170691
2017-12-11 15:09:30,733 INFO  [main]: tez.DagUtils (DagUtils.java:localizeResource(954)) - Localizing resource because it does not exist: file:/server/app/hive/will_hive_udf.jar to dest: hdfs://dd/tmp/hive/root/_tez_session_dir/97240ae8-cce2-4000-83d6-f6729146ebab/will_hive_udf.jar
2017-12-11 15:09:30,735 INFO  [main]: tez.DagUtils (DagUtils.java:localizeResource(958)) - Looks like another thread is writing the same file will wait.
2017-12-11 15:09:30,736 INFO  [main]: tez.DagUtils (DagUtils.java:localizeResource(965)) - Number of wait attempts: 5. Wait interval: 5000
2017-12-11 15:09:55,745 ERROR [main]: tez.DagUtils (DagUtils.java:localizeResource(981)) - Could not find the jar that was being uploaded
2017-12-11 15:09:55,745 ERROR [main]: exec.Task (TezTask.java:execute(212)) - Failed to execute tez graph.
java.io.IOException: Previous writer likely failed to write hdfs://dd/tmp/hive/root/_tez_session_dir/97240ae8-cce2-4000-83d6-f6729146ebab/will_hive_udf.jar. Failing because I am unlikely to write too.
        at org.apache.hadoop.hive.ql.exec.tez.DagUtils.localizeResource(DagUtils.java:982)
        at org.apache.hadoop.hive.ql.exec.tez.DagUtils.addTempResources(DagUtils.java:862)
        at org.apache.hadoop.hive.ql.exec.tez.DagUtils.localizeTempFilesFromConf(DagUtils.java:805)
        at org.apache.hadoop.hive.ql.exec.tez.TezSessionState.refreshLocalResourcesFromConf(TezSessionState.java:233)
        at org.apache.hadoop.hive.ql.exec.tez.TezSessionState.open(TezSessionState.java:158)
        at org.apache.hadoop.hive.ql.exec.tez.TezTask.updateSession(TezTask.java:271)
        at org.apache.hadoop.hive.ql.exec.tez.TezTask.execute(TezTask.java:151)
        at org.apache.hadoop.hive.ql.exec.Task.executeTask(Task.java:160)
        at org.apache.hadoop.hive.ql.exec.TaskRunner.runSequential(TaskRunner.java:89)

```


#### 总结原因

- udf版本不一致，导致后面jar包上传失败

#### 解决

把.hiverc中的命令暂时去掉，重试tez引擎。就能成功调用了。


#### 其他思考

为什么已经设定了`hive.aux.jars.path`了，还要在.hiverc额外处理呢。

先在hivecli中确认一下：
```
[root@schedule ~]#hive -e "set hive.aux.jars.path"
WARNING: Use "yarn jar" to launch YARN applications.

Logging initialized using configuration in file:/etc/hive/2.4.2.0-258/0/hive-log4j.properties
Putting the global hiverc in $HIVE_HOME/bin/.hiverc is deprecated. Please use $HIVE_CONF_DIR/.hiverc instead.
hive.aux.jars.path=file:///usr/hdp/current/hive-webhcat/share/hcatalog/hive-hcatalog-core.jar,file:///usr/hdp/2.4.2.0-258/hive/auxlib/tinyv_hive_udf.jar,file:///usr/hdp/2.4.2.0-258/hive/auxlib/tinyv_json_serde_1.3.8.jar,file:///usr/hdp/2.4.2.0-258/hive/auxlib/will_hive_udf.jar

```


但是测试了一下，没有.hiverc里的初始化，就不能执行udf函数。
```
hive> select iptonum('10.2.19.32');
FAILED: SemanticException [Error 10011]: Line 1:7 Invalid function 'iptonum'

```


可是hue里，也就是hiveserver2里却可以执行udf函数。



