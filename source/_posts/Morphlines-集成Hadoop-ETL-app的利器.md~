title: 'Morphlines: 集成Hadoop ETL app的利器'
date: 2015-08-05 02:53:16
tags: [Morphlines, ETL, Flume, Solr]
---


> ** Morphlines replaces Java programming with simple configuration steps, reducing the cost and effort of doing custom ETL ** .

--- 

使用Morphlines我们可以不用进行实际的繁琐编程而进行ETL工作，不用了解MR技术。一个morphline是指的一个配置文件，通过这个配置文件，我们可以从任意的数据源消费数据，然后通过一个的transformation chain，最终将处理结果送进某个Hadoop组件中去。相比编程java而言，我们只需要配置一下就可以了 。

Morphlines是一个可嵌入的java工具包，一个morphline是transformation command的一个内置container。command是morphline的Plugin，也就是加载数据、解析数据、转化数据或者其他针对某record进行某些操作。这里说的record是指的内存中的name-values对，它也可以有一些blob附件或者pojo附件。这个框架已经以一种直接而简单的方式将现有的功能与一些第三方系统进行了集成，而且扩展性较好。

Morphlines是服务于cloudera search的。它致力于flume和MR的ETL，前者是实时流，后者是批处理。

这篇文章，我们会通过一个例子讲明如果使用Morphlines怎样收集syslog的输出，并使之能在solr中进行查询。但是在此之前，我们还是先认识一下Morphlines的数据模型和数据处理模型。

### 数据处理模型

Morphlines可以说是Unix pipeline的进化，它将要处理的数据泛化。一个morphline负责的工作：消费record(可以是flume events, HDFS文件，RDBMS 表，或者Apache avro 对象等)，将消费的record转为流，将这些流以管道的形式通过一些列的配置好的transformation，最后进入目的地，例如本例中的solr。
![1 处理流程](/imgs/morphlines/morphline-1.1 处理流程.png)

如上图所示，Flume Source读取syslog后将event弄到Morphlines Sink里去，然后Morphlines Sink把flume的event转为一个record，交给后面的一连串command进行处理。第一个command readLine抽取log行，第二个command grok使用正则抽取log行里的一部分关键信息。第三个command loadSolr就把上个command输出的结构化的信息索引进入solr了。在这个过程中，原始数据或者半结构化数据根据app的模型需求通过一些列转化成为结构化数据被存储。

Morphlines已经有了一些常用的转化和IO操作功能，但是我们仍然可以通过Plugin system加入一些新的转化和IO操作类型，也可以集成其他的第三方框架。

使用Morphlines可以很简单地实现ETL应用间的ETL逻辑的重用。

SDK是在运行时编译的。所有的command都在同一个线程内执行，也就是说pipline是通过某个方法调用另一个方法实现的。不存在队列，不存在线程间对象传递，不存在context切换，不存在command之间的序列化问题。

### 数据模型

Morphlines操作海量数据。一个command可能对一个record进行transformation之后会产生零个或多个record。数据模型如下：一个record包含一些列的name-value对，所有record包含的filed应该是一样的(可以理解成command的产出是有一定的schema的)，而一个name可以对应多个value，一个value可以是任何java对象。这个灵活的数据模型其实和Lucene/solr的数据模型很像。

![2. record数据模型与实例](/imgs/morphlines/morphline-1.2 record数据模型.png)


一个morphline可以处理结构化数据和二进制数据。还有一些可选的附件属性，这些附件属性可以结合Email那些东西进行理解。

这种通用的数据模型可以支持很多应用。典型应用就是Apache flume Morphlines Solr Sink。它把flume event的body放进了morphline record的_attachment_body字段中，event的header信息全部写成record的name-value对。另一个例子就是MapReduceIndexerTool的Mapper，将当前正在处理的HDFS文件的InputStream写入morphline record的_attachment_body字段中，把HDFS文件的元数据信息写成了record的name-value对。

###  用例

command可以操作record的所有字段，包括解析、添加、删除、重命名、分割，甚至还能删除整个record。command有一个返回值来指示处理结果成功或失败。

例如一个多行的输入可以被一个command把每行切分为一个record输出，也就是产生多个record输出，然后另一个command在针对每一行使用正则切分为单词或短语(根据业务需求)，产生更多的record输出。

command可以对record做的操作包括：抽取、清理、转化、join、integrate、enrich、以各种方式装饰record。例如，可以使record与外部数据(RDBMS, KVDB,本地文件等)join。

