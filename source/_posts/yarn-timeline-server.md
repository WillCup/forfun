
title: yarn-timeline-server
date: 2016-12-05 17:08:58
tags: [youdaonote]
---


> timeline server主要是为yarn管理应用的当前运行状态以及历史状态的，以前也叫history server。

负责管理以下两个事情：

#### 已运行完成的应用基本信息
这些信息包括ApplicationSubmissionContext中的队列名称、用户信息等。
启动一个应用经历过的尝试次数，每次尝试的信息，每次尝试启用的container都有哪些，每个container的基本信息。
这些基本信息被ResourceManager春出道了一个history-store(默认是一个文件系统)然后提供给web-UI来展示这些已完成的应用的这些基本信息。

#### 正在运行或者已完成的应用的Per-framework information
Per-framework information是指针一个框架或者应用特有的信息。例如，Hadoop mapreduce框架会包含一些类似map task数量、reduce task数量、counter等信息。开发者可以自己通过TimeLineClient定制这些信息。这些信息可以通过rest api查询，也可以直接提供给某些特定UI进行渲染。

----

## 1. 当前状态

Timeline sever 还正在开发中.基本信息和框架特定信息已经可以满足了基本的存储于查询，但是目前timeline server还不能在secure模式下正常工作。 
基本信息与框架指定信息是分别进行收集与呈现的，并没有很好的整合在一起。
框架指定信息目前只能通过restful api访问，使用json形式，目前还不支持在yarn里安装框架。

## 2. 基本配置Basic Configuration

用户必须先配置才能启动，下面是一个简单的示例，在yarn-site.xml里设置timeline server的主机名:

```xml
<property>
  <description>The hostname of the Timeline service web application.</description>
  <name>yarn.timeline-service.hostname</name>
  <value>0.0.0.0</value>
</property>
```
## 3. 高级配置

In addition to the hostname, admins can also configure whether the service is enabled or not, the ports of the RPC and the web interfaces, and the number of RPC handler threads.
```xml
<property>
  <description>Address for the Timeline server to start the RPC server.</description>
  <name>yarn.timeline-service.address</name>
  <value>${yarn.timeline-service.hostname}:10200</value>
</property>

<property>
  <description>The http address of the Timeline service web application.</description>
  <name>yarn.timeline-service.webapp.address</name>
  <value>${yarn.timeline-service.hostname}:8188</value>
</property>

<property>
  <description>The https address of the Timeline service web application.</description>
  <name>yarn.timeline-service.webapp.https.address</name>
  <value>${yarn.timeline-service.hostname}:8190</value>
</property>

<property>
  <description>Handler thread count to serve the client RPC requests.</description>
  <name>yarn.timeline-service.handler-thread-count</name>
  <value>10</value>
</property>
```
## 4. 基本信息相关的配置

Users can specify whether the generic data collection is enabled or not, and also choose the storage-implementation class for the generic data. There are more configurations related to generic data collection, and users can refer to yarn-default.xml for all of them.

```xml
<property>
  <description>Indicate to ResourceManager as well as clients whether
  history-service is enabled or not. If enabled, ResourceManager starts
  recording historical data that Timelien service can consume. Similarly,
  clients can redirect to the history service when applications
  finish if this is enabled.</description>
  <name>yarn.timeline-service.generic-application-history.enabled</name>
  <value>false</value>
</property>

<property>
  <description>Store class name for history store, defaulting to file system
  store</description>
  <name>yarn.timeline-service.generic-application-history.store-class</name>
  <value>org.apache.hadoop.yarn.server.applicationhistoryservice.FileSystemApplicationHistoryStore</value>
</property>
```
## 5. 框架特有信息的相关配置

Users can specify whether per-framework data service is enabled or not, choose the store implementation for the per-framework data, and tune the retention of the per-framework data. There are more configurations related to per-framework data service, and users can refer to yarn-default.xml for all of them.

```xml
<property>
  <description>Indicate to clients whether Timeline service is enabled or not.
  If enabled, the TimelineClient library used by end-users will post entities
  and events to the Timeline server.</description>
  <name>yarn.timeline-service.enabled</name>
  <value>true</value>
</property>

<property>
  <description>Store class name for timeline store.</description>
  <name>yarn.timeline-service.store-class</name>
  <value>org.apache.hadoop.yarn.server.applicationhistoryservice.timeline.LeveldbTimelineStore</value>
</property>

<property>
  <description>Enable age off of timeline store data.</description>
  <name>yarn.timeline-service.ttl-enable</name>
  <value>true</value>
</property>

<property>
  <description>Time to live for timeline store data in milliseconds.</description>
  <name>yarn.timeline-service.ttl-ms</name>
  <value>604800000</value>
</property>
```
## 6. 启动timeline server

Assuming all the aforementioned configurations are set properly, admins can start the Timeline server/history service with the following command:
```shell
  $ yarn historyserver
```
Or users can start the Timeline server / history service as a daemon:
```shell
  $ yarn-daemon.sh start historyserver
```
Accessing generic-data via command-line

Users can access applications' generic historic data via the command line as below. Note that the same commands are usable to obtain the corresponding information about running applications.
```shell
  $ yarn application -status <Application ID>
  $ yarn applicationattempt -list <Application ID>
  $ yarn applicationattempt -status <Application Attempt ID>
  $ yarn container -list <Application Attempt ID>
  $ yarn container -status <Container ID>
```
Publishing of per-framework data by applications

Developers can define what information they want to record for their applications by composing TimelineEntity and TimelineEvent objects, and put the entities and events to the Timeline server via TimelineClient. Below is an example:

``` java
  // Create and start the Timeline client
  TimelineClient client = TimelineClient.createTimelineClient();
  client.init(conf);
  client.start();

  TimelineEntity entity = null;
  // Compose the entity
  try {
    TimelinePutResponse response = client.putEntities(entity);
  } catch (IOException e) {
    // Handle the exception
  } catch (YarnException e) {
    // Handle the exception
  }

  // Stop the Timeline client
  client.stop();
```
