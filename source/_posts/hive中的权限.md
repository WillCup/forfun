
title: hive中的权限
date: 2016-12-05 17:05:20
tags: [youdaonote]
---


## 用例
 1. 把hive当做存储工具。使用pig、MR等并行计算工具访问hive，hdfs权限通过文件认证来解决。metadata的访问需要通过hive配置的认证。
 2. 把hive作为一个查询引擎。又分为两种：
 - a. hive命令行。这些用户可以有hdfs权限，也能看到hive的metastore，有点儿像1。
 - b. jdbc等通过hiveserver2访问。他们不能访问HDFS。

三种认证机制。
##### Storage Based Authorization in the Metastore Server
在1和2.a里，用户可以直接访问到数据，hive没有控制这块。hdfs权限验证担当了主要任务。当你访问一个database、table、partition的时候，他会检查你是否有对应目录的权限。为了看到通过hiveserver2访问的用户的真实信息，需要设置hive.server2.enable.doAs为true。
##### SQL Standards Based AuthorizationSQL Standards Based Authorization in HiveServer2
虽然Storage Based Authorization可以控制对database、table、partition的访问，但是他不能很好地控制对字段和view的访问，因为他是基于文件系统权限控制的。好的控制应该是可以控制用户访问的行和列。hiveserver2可以满足这种需求，可以只提供你的sql需要的行和列。

SQL Standards Based Authorization是基于sql标准的，可以使用revoke/grant语句控制权限。但是必须在hiveserver2的配置里设置一下。

注意如果启用了SQL Standards Based Authorization，那么2a这种方式就被禁用了。因为安全起见，不可能使用hive里的访问控制策略来控制hive命令行，用户可以直接访问HDFS，这样就直接绕过了认证机制。
##### Default Hive Authorization (Legacy Mode)
这并不能完全控制好权限，会有很多漏洞。例如，没有统一的分配权限的用户，所有的用户都可以自己grant权限。



## 参考
 - https://cwiki.apache.org/confluence/display/Hive/Storage+Based+Authorization+in+the+Metastore+Server
 - https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Authorization
