
title: apache-atlas数据治理
date: 2017-08-22 14:46:36
tags: [youdaonote]
---

Atlas是一个可扩展的一系列的核心基础管理服务。



特性
- 数据分类
    - 为数据定义面向业务的分类标签
    - 定义、annotate、自动获取数据集之间的关系【source、target、中间处理的process】
    - 导出元数据到第三方系统
- 中心化监听
    - 捕获每个app、process等数据交互的安全信息
    - 捕获每个步骤【execution、step、activities】的操作信息
- 搜索和lineage
    - 预定义导航路径来检索数据分类和监听信息
    - 基于文本的索引，定位对应数据和监听的事件
    - 对数据集的血统可视化，可以看到操作、安全等信息
- 安全和策略引擎
  - 基于数据分类、属性、角色的运行时Rationalize compliance policy
  - Advanced definition of policies for preventing data derivation based on classification (i.e. re-identification) – Prohibitions
  - Column and Row level masking based on cell values and attibutes.
