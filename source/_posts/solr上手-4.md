title: solr上手（四）
date: 2015-08-06 18:16:18
tags: [solr]
---


走马观花过完了solr官方的入门文档，我们下一个目的就是搞清楚Collection的CURD, 以及弄清楚schema在哪里配置或者生成的。

找到官方wiki跟做.

### 创建Collection
> localhost:8983/admin/collections?action=CREATE&name=willCollection&numShards=2&replicationFactor=1

![1 solrCloud创建collection](/imgs/solr/4.1 solrCloud创建collection.png)
![2 创建collection后即可在core看到信息](/imgs/solr/4.2 创建collection后即可在core看到信息.png)

其他操作其实就参考https://cwiki.apache.org/confluence/display/solr/Collections+API就可以了，没什么好抄袭的感觉。


### schema相关

一直纳闷儿solr里是不是像ES那样的数据结构： index -> type -> field？经过各种纠结的实验后，总结出：No！！
配置有两种情况，当运行solr单例的时候，5.2.1版本配置文件是放置在server/configsets/basic_configs/conf/schema.xml的，这里面记录了所有field的schema，没错整个示例都用同一个schema！不分什么type，大家在一起！


![3 schema配置文件](/imgs/solr/4.3 schema配置文件.png)

但是我怎么也找不到例子cloud里的那些字段所在的schema文件，到example和solr目录下都么有。。。功夫不负有心人，想到了zookeeper，连接上去查看一下，果然看到了预期的结果。
![4 solr的zookeeper存储](/imgs/solr/4.4 solr的zookeeper存储.png)

贼不走空，我更不闲看到的信息多，顺便就也看了一下其他的数据节点。

另外，其实solr的schema在restful API也是可查的，这个可参考：https://cwiki.apache.org/confluence/display/solr/Schema+API

![5 solr的线上schema查询](/imgs/solr/4.5solr的线上schema查询.png)
