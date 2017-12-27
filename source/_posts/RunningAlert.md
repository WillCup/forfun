
title: RunningAlert
date: 2017-08-31 17:20:38
tags: [youdaonote]
---

ReadMe
---
一个大数据集群监控项目，主要依赖于当前比较流行的[Prometheus](https://prometheus.io/)。


流程
---

#### 1. 采集注册

指的是向JsonExporter
```
app_name : h2_streaming_...
url: http://sdfsfdsd:0080/metrics
service_type : spark_streaming
```

提供数据示例参考，也就是马上爬取一下metrics，并显示出来，以方便表达式编写。



#### 2. json_exporter 开始采集
启动marathon上json_exporter的docker镜像，并指定当前对应的参数。开始metric的采集工作。


#### 3. 指定alertManager的告警规则

直接开放给用户直接编辑

```
ALERT {{ pro_alert.alert_name }}
   IF absent(json_val{rowkey=~"{{ pro_alert.key_regex }}", app_name="{{ app_name }}"})
   FOR {{ pro_alert.time_spent }}s
   LABELS { severity = "critical"}
   ANNOTATIONS {
    summary= "{{ pro_alert.summary }}",
    description= "{{ pro_alert.description }}"
  }

ALERT {{ pro_alert.alert_name }}
   IF consul_health_service_status{check="service:web",node="agent-two",service_id="web",service_name="web",status="critical"}  == 1
   FOR {{ pro_alert.time_spent }}s
   LABELS { severity = "critical"}
   ANNOTATIONS {
    summary= "{{ pro_alert.summary }}",
    description= "{{ pro_alert.description }}"
  }
```

之后，保存或者保存并立即生效。

立即生效其实就是调用一下prometheus的alertManager的/-/reload/接口。

之后Prometheus会自动检验对应metric规则，并发送给AlertManager的receiver，并产生告警动作。

