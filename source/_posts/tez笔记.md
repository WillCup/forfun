
title: tez笔记
date: 2017-11-27 18:47:27
tags: [youdaonote]
---

1. 部署hadoop2.7以上版本
2. 构建tez`mvn clean package -DskipTests=true -Dmaven.javadoc.skip=true`
 - JDK8, MAVEN 3
 - protocol buffer 2.5.0
 - 如果使用单元测试，把skipTests去掉就行
 - 如果使用eclipse，可以使用import maven project引入。
3. 把对应的tez包copy到HDFS，配置tez-site.xml
- tez包包含tez和hadoop的类库，tez-dist/tez-x.y.z-SNAPSHOT.tar.gz
- 假设tez放在了HDFS的/apps下，命令如下
```
hadoop fs -mkdir /apps/tez-x.y.z-SNAPSHOT
hadoop fs -copyFromLocal tez-dist/target/tez-x.y.z-SNAPSHOT.tar.gz /apps/tez-x.y.z-SNAPSHOT
```
- tez-site.xml配置
    - 设置`tez.lib.uris`指定HDFS上的tar.gz的位置。假设是上面的话，就设置成`${fs.defaultFS}/apps/tez-x.y.z-SNAPSHOT/tez-x.y.z-SNAPSHOT.tar.gz`
    - 确认`tez.cluster.hadoop-libs`没有在tez-site.xml中设置，这个值应该是false
