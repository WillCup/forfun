
title: ubuntu磁盘相关
date: 2017-11-08 14:40:17
tags: [youdaonote]
---

主要是自己工作用的ubuntu虚拟机因为最开始分配的磁盘不够，需要从外面额外扩展一下磁盘。

过程中，小小记录一下一些点，下次处理快一些。


#### 图形化磁盘分析工具

baobab，这个工具是ubuntu16里已经默认有了的。其实就类似du --max-depth=1 -h /，然后不用自己一层层去找所有目录的大小。它是一下子帮我们分析出所有的目录空间，当然时间就会长一些。相比命令行，优点当然就是查看与比较起来更加方便，快速定位到大磁盘，审慎地处理一些没用的数据。


![](https://dn-linuxcn.qbox.me/data/attachment/album/201404/24/151605g7xh8uhk7a8b5suu.png)

当然，因为要统计所有目录空间大小，再呈现出来，所以速度会慢一些。


####  图形化磁盘分区工具

通过vmware给虚拟机扩展磁盘到80以后，执行fdisk -l，发现并没有出现新的40G空间。

gparted, 这个需要额外通过apt进行安装。可以方便地完成磁盘格式化等操作，自己敲命令进行分区与格式化，会稍微有些不自信。


![](http://1833.img.pp.sohu.com.cn/images/blog/2008/9/18/16/2/11d1b7819a1g213.jpg)


方案确认之后，点击上面的对勾，开始执行所有的变更操作。之后`fdisk -l`就可以看到新的空间了。



#### 挂载磁盘

一次性挂载，就使用mount命令就可以了。


永久的话，就需要编辑/etc/fstab了。但是在此之前，我们要通过`blkid`命令先获取到要挂载磁盘的UUID,，然后再添加下面内容。

```
UUID=e23a1c1e-8d91-4df8-8fba-f0656a1080ab	/will_data	ext4	defaults,errors=remount-ro	0	1
```


参考：
- https://gist.github.com/gaoyifan/019ad7766f030ab5be50
- http://www.jianshu.com/p/ec5579ef15a6



#### 后记
妈蛋，后来发现是自己的Trash一直没有清空过....有5G的数据。清空后空间够了.....
