
title: prometheus自定义exporter
date: 2017-08-18 15:30:10
tags: [youdaonote]
---

可维护性和简洁度
---
最核心的地方是考虑一下获取metrics的成本与途径。

如果已经有了一个不错的、不怎么修改的metrics，那么就很容易搞定。例如haproxy exporter

如果有上百个metrcis，而且随着版本变化metric也一直在变，那就会有得受了。例如mysql exporter 


node exporter介于两者之间，是个别的module总在变化。


配置
---
exporter不应该关心app在哪里的配置，也能够使用filter过滤一些太稀碎的metric。

与监控系统协作的时候，framework和protocol相关的东西会很复杂。

最好是既有系统的数据结构能有prometheus一致，或者近似。这样就可以自动转化metric的形式，例如Cloudwatch, SNMP and Collectd。

其实更多情况是不那么标准的。例如jmx exporter， 用户必须说明怎样进行metric转化才行。


YAML是标准的prometheus配置文件格式。


metrics
---

#### 命名

#### label
避免使用type作为label名称，太通用，而且无意义。应该是用避免使用一些类似region, zone, cluster,az,datacenter,dc,woner,customer,stage,environment, env等字眼。

#### 实例

```python
from prometheus_client import start_http_server, Metric, REGISTRY
import json
import requests
import sys
import time


class JsonCollector():

    def init(self, endpoint, appname, srvtype, separator='__'):
        self._endpoint = endpoint
        self._appname = appname
        self._separator = separator
        self._srvtype = srvtype
    
    def __init__(self, target):
        self.init(target.url, target.appname, target.srvtype)


    def handle_type(self, prefix, d, metric
        '''
        本来是应该使用具体的metric类型的，调用其add_metric接口。
        但是由于这是一个JSON到prometheus的通用方法，所以没有那样处理。
        '''
        prefix = prefix.upper()
        if isinstance(d, dict):
            self.extract_dict(prefix, d, metric)
        elif isinstance(d, list):
            self.extract_list(prefix, d, metric)
        else:
            if not isinstance(d, (str, unicode)):
                metric.add_sample('json_val',
                                  value=d, labels={'app_name': self._appname, 'rowkey': prefix})
            else:
                try:
                    val = float(d)
                    metric.add_sample('json_val',
                                  value=val, labels={'app_name': self._appname, 'rowkey': prefix})
                except ValueError:
                    print('Error convert %s to float for %s' % (d, prefix))
    
    
    def extract_dict(self, prefix, dd, metric):
        for k, v in dd.items():
            if len(prefix) == 0:
                self.handle_type(k, v, metric)
            else:
                self.handle_type(prefix + self._separator + k, v, metric)
    
    
    def extract_list(self, prefix, dd, metric):
        for i in dd:
            if len(prefix) == 0:
                self.handle_type(dd.index(i), i, metric)
            else:
                self.handle_type(prefix + self._separator + dd.index(i), i, metric)

    def collect(self):
        '''
        此处需要注意，接口返回的是这里的结果，不要把metric弄成成员变量。
        应该每次都new一个返回去，不然add_sample会把所有的变量都累加进去
        '''
        try:
            response = json.loads(requests.get(self._endpoint).content.decode('UTF-8'))
            metric = Metric(self._srvtype, 'spark program id', 'summary')
            self.handle_type('', response, metric)
            yield metric
        except Exception, e:
            print e.message


class Target():
    def __init__(self, srvtype, appname, url):
        self.srvtype = srvtype
        self.appname = appname
        self.url = url

if __name__ == '__main__':
    # Usage: json_exporter.py port
    
    print(sys.argv)
    start_http_server(int(sys.argv[1]))
    targets = list()
    targets.append(Target('flume', 'haijun_flume_test1', 'http://10.2.19.94:34545/metrics'))
    targets.append(Target('spark_streaming', 'h5_streaming_test1', 'http://datanode02.will.com:8088/proxy/application_1501827559666_32228/metrics/json'))
    for target in targets:
        REGISTRY.register(JsonCollector(target))
    #REGISTRY.register(JsonCollector())

    while True: time.sleep(10)
```


下面是爬取yarn中指定名称规范的运行中的app的简单示例代码。
```python
import requests
import json
import re


url = 'http://datanode02.will.com:8088/ws/v1/cluster/apps?states=RUNNING'
html_doc = requests.get(url).content
result = json.loads(html_doc)
for app in result['apps']['app']:
    if re.match('.*_online', app['name']): 
        print(app['name'], app['trackingUrl'] + 'metrics/json')

```
