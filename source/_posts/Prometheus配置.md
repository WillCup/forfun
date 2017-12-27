
title: Prometheus配置
date: 2017-09-01 15:34:21
tags: [youdaonote]
---

主要定义了需要抓取的job和这些job所包含的instance，还有告警规则文件。

可以执行 prometheus -h 来查看具体支持的命令行参数。

Prometheus可以在运行时reload，如果新的配置是有效的，我们只需要发送一个SIGHUP给到Prometheus进程，或者发送一个HTTP POST请求到/-/reload终端REST API。


配置文件
---
可以通过-config.file指定具体文件，需要是yaml格式。

下面是来源于https://github.com/prometheus/prometheus/blob/master/config/testdata/conf.good.yml的示例配置文件：

```
# 全局配置
global:
  scrape_interval:     15s
  evaluation_interval: 30s
  # scrape_timeout is set to the global default (10s).

  external_labels:
    monitor: codelab
    foo:     bar

# 告警规则文件
rule_files:
- "first.rules"
- "my/*.rules"

# 与实际远程写入相关的配置
remote_write:
  - url: http://remote1/push
    write_relabel_configs:
    - source_labels: [__name__]
      regex:         expensive.*
      action:        drop
  - url: http://remote2/push

# 抓取配置
scrape_configs:
- job_name: prometheus

  honor_labels: true
  # scrape_interval is defined by the configured global (15s).
  # scrape_timeout is defined by the global default (10s).

  # metrics_path defaults to '/metrics'
  # scheme defaults to 'http'.

  #基于文件的服务发现
  file_sd_configs:
    - files:
      - foo/*.slow.json
      - foo/*.slow.yml
      - single/file.yml
      refresh_interval: 10m
    - files:
      - bar/*.yaml

  static_configs:
  - targets: ['localhost:9090', 'localhost:9191']
    labels:
      my:   label
      your: label

  relabel_configs:
  - source_labels: [job, __meta_dns_name]
    regex:         (.*)some-[regex]
    target_label:  job
    replacement:   foo-${1}
    # action defaults to 'replace'
  - source_labels: [abc]
    target_label:  cde
  - replacement:   static
    target_label:  abc
  - regex:
    replacement:   static
    target_label:  abc

  bearer_token_file: valid_token_file


- job_name: service-x

  basic_auth:
    username: admin_name
    password: "multiline\nmysecret\ntest"

  scrape_interval: 50s
  scrape_timeout:  5s

  sample_limit: 1000

  metrics_path: /my_path
  scheme: https

  dns_sd_configs:
  - refresh_interval: 15s
    names:
    - first.dns.address.domain.com
    - second.dns.address.domain.com
  - names:
    - first.dns.address.domain.com
    # refresh_interval defaults to 30s.

  relabel_configs:
  - source_labels: [job]
    regex:         (.*)some-[regex]
    action:        drop
  - source_labels: [__address__]
    modulus:       8
    target_label:  __tmp_hash
    action:        hashmod
  - source_labels: [__tmp_hash]
    regex:         1
    action:        keep
  - action:        labelmap
    regex:         1
  - action:        labeldrop
    regex:         d
  - action:        labelkeep
    regex:         k

  metric_relabel_configs:
  - source_labels: [__name__]
    regex:         expensive_metric.*
    action:        drop

- job_name: service-y

# 基于consul的服务发现
  consul_sd_configs:
  - server: 'localhost:1234'
    token: mysecret
    services: ['nginx', 'cache', 'mysql']
    scheme: https
    tls_config:
      ca_file: valid_ca_file
      cert_file: valid_cert_file
      key_file:  valid_key_file
      insecure_skip_verify: false

  relabel_configs:
  - source_labels: [__meta_sd_consul_tags]
    separator:     ','
    regex:         label:([^=]+)=([^,]+)
    target_label:  ${1}
    replacement:   ${2}

- job_name: service-z

  tls_config:
    cert_file: valid_cert_file
    key_file: valid_key_file

  bearer_token: mysecret

- job_name: service-kubernetes

  kubernetes_sd_configs:
  - role: endpoints
    api_server: 'https://localhost:1234'

    basic_auth:
      username: 'myusername'
      password: 'mysecret'

# 基于kubernetes的服务发现
- job_name: service-kubernetes-namespaces

  kubernetes_sd_configs:
  - role: endpoints
    api_server: 'https://localhost:1234'
    namespaces:
      names:
        - default

# 基于marathon的服务发现
- job_name: service-marathon
  marathon_sd_configs:
  - servers:
    - 'https://marathon.example.com:443'

    tls_config:
      cert_file: valid_cert_file
      key_file: valid_key_file

- job_name: service-ec2
  ec2_sd_configs:
    - region: us-east-1
      access_key: access
      secret_key: mysecret
      profile: profile

- job_name: service-azure
  azure_sd_configs:
    - subscription_id: 11AAAA11-A11A-111A-A111-1111A1111A11
      tenant_id: BBBB222B-B2B2-2B22-B222-2BB2222BB2B2
      client_id: 333333CC-3C33-3333-CCC3-33C3CCCCC33C
      client_secret: mysecret
      port: 9100

- job_name: service-nerve
  nerve_sd_configs:
    - servers:
      - localhost
      paths:
      - /monitoring

- job_name: 0123service-xxx
  metrics_path: /metrics
  static_configs:
    - targets:
      - localhost:9090

- job_name: 測試
  metrics_path: /metrics
  static_configs:
    - targets:
      - localhost:9090

- job_name: service-triton
  triton_sd_configs:
  - account: 'testAccount'
    dns_suffix: 'triton.example.com'
    endpoint: 'triton.example.com'
    port: 9163
    refresh_interval: 1m
    version: 1
    tls_config:
      cert_file: testdata/valid_cert_file
      key_file: testdata/valid_key_file

# 关于alertmanager相关的配置
alerting:
  alertmanagers:
  - scheme: https
    static_configs:
    - targets:
      - "1.2.3.4:9093"
      - "1.2.3.5:9093"
      - "1.2.3.6:9093"
```

scrape_config部分描述一些列的target和参数，指定抓取策略。一般情况下，一个scrape配置就对应一个job。

target可以是通过static_config静态配置的，也可以是使用某种动态发现机制进行动态获取的。

另外，relabel_config允许在抓取之前更新任意target的label。


已经测试过file_sd_config是可以跟随文件的变化，动态调整target的。
但是对于rule文件的reload，并不能自动识别并执行，官网建议是通过[**定时触发prometheus的reload**](https://github.com/prometheus/prometheus/issues/108)来实现这个东西。
个人认为：如果rule 的修改也并不是那么频繁，而且既有的app里已经包含队列的话，可以在每次修改rule之后，每隔几分钟消费一次reload事件。或者可以修改prometheus的rule相关代码，让其定时消费mysql或者某些目录的rule文件，如有变化则更新rule的缓存。

其他与动态加载rule相关 
- https://groups.google.com/forum/#!topic/prometheus-developers/UsiCxDbFjN0
- https://www.robustperception.io/reloading-prometheus-configuration/
- 
