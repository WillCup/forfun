
title: cloudnative
date: 2017-09-15 09:30:06
tags: [youdaonote]
---

sdf|特点|说明
---|---|---
pet|1.物理机|dsf
cattle | 



pet

---
 - 物理机 或 虚拟机
 - OS + app + data
 - stateful : LB后的server对state有粘性sticky【1对1】
 - server对OS环境的依赖性极大，迁移扩展困难
 - 


cattle
---
 - 虚拟技术(container) + manifest【记录环境部署的一切】，迁移或扩展方便
 - state存储于第三方，对于所有app共享【数据、server启动的配置信息、manifest信息】
 - puppet等devops工具栈
 - app与OS解耦？
 - 聚焦于app + data，不关心OS等环境
 


参考： https://www.youtube.com/watch?v=zWgq6sd1Ols
