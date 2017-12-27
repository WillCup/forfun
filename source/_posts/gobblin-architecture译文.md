
title: gobblin-architecture译文
date: 2017-06-09 14:44:08
tags: [youdaonote]
---

概览
---
gobblin主要考虑的是扩展性，永固可以添加新的adapter或者集成已经存在的adapter来加入新的source，从新的source抽取数据。架构图：
![架构图](http://gobblin.readthedocs.io/en/latest/img/Gobblin-Architecture-Overview.png)

一个gobblin job是一些列组件协同的结果。所有的组件都是可插拔可扩展的。(http://gobblin.readthedocs.io/en/latest/Gobblin-Architecture#gobblin-constructs)

一个gobblin job有一些列的task组成，每个task对应着一个工作单元，负责抽取一部分数据。这些task会根据配置选择(红色部分)通过gobblin runtime执行(上面橙色的部分)。



http://gobblin.readthedocs.io/en/latest/Gobblin-Architecture/

