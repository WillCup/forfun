
title: hue中kylin作为jdbc数据源的问题
date: 2017-12-26 11:49:12
tags: [youdaonote]
---

主要就是普通的jdbc能够提供获取database列表、获取table列表的功能sql。而kylin是没有的。

官方是没有提供对应的文章，所以研究一下kylin的jdbc相关的代码。


我们有一个保底的方案： kylin的rest接口。

参考：
- http://kylin.apache.org/docs/howto/howto_use_restapi.html
- https://zhuanlan.zhihu.com/p/30613434

