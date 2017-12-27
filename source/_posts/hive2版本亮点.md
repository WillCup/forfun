
title: hive2版本亮点
date: 2017-11-15 10:53:01
tags: [youdaonote]
---

- LLAP。

Live Long and Process智能将数据缓存到多台机器的内存中，并允许所有客户端共享这些缓存的数据，同时保留了弹性伸缩能力。开启LLAP后，性能相比1版本提升25倍。

LLAP提供了一个高级的执行模式，他启用一个长时间存活的守护程序去和HDFS DataNode直接交互，也是一个紧密集成的DAG框架。这个守护程序中加入了缓存、预抓取、查询过程和访问控制等功能。短小的查询由守护程序执行，大的重的操作由YARN执行。
- 支持使用HPL/SQL的存储过程

支持使用变量、表达式、控制流声明、迭代来实现业务逻辑，支持使用异常处理程序和条件处理器来实现高级错误处理。

使用sql on hadoop更加动态：支持使用高级表达式、各种内置函数，基于用户配置、先前查询的结果、来自文件或者非hadoop数据源的数据、即时动态的生成SQL条件。

利用已有的存储过程SQL，提供函数和声明。

方便集成和支持多种类型数据仓库，可以实现单个脚本处理hadoop、RDBMS、Nossql等多个系统的数据。

- 更智能的成本优化其CBO
- 提供全面的监控和诊断工具。hive server2的UI界面、LLAP 的UI、Tez的UI等。




升级只需要升级hive的元数据库信息即可，2版本提供了升级脚本
```
hive-metastore/bin/schematool -upgradeSchema -dbType
```
