title: ambari-web解读（一）-请求流程
date: 2016-06-20 15:26:42
tags: [ambari, 源码]
---

最近为公司选型了大数据维护工具ambari，官网:ambari.apache.org。主要是为了运维方便一些。
另外对于数据分析人员开放了一个hive view页面，提供提数支持。但是对于原始的hive view，我们有些不满意的地方：
1. table的comment没有展示完全
2. 后台一直在每15秒钟请求一次后台hiveserver。虽然这并不会对和iveserver造成很大压力，但是感觉不友好，产生很多垃圾log。

经过差不多一周左右断断续续地探查与学习，基本弄清楚了ambari-web的请求处理脉路。可能对nodejs大神来说这不算啥，但是个人感觉稍微有点儿吃力。学习了一下nodejs的新的工具ember.js。


### 1. 


### 参考
|链接|说明|
|---|---|
|https://github.com/emberjs/starter-kit/| 学习ember.js的基础项目 |
|http://www.html-js.com/article/1685|Ember.js Application Development How-to的中文翻译|
|http://www.ruanyifeng.com/blog/2011/08/a_detailed_explanation_of_jquery_deferred_object.html|对于jquery回调工具deferred的解释|
