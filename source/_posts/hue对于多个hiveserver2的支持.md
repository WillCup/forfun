
title: hue对于多个hiveserver2的支持
date: 2017-04-26 16:26:28
tags: [youdaonote]
---

相关连接代码：
```python
def hiveserver2_jdbc_url():
  urlbase = 'jdbc:hive2://%s:%s/default' % (beeswax.conf.HIVE_SERVER_HOST.get(),
                                            beeswax.conf.HIVE_SERVER_PORT.get())
  if get_conf().get(_CNF_HIVESERVER2_USE_SSL, 'FALSE').upper() == 'TRUE':
    return '%s;ssl=true;sslTrustStore=%s;trustStorePassword=%s' % (urlbase,
            get_conf().get(_CNF_HIVESERVER2_TRUSTSTORE_PATH),
            get_conf().get(_CNF_HIVESERVER2_TRUSTSTORE_PASSWORD))
  else:
     return urlbase
```

可以看到，除非启用SSL，否则咋样都拼不进去下面这种串：
```python
jdbc:hive2://zkNode1:2181,zkNode2:2181,zkNode3:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2_zk
```


Tip:

可以先用tengine配置TCP代理，然后在这里实现负载均衡，hue端只需要配置代理的地址就可以了。 —— 希望hive的driver可以支持，有待测试。

参考：
- http://lxw1234.com/archives/2016/05/675.htm
