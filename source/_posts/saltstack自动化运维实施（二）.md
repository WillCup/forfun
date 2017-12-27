
title: saltstack自动化运维实施（二）
date: 2016-12-30 19:02:36
tags: [youdaonote]
---

我们安装saltstack的痛点在于文件同步，与分布式命令执行。

这一章我们弄一下文件同步的事情.
编辑master的配置文件
```
file_recv: True
```
重启master。


从一个minion向master推送一个文件：
```

```

参考： 
- http://docs.saltstack.cn/ref/file_server/all/index.html
- http://docs.saltstack.cn/ref/file_server/all/salt.fileserver.minionfs.html#module-salt.fileserver.minionfs
