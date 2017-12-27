
title: 一致性协议zab与raft
date: 2016-12-09 14:56:21
tags: [youdaonote]
---

zookeeper的zab1.0协议
raft协议
重视看来一下zookeeper的选举过程，并不是原来说的大家都排队拿一个号，谁小谁当。而是依次比较epoch(选举轮数)、zxid(处理的事务id)、serverid(my.ini里的server.id)，可能最后的server.id是原来那样说的原因吧。
zookeeper的实现是通过一个MessageSender和一个MessageReceiver来实现


raft算法中，follower跟leader不一致的地方会被leader覆盖掉。zookeeper同样也是这样，TRUNC类型的message的处理。

raft不同于zookeeper的地方：
选举的时候serverid的地方并不一样，是通过随机选举超时时间，防止瓜分选票

有一个难点，假设一共5个server，如果某个leader在将所有的logid/zxid同步到了大多数机器，也就是3台，term或者epoch为2。此时挂掉，选举了一个新的leader，它对于这三台机器中的新的log/zxid的处理方式。
raft会判断是否多数已经同步，如果是，就接受并发送出去，同步给其他server。zookeeper并没有提及 —— TODO 源码确认

####
参考：
- https://cwiki.apache.org/confluence/display/ZOOKEEPER/Zab1.0
- http://www.tuicool.com/articles/IfQR3u3
- https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md
-http://www.cnblogs.com/yuyijq/p/4116365.html
-http://iwinit.iteye.com/blog/1773531
