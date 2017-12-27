
title: ambari源码解析-自定义告警
date: 2017-10-31 15:35:50
tags: [youdaonote]
---

ambari原生支持两种告警手段：邮件告警、SNMP告警。

个人主要是研发，对SNMP一直不太了解，试着搜了一些资料，稍显麻烦，而且目前我们的数据集群运维工作中并没有使用到SNMP相关的工具，所以暂时并不认为必要。


我们期望的告警方式是短信告警，具有最高的时效性，已经具有了短信告警接口。所以下一步就是在ambari这边找到调用此接口，并传送有效告警信息的方式。

翻阅文档，发现amabri也是支持[脚本告警](https://cwiki.apache.org/confluence/display/AMBARI/Creating+a+Script-based+Alert+Dispatcher)的。单独看文档稍微有些懵逼，还是看下对应源码吧。

关键字dispatcher，找到对应的类AlertScriptDispatcher。

```
 /**
   * 在amabri.properties中配置执行告警的脚本的配置项
   */
  public static final String SCRIPT_CONFIG_DEFAULT_KEY = "notification.dispatch.alert.script";

  /**
   * 在amabri.properties中配置以毫秒为单位的执行告警超时的脚本的配置项，默认是5000，也就是5s
   */
  public static final String SCRIPT_CONFIG_TIMEOUT_KEY = "notification.dispatch.alert.script.timeout";

  /**
   * 用来定义指定执行告警脚本的配置项的关键字，如果没有配置的话，就默认是SCRIPT_CONFIG_DEFAULT_KEY[也就是notification.dispatch.alert.script]
   */
  public static final String DISPATCH_PROPERTY_SCRIPT_CONFIG_KEY = "ambari.dispatch-property.script";
```

调用逻辑
```java
public void dispatch(Notification notification) {
    String scriptKey = getScriptConfigurationKey(notification);
    String script = m_configuration.getProperty(scriptKey);
    .....
    .....

    // execute the script asynchronously
    long timeout = getScriptConfigurationTimeout();
    TimeUnit timeUnit = TimeUnit.MILLISECONDS;
    AlertNotification alertNotification = (AlertNotification) notification;
    ProcessBuilder processBuilder = getProcessBuilder(script, alertNotification);

    AlertScriptRunnable runnable = new AlertScriptRunnable(alertNotification, script,
        processBuilder,
        timeout, timeUnit);

    m_executor.execute(runnable);
  }
```

脚步组装
```java
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

    ....
    ....
    
    Object[] params = new Object[] { script, definitionName, alertLabel, serviceName,
        alertState.name(), alertText };

    String foo = StringUtils.join(params, " ");
    
    // 示例
    // sh -c '/foo/sys_logger.py ambari_server_agent_heartbeat "Agent Heartbeat"
    // AMBARI CRITICAL "Something went wrong with the host"'
    return new ProcessBuilder(shellCommand, shellCommandOption, foo);
  }
```

对应了文档中
```
# :param definitionName: alert的唯一名字
# :param definitionLabel: 可读性强的alert标签
# :param serviceName: 告警对应的service名称
# :param alertState: 告警状态 (OK, WARNING, etc)
# :param alertText: 告警内容
 
def main():
    definitionName = sys.argv[1]
    definitionLabel = sys.argv[2]
    serviceName = sys.argv[3]
    alertState = sys.argv[4]
    alertText = sys.argv[5]
```


负责告警触发的类是AlertNoticeDispatchService。它会扫描数据表`alert_notice`中的pending状态的告警实例，通过告警分发系统处理这些告警实例。然后分发系统会回调`AlertNoticeDispatchCallback`处理告警实例的状态更新。
