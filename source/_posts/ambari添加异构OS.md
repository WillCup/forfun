
title: ambari添加异构OS
date: 2017-10-26 15:35:13
tags: [youdaonote]
---



首先声明，HDFS集群中肯定是最好不要有异构机器存在的，会造成很多兼容性问题。

背景
---

系统 | 作用 | 数量
---| --- |---
centos6 | yarn/HDFS等服务的选定集群，主要用来运行日常任务 | 50
centos7 | marathon/docker所在的小集群，预想用来调度任务 | 5


其实我们已经基于centos6.8 final有了一个比较稳定的集群，上面通过ambari安装并稳健运行着HDFS、MR、Yarn、Spark、HBase等大数据服务。但是我们想要将spark streaming、hive等ETL工作的客户端放置在mesos上运行，因为当这些工作比较多的时候，也同样需要资源管理工作。鉴于mesos的资源管理相对yarn来说更加可定制化、而且已经有很多开源且稳定的framework可用，我们就选用mesos来做这个客户端集群的工作。对于long time的spark streaming，我们期望使用marathon来管理。一段时间的hive ETL任务，可以使用docker或者chronos来执行。

鉴于以上计划，我们需要在mesos的agent机器上安装spark、hive等客户端配置文件。考虑到易操作性和软件的版本统一兼容问题，决定继续使用ambari在mesos agent上安装这些软件的client端，然后就可以直接调用了。


问题
---

在添加新host的过程中，ambari会比较智能的告诉我们，新的OS与既有的集群OS可能存在不兼容的问题，并阻止我们进一步执行。


处理
---
考虑到我们只是使用客户端，而且我们的程序客户端是基于java的，具有可移植性。所以我就去找到ambari-server在新增机器的时候，对于OS的检测脚本，暂时注释掉，添加完新的host之后，再放开。

2.2.2.0的脚本位置:`/usr/lib/python2.6/site-packages/ambari_server/bootstrap.py`：
```
def run(self):
    """ Copy files and run commands on remote host """
    self.status["start_time"] = time.time()
    # Population of action queue
    action_queue = [self.createTargetDir,
                    self.copyCommonFunctions,
                    self.copyOsCheckScript,
                    self.runOsCheckScript,
                    self.checkSudoPackage
    ]
    if self.hasPassword():
      action_queue.extend([self.copyPasswordFile,
                           self.changePasswordFileModeOnHost])
    action_queue.extend([
      self.copyNeededFiles,
      self.runSetupAgent,
    ])
    ....

```

把action_queue里的OSCheck相关的暂时注释掉就可以了。然后去添加host的界面上点击retry failed。只要注册成功就可以了。


install过程跳过去了，但是注册的时候出现了问题。

报错：
```
INFO 2017-10-26 16:14:42,747 NetUtil.py:60 - Connecting to https://namenode01.will.com:8440/ca
ERROR 2017-10-26 16:14:42,848 NetUtil.py:84 - [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed (_ssl.c:579)
ERROR 2017-10-26 16:14:42,848 NetUtil.py:85 - SSLError: Failed to connect. Please check openssl library versions. 
Refer to: https://bugzilla.redhat.com/show_bug.cgi?id=1022468 for more details.
WARNING 2017-10-26 16:14:42,850 NetUtil.py:112 - Server at https://namenode01.will.com:8440 is not reachable, sleeping for 10 seconds...
', None)
```

在新的host上找到ambari-agent相关代码：
```
  def checkURL(self, url):
    """Try to connect to a given url. Result is True if url returns HTTP code 200, in any other case
    (like unreachable server or wrong HTTP code) result will be False.

       Additionally returns body of request, if available
    """
    logger.info("Connecting to " + url)
    responseBody = ""

    try:
      parsedurl = urlparse(url)

      if sys.version_info >= (2,7,9):
          import ssl
          ca_connection = httplib.HTTPSConnection(parsedurl[1], context=ssl._create_unverified_context())
      else:
          ca_connection = httplib.HTTPSConnection(parsedurl[1])
      ...
    except SSLError as slerror:
      logger.error(str(slerror))
      logger.error(ERROR_SSL_WRONG_VERSION)
      return False, responseBody

    except Exception, e:
      logger.warning("Failed to connect to " + str(url) + " due to " + str(e) + "  ")
      return False, responseBody
```

从报错来看，是出现了SSLError，好像是SSL的兼容问题。对比看一下ssl的版本：
- centos7 openssl-1.0.2k-8.el7.x86_64
- centos6 openssl-1.0.1e-57.el6.x86_64


考虑到既有集群的稳定性，失败告终。配置同步问题转由rsync、nfs之类的方案解决吧。







参考：
- https://community.hortonworks.com/questions/4324/hdp-support-for-mix-of-os-releases-within-a-cluste.html
- https://community.hortonworks.com/questions/18479/how-to-register-host-with-different-os-to-ambari.html
