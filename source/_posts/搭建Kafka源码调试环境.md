title: 搭建Kafka源码调试环境
date: 2015-07-09 00:14:06
tags:
---
# 简单流程

1. git clone https://github.com/apache/kafka.git
2. git tag -l 瞅准0.8.2.1
3. git checkout 0.8.2.1 -b will0821
4. http://gradle.org/下载安装gradle，也为自己的IDE安装gradle插件【对gradle不是特别了解，所以多用插件】。我下载的是gradle2.5，解压后需要在IDE里指定GRADLE_HOME变量。
Eclipse是通过gradle EnIDE的配置窗口指定的。其他请自行解决。
![Figure  - eclipse截图](/imgs/eclipse_ide_gradle.png)
5. build，尝试运行

***
# 错误处理

* 我用的是eclipse，报错**“找不到或无法加载主类”**，按照这个思路去Google，没有找到实际能解决的办法。此时注意到eclipse的下方Maker里有一条错误提示**“scalatest_2.10-1.9.1.jar of core build path is cross-compiled with an incompatible version of Scala (2.10.0)”**，找到官方JIRA提供的解决方案 https://issues.apache.org/jira/browse/KAFKA-1873。将kafka下的**gradle.properties**里scalaVersion从2.10.4修改到2.11.5即可。猜测这个文件应该是类似于maven里的build配置项。
重新build一把，再次执行入口类Kafka即可。

