
title: azkaban文档翻译
date: 2016-12-05 17:05:04
tags: [youdaonote]
---

### mysql的作用

mysql被用来存储一些状态信息，azkaban的AzkabanWebServer和AzkabanExecutorServer都会访问mysql来获取与更新状态。

AzkabanWebServer使用Mysql做什么？

 - 项目管理。管理项目以及相关权限，还有上传的文件；
 - 正在执行的flow的状态。追踪在执行的flow，并且查看那个executor正在执行这个flow。
 - 查看以前的flow或者job。
 - 调度器。保存调度完成的job的状态。
 - SLA。保存所有的sla规则。

AzkabanExecutorServer使用Mysql做什么？

 - 访问project。从db里抽取project文件。
 - 执行flow/job。抽取并更新flow的数据。
 - 日志。保存job和flow的执行日志到db里。
 - interflow的依赖。如果一个flow运行在不同的executor上，则应当把所有的状态存储到db里去。

### AzkabanWebServer
用来管理azkaban，管理项目、认证、调度、监控。
azkaban使用*.job的kv值文件来定义flow中的task，_dependencies_用来定义job chain之间的依赖。这个job文件和相关的代码可以打成一个zip包通过web server上传。

### AzkabanExecutorServer
以前web server和executor server是在一起的，分离的原因是：我们预期要能够给executor自由快速扩展的能力。另外如果azkaban升级的话，对我们的用户的影响也更小，因为其实azkaban的发展还是很快的。

