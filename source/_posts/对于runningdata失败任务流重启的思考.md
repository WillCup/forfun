
title: 对于runningdata失败任务流重启的思考
date: 2017-03-09 10:09:41
tags: [youdaonote]
---

对于单个ETL的重导。可以在ETL里自己写IF语句，然后配置调度的时候.


重导的方案A：
---
```
files=$1
other_files=$2
sh $other_files
if [ $? -eq 0 ]; then
     echo "echo $other_files done" > $file
fi
```
然后在azkaban执行这个就可以了。
other_files里是我们实际要执行的命令，$file是我们的azkaban执行的command.
一个更简便的方案，就是把azkaban的命令写成这样，省去自己判断上个命令推出状态：
`ipconfig && echo xxxxxxxxxxxxxx > test.sh`
如果第一个命令出问题，那么肯定就不会执行第二个了。
这样就可以执行日常任务的时候进行任务恢复了，只要是某个任务出现功能性失败，那么就会报错，而且不影响要执行的命令脚本；而成功的任务则会在执行完成后，修改执行的命令脚本，只是echo一下这个任务已经执行过了就好了。
然后每次要执行失败任务恢复的时候，就直接重新执行整个flow，前面成功执行过的就会自动pass，输出已经执行过了。

还有一个问题，就是如果是代码问题，例如sql问题，必须通过修改ETL里的sql或者配置、或者presql等来解决的话，应该怎样处理？

最简单的支持方式是：直接提供一个页面给管理员，编辑azkaban执行的任务文件，再进行重调。

现在我们azkaban的任务调度形式：
```
hive -f /var/azkaban-metamap/h2h-20170308050120/fact_jlc@redeem_record.hql
```

对应修改成的就是：
```
hive -f /var/azkaban-metamap/h2h-20170308050120/fact_jlc@redeem_record.hql && echo "select '/var/azkaban-metamap/h2h-20170308050120/fact_jlc@redeem_record.hql' done"
```

如果出现问题，需要从这个地方开始恢复数据的话，走以下流程：
 - a. 编辑修改ETL
 - b. 测试新ETL是否无误
 - c. review hql，如需要修改个别地方手动修改
 - d. 打开编辑/var/azkaban-metamap/h2h-20170308050120/fact_jlc@redeem_record.hql的web页面，将c步骤中获取的内容放进来
 - e. 到azkaban重新执行今天的任务
 
期望出现的现象：
 - a. 前面已经执行过的任务，自动输出已执行并跳过
 - b. 失败任务继续执行，读取的/var/azkaban-metamap/h2h-20170308050120/fact_jlc@redeem_record.hql是已经修改了的没有问题的文件


额外提示：
 - a. 如果使用此种失败重启的方式，那么所有依赖关系都会被认为是**强依赖**。假设一个极端情况，就是一个大宽表依赖10个小表，并服务于8个应用，如果只有一个小表失败，就得使得整个宽表都不能跑。有可能这个小表只服务于一个小业务模块，但是其他几个就很重要。这样影响面比较大。
 - b. 如果使用此种重启方式，使用弱依赖，也就是当前表的失败并不影响下游任务继续执行的话，那么重新调度的时下游就不能执行了，因为它可能已经echo自己是成功的了。变态支持此方案的话，需要在d步骤的时候，添加一个hook，遍历所有没有成功的ETL的所有子节点，然后把这些子节点恢复为可执行状态。
 
 
方案B
---
指定ETL，生成这个ETL及其所有相关字节点的任务，上传到azkaban进行调度。

**问题**：
如果多个ETL失败，交叉的子节点会重复执行。

**解决方案**：
一次接收多个ETL为参数，放进set里，再去生成相应任务