- 注意tar包版本应该与用来提交tez job的客户端版本一致。
4. 可选的：如果在tez上运行已经存在的MR任务，修改mapred-site.xml，修改mapreduce.framework.name, 从yarn修改为yarn-tez.
5. 配置client节点，把tez类库加入到hadoop类库中。
- 抽取tez最小的tar包到本地目录
- 设置TEZ_CONF_DIR为tez-site.xml的位置
- 添加$TEZ_CONF_DIR，${TEZ_JARS}/*和${TEZ_JARS}/lib/*到app的classpath。例如通过标准hadoop工具链设置的话：
```
export HADOOP_CLASSPATH=${TEZ_CONF_DIR}:${TEZ_JARS}/*:${TEZ_JARS}/lib/*
```
- 注意 `*`是必须的

6. 下面是一个提交MR任务的例子：
```
$HADOOP_PREFIX/bin/hadoop jar tez-examples.jar orderedwordcount <input> <output>
```

这个会使用tez dag ApplicationMaster来运行wordcount job。这个wordcount相比简单的，多了个按照词频顺序输出。

Tez DAG可以分别运行在不同的app上，使用同一个TEZ session就可以了。tez-tests中有一个odrderedwordcount是支持session的使用的，同时处理多个input-output pairs。可以在不同的input/output上连续运行多个DAG。

```
$HADOOP_PREFIX/bin/hadoop jar tez-tests.jar testorderedwordcount <input1> <output1> <input2> <output2> <input3> <output3> ...
```


上面的例子就会为每个input-output pair运行多个DAG了。

要使用tez session的话，设置 -DUSE_TEZ_SESSION=true
```
$HADOOP_PREFIX/bin/hadoop jar tez-tests.jar testorderedwordcount -DUSE_TEZ_SESSION=true <input1> <output1> <input2> <output2>
```

7. 像平时一样提交一个MR任务
```
$HADOOP_PREFIX/bin/hadoop jar hadoop-mapreduce-client-jobclient-3.0.0-SNAPSHOT-tests.jar sleep -mt 1 -rt 1 -m 1 -r 1
```

这会使用TEZ DAG ApplicationMaster来运行MR job。可以通过查看YARN UI上的AM的log来验证一下。记住，mapred-site.xml里的mapreduce.framework.name需要设置成yarn-tez。

#### 指定tez.lib.uris的几种方式

tez.lib.uris属性支持逗号分隔的多个值，可以是单个文件，一个目录，压缩包(tar,zip等)。

对于文件和目录，tez会把第一层的文件放到tez运行时的工作目录中，然后放入classpath。对于压缩文件，会被解压到工作目录中。

#### hadoop依赖安装
上面的使用tez的方式，也就是预装在hadoop类库中是我们推荐的方式。带有所有依赖的完整的tar包是一个更好的方式，可以保证已经存在的job在集群回滚或者升级的时候继续正常运行。

尽管`tez.lib.uris`配置项有很广泛的使用模型，但是还有两个主要的可选模式:
- A： 在hadoop类库可用的集群上使用tez tar包
- B： 与hadoop tar包一起使用tez tar包

 这两个模式需要一个没有hadoop依赖而编译地的tez，可以是tez-dist/target/tez-x.y.z-minimal.tar.gz。
 
 #### 对于模式A：通过yarn.application.classpath使用集群已有的hadoop类库
 对于使用rolling  upgrade的集群不推荐这种方式。另外，用户需要负责保证tez版本与正在运行集群的hadoop的兼容性。对于上面的第三个步骤，也需要修改。后续的步骤应该使用tez-dist/target/tez-x.y.z-minimal.tar.gz而不是tez-dist/target/tez-x.y.z.tar.gz。
 - 如果tez jar已经放在了HDFS的/apps里，那么minimal的tez就可以运行了
 ```
 "hadoop fs -mkdir /apps/tez-x.y.z"
"hadoop fs -copyFromLocal tez-dist/target/tez-x.y.z-minimal.tar.gz /apps/tez-x.y.z"
 ```
 - tez-site.xml配置
    - 设置tez.lib.uris为hdfs包含tez jar的位置。
    - 设置tez.use.cluster.hadoop-libs为true

#### 对于模式B：带有hadoo tar包的tez  tar包
这个模式是支持rolling upgrade的。但是用户需要确认自己选择的tez和hadoop版本兼容。也需要修改第三步：
- 假设tez的压缩包在HDFS的/apps下
```
hadoop fs -mkdir /apps/tez-x.y.z 
hadoop fs -copyFromLocal tez-dist/target/tez-x.y.z-minimal.tar.gz /apps/tez-x.y.z
```
- 或者，可以把minimal目录直接放到HDFS，然后再把每个jar包放进去。
```
hadoop fs -copyFromLocal tez-dist/target/tez-x.y.z-minimal/* /apps/tez-x.y.z
```
- 构建完hadoop之后，hadoop tar包在hadoop/hadoop-dist/target/hadoop-x.y.z-SNAPSHOT.tar.gz
- 假设hadoop jar包放在了HDFS上的/apps里
```
hadoop fs -mkdir /apps/hadoop-x.y.z
hadoop fs -copyFromLocal hadoop-dist/target/hadoop-x.y.z-SNAPSHOT.tar.gz /apps/hadoop-x.y.z
```
- tez-site.xml的配置
    - tez.lib.uris只想tez/hadoop需要的jar或者归档文件所在位置
    - 例子：当时用tez和hadoop归档文件时，设置tez.lib.uris为`${fs.defaultFS}/apps/tez-x.y.z/tez-x.y.z-minimal.tar.gz#tez,${fs.defaultFS}/apps/hadoop-x.y.z/hadoop-x.y.z-SNAPSHOT.tar.gz#hadoop-mapreduce`
    - 例子：当时用带有hadoop归档文件的tezjar的时候，设置tez.lib.uris为`${fs.defaultFS}/apps/tez-x.y.z,${fs.defaultFS}/apps/tez-x.y.z/lib,${fs.defaultFS}/apps/hadoop-x.y.z/hadoop-x.y.z-SNAPSHOT.tar.gz#hadoop-mapreduce`
    - 在tez.lib.uris中，跟在`#`后面的文档会自动创建对应的fragment 链接。如果没有给出fragment，那么链接就被设置为归档的名字。fragment不应该是目录或者jar
    - 如果在tez.lib.uris中指定了任何归档，就也要设置tez.lib.uris.classpath，定义好这些归档文件的classpath，因为归档文件结构是未知的。
    - 例子：当使用tez和hadoop归档时，设置tez.lib.uris.classpath：
    ```
    ./tez/*:./tez/lib/*:./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/common/*:./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/common/lib/*:./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/hdfs/*:./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/hdfs/lib/*:./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/yarn/*:./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/yarn/lib/*:./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/mapreduce/*:./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/mapreduce/lib/*
    ```
    - 例子：当使用tez jar和hadoop归档文件时，设置tez.lib.uris.classpath为：
    ```
    ./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/common/*:./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/common/lib/*:./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/hdfs/*:./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/hdfs/lib/*:./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/yarn/*:./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/yarn/lib/*:./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/mapreduce/*:./hadoop-mapreduce/hadoop-x.y.z-SNAPSHOT/share/hadoop/mapreduce/lib/*
    ```


