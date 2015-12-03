title: mac使用杂记
date: 2015-12-04 12:00:09
tags: [mac]
---
### 1.查看磁盘空间
开始 -> 关于本机 -> 存储
![](/imgs/mac/磁盘空间)
### 2.快捷键 
|*键*|*描述* |
|---|:---|
|空格|可以查看各种媒体文件|
|command + option + esc|强杀进程|
|F11| 一键显示桌面 |
### 3. 多jdk环境
```
➜  ~  cat ~/.bashrc
export NVM_DIR=~/.nvm
source $(brew --prefix nvm)/nvm.sh

export JAVA_6_HOME=/Library/Java/JavaVirtualMachines/1.6.0.jdk/Contents/Home
export JAVA_7_1_HOME=/Library/Java/JavaVirtualMachines/jdk1.7.0_07.jdk/Contents/Home
export JAVA_7_HOME=/Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home
export JAVA_HOME=$JAVA_7_HOME

# 下面的命令用来切换当前jdk环境
alias jdk6='export JAVA_HOME=$JAVA_6_HOME && export PATH=$JAVA_HOME/bin:$PATH'
alias jdk71='export JAVA_HOME=$JAVA_7_1_HOME && export PATH=$JAVA_HOME/bin:$PATH'
alias jdk72='export JAVA_HOME=$JAVA_7_2_HOME && export PATH=$JAVA_HOME/bin:$PATH'

➜  ~  java -version
java version "1.7.0_80"
Java(TM) SE Runtime Environment (build 1.7.0_80-b15)
Java HotSpot(TM) 64-Bit Server VM (build 24.80-b11, mixed mode)
➜  ~  jdk6
➜  ~  java -version
java version "1.6.0_65"
Java(TM) SE Runtime Environment (build 1.6.0_65-b14-468-11M4833)
Java HotSpot(TM) 64-Bit Server VM (build 20.65-b04-468, mixed mode)
```
参考自：http://ningandjiao.iteye.com/blog/2045955
