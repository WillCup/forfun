
title: Yelp开源的pass平台工具
date: 2017-09-08 14:48:13
tags: [youdaonote]
---

1. git push到git仓库
2. jenkins pipeline - jenkins执行git pull 
3. jenkins pipeline - jenkins执行单元测试
3. jenkins pipeline - 通过测试后添加git tag: passta-cluseter1.servicename-20160505T090305-deploy、passta-cluseter1.servicename-20160505T090305-start
4. jenkins pipeline - docker build && 推送到仓库
5. jenkins pipeline - marathon拉取指定tag的commit id，开始运行service服务。
6. 使用smartstack做服务发现工作
7. sensu进行监控工作


- https://github.com/Yelp/paasta
- https://www.youtube.com/watch?v=vISUXKeoqXM&feature=youtu.be
