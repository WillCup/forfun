
title: label的生命周期
date: 2017-09-04 17:41:38
tags: [youdaonote]
---

Prometheus label让我们为我们的app部署与组织间建立模型。直接支持所有可能出现的配置是不太可能，但是prometheus给力一个relabel功能，给出了足够大的灵活性。

基本原则：你的服务发现给你提供了一些元数据，例如机器类型、tag、region，这些都可以在__meta_*的label中找到。然后你可以在relbel_configs配置中把他们进行定制化修改。还可以制定drop和keep操作。

在我们抓取target的时候也一样，metric_relabel_config配置可以让我们纠正数据的时间戳。还可以过滤掉一些比较[昂贵，比如 太大](https://www.robustperception.io/dropping-metrics-at-scrape-time-with-prometheus/)的metric。

为了更好的理解这些内容，看下图：
![](http://www.robustperception.io/wp-content/uploads/2016/03/Life-of-a-Label-Target-Labels.png)

这个图包含了创建target、爬取、在时序数据插入DB之前的操作等。

- 从服务发现获取target
- 基于配置设置__param_*的label
- 如果job,__scheme__或者__metrics_path__的label没有设置，就基于配置设置一下
- 是否有__address__的label
    - 没有，丢掉这个target
    - 有，执行relabel_config
        - drop操作的话直接drop掉
        - __address__有么有端口
            - 有端口， __address__是否包含 "/"
                - drop target
                - 删除所有以__meta_开头的label
            - 没有端口，__scheme__是http就默认80，https就默认443


下面是实际爬取过程中发生的事情：
![爬取流程图](http://www.robustperception.io/wp-content/uploads/2016/03/Life-of-a-Label-Scraping.png)

__param_*的label包含每个URL参数的第一个值，可以进行relabel。在爬取的时候，这些会与第二个以及后续的参数值连接在一起。

因为metric_relabel_configs需要每次爬取的时候都执行一次，最好还是通过improve instrumentation。
