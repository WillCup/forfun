
title: Prometheus关键概念
date: 2017-09-01 11:28:51
tags: [youdaonote]
---

job和instance
在prometheus看来，每个单独的抓取目标是一个instance，通常对应一个单独的进程。同种类型的instance的集合组成一种job。


例如，一个API server的job会有下面的instance：
- job: api-server
    - instance 1.2.3.4:5769
    - instance 1.2.3.4:5768

自动产生的label
---
在prometheus抓取target的时候，会自动在后面追加一些label：
 - job
 - instance

如果这些label是否已经出现在抓取的数据中，就参考honor_labels的配置部分。

每个instance的抓取，prometheus都存储一个sample：
 - up{job="<job-name>", instance="<instance-id>"}: 1 表示这个instance是否健康的存在
 - scrape_duration_seconds{job="<job-name>", instance="<instance-id>"}: 抓取时间
 - scrape_samples_post_metric_relabeling{job="<job-name>", instance="<instance-id>"}:metric被relabel的sample个数
 - scrape_samples_scraped{job="<job-name>", instance="<instance-id>"}: target中暴露的sample个数
 
up时间序列一般用来监控instance的可用性。



但是，如果这样的话，我的instance就没有办法动态添加进去，因为Prometheus并没有提供动态修改target的REST API.

https://prometheus.io/docs/querying/api/

[一片不错的prometheus监控使用总结](http://ordina-jworks.github.io/monitoring/2016/09/23/Monitoring-with-Prometheus.html#architecture)


相关链接： https://groups.google.com/forum/#!topic/prometheus-users/fTPKx5Eg8bo

https://github.com/prometheus/jmx_exporter/pull/180#issuecomment-326229999
