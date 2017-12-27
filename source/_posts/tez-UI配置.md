
title: tez-UI配置
date: 2017-12-11 18:43:28
tags: [youdaonote]
---


#### 准备

将tez根目录下的war包copy到tomcat的webapps目录中（0.8.5版本有两个ui包,但是没有感觉到有什么区别）
```
cp ~/apache-tez-0.8.5-bin/tez-ui-0.8.5.war /server/tomcat9/webapps
```
重启tomcat，war包会自动发布。修改配置文件里的timeline-service和resourcemanager的位置信息后，重启tomcat。
```
timeline: "http://datanode04.will.com:8188",
rm: "http://datanode02.will.com:8088",
```

#### 配置tez-site.xml
```
<property>
  <name>tez.tez-ui.history-url.base</name>
  <value>http://schedule.will.com:8181/tez-ui-0.8.5</value>
</property>

<property>  
    <name>tez.history.logging.service.class</name>  
    <value>org.apache.tez.dag.history.logging.ats.ATSHistoryLoggingService</value>  
</property>

```

#### 问题

设置yarn-site.xml中timeline-service的cros属性为true。之后重启timeline-service

```
yarn.timeline-service.http-cross-origin.enabled = true
```

其实tez ui就是把所有tez类型的app历史拉出来，按照tez的特性进行定制化的完美展示。

参考：
 - https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=61331897
 - https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/TimelineServer.html
