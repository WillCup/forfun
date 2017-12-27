
title: hue兼容slider部署的presto
date: 2017-12-26 17:17:34
tags: [youdaonote]
---

通常对于应用部署，尤其是presto这种占用资源比较大，而且可能需要动态扩缩容的场景，我们会考虑使用资源管理框架来进行控制。首先想到的就是yarn和mesos。

我们可以通过slider将presot部署到yarn上，或者使用docker部署persto到mesos上实现资源管控。对于前者slider的服务发现与注册功能稍有欠缺，而后者的话，通过marathon-lb很容易实现服务发现功能，只需要把coordinator注册过去就可了。我们目前暂时使用第一种方案。

前面我们开发了自己的presto connector，但是我们还要面临一个跟jdbc相关的presto的问题，就是当我们使用slider部署presto到yarn的时候，presto的coordinator是变化的。为解决这个问题，我们需要修改一下presto建立数据库连接的位置的代码，使用动态获取yarn应用的方式进行url更新。

两种方案

#### 从yarn爬取

原来的presto connector是从jdbc复制过来，需要的option参数如下：
```py
[[[presto]]]
      name=Presto
      interface=presto
      options='{"password": "***empty***", "url": "jdbc:presto://datanode13.will.com:8099/hive/", "driver": "com.facebook.presto.jdbc.PrestoDriver"}'

```

我们修改为如下：
```
[[[presto]]]
      name=Presto
      interface=presto
      options='{"password": "***empty***", "url": "jdbc:presto://{coordinator_host}:8099/hive/", "driver": "com.facebook.presto.jdbc.PrestoDriver", "app_name": "presto_will_online", "resource_manager": "http://datanode02.will.com:8088"}'

```

然后presto.py原代码：
```py
class PrestoApi(Api):

  def __init__(self, user, interpreter=None):
    global API_CACHE
    Api.__init__(self, user, interpreter=interpreter)

    self.db = None
    self.options = interpreter['options']

    # 爬取yarn上的presto地址，清洗url参数

    if self.cache_key in API_CACHE:
      self.db = API_CACHE[self.cache_key]
    elif 'password' in self.options:
      username = self.options.get('user') or user.username
      self.db = API_CACHE[self.cache_key] = Jdbc(self.options['driver'], self.options['url'], username, self.options['password'])

```




参考： 
- https://hadoop.apache.org/docs/r2.7.3/hadoop-yarn/hadoop-yarn-site/ResourceManagerRest.html#Cluster_Application_Statistics_API


#### presto coordinator启动后自动注册当前主机

找到presto-yarn项目解压后的脚本`package/scripts/presto_coordinator.py`内容
```py
from presto_server import PrestoServer

if __name__ == "__main__":
    PrestoServer('COORDINATOR').execute()
    
    # 添加向指定位置注册当前主机名的代码, 可以注册到redis、mysql、hdfs、zk等
    
    
    
```

最终选用的方案是方案一，考虑是已经对hue有了一些二次开发了。就直接把修改放在hue上吧，后续我们还是有可能把presto迁移到docker上的。




