
title: ambari源码解析-告警问题
date: 2017-11-02 11:34:39
tags: [youdaonote]
---


按照官网部署了一个简单的notice target脚本，就是把所有的脚本参数都输出到一个临时文件，然后发现虽然ambari server的web页面中有一大堆alert，但是实际触发alert_notice的很少，而且与web页面中看到的alert定义的check interval严重不符。到ambari-server的log里也没有找到heartbeat的连续信息之类。。。
```
echo $* >> /tmp/ambari_alerts
```

回顾一下上面的代码: 只有接收到AlertStateChangeEvent才会触发持久化到alert_notice表，也就是说只针对alert有状态变更才发送notice。尼玛，那如果报了一次之后，后面一直没有收到新的相关notice的话，就默认它是原状？？？ 细想好像这样也对，可以省掉最新。可是并不符合我们最初设想的持续告警。


另外还有一个问题，就是脚本参数里并不包含对应的host信息：
```
datanode_heap_usage DataNode Heap Usage HDFS WARNING Used Heap:[81%, 1633.3276 MB], Max Heap: 2028.0 MB
datanode_heap_usage DataNode Heap Usage HDFS OK Used Heap:[78%, 1587.691 MB], Max Heap: 2028.0 MB
datanode_heap_usage DataNode Heap Usage HDFS WARNING Used Heap:[80%, 1623.7538 MB], Max Heap: 2028.0 MB
datanode_heap_usage DataNode Heap Usage HDFS OK Used Heap:[78%, 1584.6304 MB], Max Heap: 2028.0 MB
datanode_heap_usage DataNode Heap Usage HDFS OK Used Heap:[78%, 1577.2762 MB], Max Heap: 2028.0 MB
datanode_heap_usage DataNode Heap Usage HDFS WARNING Used Heap:[81%, 1650.835 MB], Max Heap: 2028.0 MB
```

看来需要一系列的代码改造才能完全满足我们的需求，就是每次收到heartbeat中的alert信息之后，根据优先级进行过滤，这个优先级最好是读取配置文件的。然后把所有的alert都持久化到alert_notice表中。如果要兼容官网原生的话，就在AlertReceivedListener的onAlertEvent的修改中加入一个option，比如1代表使用官网的notice，2代表使用自定义的notice，这个option可以是全局的，也可以是附属在alert_definition里的【后者需要修改alert_definition, 动作偏大，而且一般环境中只有一种告警方式就够了】。

写点儿伪代码，体会一下，先修改AlertReceivedListener.onAlertEvent。
```java
....
for (Map.Entry<Alert, AlertCurrentEntity> entry : toCreateHistoryAndMerge.entrySet()) {
  Alert alert = entry.getKey();
  AlertCurrentEntity entity = entry.getValue();
  Long clusterId = getClusterIdByName(alert.getCluster());

  boolean fire = false;
  if(m_configuration.getCustom_alert() && alert.getState() == AlertState.CRITICAL) {
    fire = true;
  } else {
    if (clusterId == null) {
      //super rare case, cluster was removed after isValid() check
      LOG.error("Unable to process alert {} for an invalid cluster named {}",
              alert.getName(), alert.getCluster());
      continue;
    }
    fire = true;
  }
  if (fire) {
    AlertStateChangeEvent alertChangedEvent = new AlertStateChangeEvent(clusterId, alert, entity,
            oldStates.get(alert));

    m_alertEventPublisher.publish(alertChangedEvent);
  }
}
...
```
其实按照编程命名规范，如果是所有的alert都要按照级别过滤后发送的话，最好自定义一种类型的event，然后再添加一个对应的eventListener。


鉴于上面提到的没有host信息，也要host加到dispatcher的参数中。我们整一下AlertScriptDispatcher, 发现他的接口参数AlertNotification中是包含hostname的

```
ProcessBuilder getProcessBuilder(String script, AlertNotification notification) {
    final String shellCommand;
    final String shellCommandOption;
    if (SystemUtils.IS_OS_WINDOWS) {
      shellCommand = "cmd";
      shellCommandOption = "/c";
    } else {
      shellCommand = "sh";
      shellCommandOption = "-c";
    }

    AlertInfo alertInfo = notification.getAlertInfo();
    AlertDefinitionEntity definition = alertInfo.getAlertDefinition();
    String definitionName = definition.getDefinitionName();
    AlertState alertState = alertInfo.getAlertState();
    String serviceName = alertInfo.getServiceName();
    // 这里
    String hostname = alertInfo.getHostName();

    // these could have spaces in them, so quote them so they don't mess up the
    // command line
    String alertLabel = "\"" + SHELL_ESCAPE.escape(definition.getLabel()) + "\"";
    String alertText = "\"" + SHELL_ESCAPE.escape(alertInfo.getAlertText()) + "\"";

    // 构建的参数里也加上
    Object[] params = new Object[] { script, definitionName, alertLabel, serviceName, hostname,
        alertState.name(), alertText };

    String foo = StringUtils.join(params, " ");

    // sh -c '/foo/sys_logger.py ambari_server_agent_heartbeat "Agent Heartbeat"
    // AMBARI CRITICAL "Something went wrong with the host"'
    return new ProcessBuilder(shellCommand, shellCommandOption, foo);
  }
```


至此，改造完成。构建完成后，发布到ambari-server的对应位置，重启就可以了。