command还可以消费record，然后把它们传送给某个外部系统。

### 嵌入host系统

一个Morphline没有任何的持久化、分布式计算、可靠性、节点失败恢复等概念——它只是一个当前线程中的transformation chain而已。morphline没必要管理多线程、多节点，因为这些已经有MR,Flume，Storm等去做了。但是morphline还是会给command子树传递一些通知消息：BEGIN_TRANSACTION, COMMIT_TRANSACTION, ROLLBACK_TRANSACTION, SHUTDOWN.

### 语法

配置文件使用HOCON格式，参考http://github.com/typesafehub/config/blob/master/HOCON.md

### 目前可用command

详见http://kitesdk.org/docs/current/kite-morphlines/morphlinesReferenceGuide.html

### syslog案例

**需求**：抽取syslog的内容，并索引到solr里。

> Feb  4 10:46:14 syslog sshd[607]: listening on 0.0.0.0 port 22.

以上为log示例. 我们需要抽取的record如下：

```
priority : 164
timestamp : Feb  4 10:46:14
hostname : syslog
program : sshd
pid : 607
msg : listening on 0.0.0.0 port 22.
message : Feb  4 10:46:14 syslog sshd[607]: listening on 0.0.0.0 port 22.
```

morphline配置 morphline.conf内容如下：

```
morphlines : [
  {
    # Name used to identify a morphline. E.g. used if there are multiple
    # morphlines in a morphline config file
    id : morphline1
 
    # Import all morphline commands in these java packages and their
    # subpackages. Other commands that may be present on the classpath are
    # not visible to this morphline.
    importCommands : ["com.cloudera.**", "org.apache.solr.**"]
 
    commands : [
      {
        # Parse input attachment and emit a record for each input line               
        readLine {
          charset : UTF-8
        }
      }
 
      {
        grok {
          # Consume the output record of the previous command and pipe another
          # record downstream.
          #
          # A grok-dictionary is a config file that contains prefabricated
          # regular expressions that can be referred to by name. grok patterns
          # specify such a regex name, plus an optional output field name.
          # The syntax is %{REGEX_NAME:OUTPUT_FIELD_NAME}
          # The input line is expected in the "message" input field.
          dictionaryFiles : [src/test/resources/grok-dictionaries]
          expressions : {
            message : """%{SYSLOGTIMESTAMP:timestamp} %{SYSLOGHOST:hostname} %{DATA:program}(?:\[%{POSINT:pid}\])?: %{GREEDYDATA:msg}"""
          }
        }
      }
 
      # Consume the output record of the previous command, convert
      # the timestamp, and pipe another record downstream.
      #
      # convert timestamp field to native Solr timestamp format
      # e.g. 2012-09-06T07:14:34Z to 2012-09-06T07:14:34.000Z
      {
        convertTimestamp {
          field : timestamp
          inputFormats : ["yyyy-MM-dd'T'HH:mm:ss.SSS'Z'", ""MMM d HH:mm:ss"]
          inputTimezone : America/Los_Angeles
          outputFormat : "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"
          outputTimezone : UTC
        }
      }
 
      # Consume the output record of the previous command, transform it
      # and pipe the record downstream.
      #
      # This command deletes record fields that are unknown to Solr
      # schema.xml. Recall that Solr throws an exception on any attempt to
      # load a document that contains a field that isn't specified in
      # schema.xml.
      {
        sanitizeUnknownSolrFields {
          # Location from which to fetch Solr schema
          solrLocator : {           
            collection : collection1       # Name of solr collection
            zkHost : "127.0.0.1:2181/solr" # ZooKeeper ensemble
          }
        }
      }
 
      # log the record at INFO level to SLF4J
      { logInfo { format : "output record: {}", args : ["@{}"] } }
 
      # load the record into a Solr server or MapReduce Reducer
      {
        loadSolr {
          solrLocator : {           
            collection : collection1       # Name of solr collection
            zkHost : "127.0.0.1:2181/solr" # ZooKeeper ensemble
          }
        }
      }
    ]
  }
]
```

想要看到每个command处理的record的内容的话，可以配置log4j.properties：
> log4j.logger.com.cloudera.cdk.morphline=TRACE


### 其他文档

- http://www.cloudera.com/content/cloudera/en/documentation/cloudera-search/v1-latest/Cloudera-Search-User-Guide/csug_morphline_example.html
- http://www.slideshare.net/cloudera/using-morphlines-for-onthefly-etl
- https://flume.apache.org/FlumeUserGuide.html#morphlinesolrsink
