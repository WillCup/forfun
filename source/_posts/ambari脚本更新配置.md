
title: ambari脚本更新配置
date: 2017-03-31 10:45:56
tags: [youdaonote]
---

类似于`hdfs_log_dir_prefix`这种配置项一旦在服务初始化完成之后，就不能在UI界面上进行修改了。具体ambari为什么要这样限制，个人表示无从知晓。

通过文件`/var/lib/ambari-agent/cache/cluster_configuration/configurations.json`，我们可以直接看到所有的环境相关配置。暂时猜测可能是为了统一一次性生成此文件，而所有UI的操作都是修改的各自的对应的xml配置文件。

官方提供了两种方式修改对于此json文件相关的配置。一个是API，另一个就是通过`/var/lib/ambari-server/resources/scripts/config.sh`脚本。


个人比较倾向于脚本，更不易出错。查看config.sh的使用方法：
```
/var/lib/ambari-server/resources/scripts/configs.py --help
Usage: configs.py [options]
Options:
  -h, --help            show this help message and exit
  -t PORT, --port=PORT  Optional port number for Ambari server. Default is
                        '8080'. Provide empty string to not use port.
  -s PROTOCOL, --protocol=PROTOCOL
                        Optional support of SSL. Default protocol is 'http'
  -a ACTION, --action=ACTION
                        Script action: <get>, <set>, <delete>
  -l HOST, --host=HOST  Server external host name
  -n CLUSTER, --cluster=CLUSTER
                        Name given to cluster. Ex: 'c1'
  -c CONFIG_TYPE, --config-type=CONFIG_TYPE
                        One of the various configuration types in Ambari. Ex:
                        core-site, hdfs-site, mapred-queue-acls, etc.
  To specify credentials please use "-e" OR "-u" and "-p'":
    -u USER, --user=USER
                        Optional user ID to use for authentication. Default is
                        'admin'
    -p PASSWORD, --password=PASSWORD
                        Optional password to use for authentication. Default
                        is 'admin'
    -e CREDENTIALS_FILE, --credentials-file=CREDENTIALS_FILE
                        Optional file with user credentials separated by new
                        line.
  To specify property(s) please use "-f" OR "-k" and "-v'":
    -f FILE, --file=FILE
                        File where entire configurations are saved to, or read
                        from. Supported extensions (.xml, .json>)
    -k KEY, --key=KEY   Key that has to be set or deleted. Not necessary for
                        'get' action.
    -v VALUE, --value=VALUE
                        Optional value to be set. Not necessary for 'get' or
                        'delete' actions.
```

首先，我们可以先用api查看一下配置项的分布：
```
[hdfs@data-test01 scripts]$ curl -k -s -u admin:admin "http://10.1.5.79:8080/api/v1/clusters/willcluster?fields=Clusters/desired_configs"
{
  "href" : "http://10.1.5.79:8080/api/v1/clusters/willcluster?fields=Clusters/desired_configs",
  "Clusters" : {
    "cluster_name" : "willcluster",
    "version" : "HDP-2.4",
    "desired_configs" : {
      "capacity-scheduler" : {
        "tag" : "version1490254923891",
        "user" : "admin",
        "version" : 2
      },
      "cluster-env" : {
        "tag" : "version1490254925199",
        "user" : "admin",
        "version" : 2
      },
      "core-site" : {
        "tag" : "version1490345767824",
        "user" : "admin",
        "version" : 5
      },
      "hadoop-env" : {
        "tag" : "version1490254923436",
        "user" : "admin",
        "version" : 2
      },
      "hadoop-policy" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "hdfs-log4j" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "hdfs-site" : {
        "tag" : "version1490344980013",
        "user" : "admin",
        "version" : 4
      },
      "kerberos-env" : {
        "tag" : "version1490254524258",
        "user" : "admin",
        "version" : 2
      },
      "krb5-conf" : {
        "tag" : "version1490254524258",
        "user" : "admin",
        "version" : 2
      },
      "mapred-env" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "mapred-site" : {
        "tag" : "version1490254925210",
        "user" : "admin",
        "version" : 2
      },
      "ranger-hdfs-audit" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "ranger-hdfs-plugin-properties" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "ranger-hdfs-policymgr-ssl" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "ranger-hdfs-security" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "ranger-yarn-audit" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "ranger-yarn-plugin-properties" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "ranger-yarn-policymgr-ssl" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "ranger-yarn-security" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "ssl-client" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "ssl-server" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "yarn-env" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "yarn-log4j" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "yarn-site" : {
        "tag" : "version1490254924330",
        "user" : "admin",
        "version" : 6
      },
      "zoo.cfg" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      },
      "zookeeper-env" : {
        "tag" : "version1490254924765",
        "user" : "admin",
        "version" : 2
      },
      "zookeeper-log4j" : {
        "tag" : "version1",
        "user" : "admin",
        "version" : 1
      }
    }
  }
}
```

上面的key就对应于`config.sh` set的一些可选项,也就是CONFIG_TYPE。


下面列出一些常用的样例。按照amabri的惯例，每个log目录都应该是由对应的user创建并修改的。这里我们的所有机器都只有/server目录，下面并没有添加或配置log目录，试一下能不能像初始化环境的时候自动配置对应权限。一试成功了。真好，哈哈
```
./configs.sh  set 10.1.5.79 willcluster hadoop-env hdfs_log_dir_prefix "/server/log/hadoop"
./configs.sh  set 10.1.5.79 willcluster mapred-env mapred_log_dir_prefix "/server/log/hadoop-mapred"
./configs.sh  set 10.1.5.79 willcluster yarn-env yarn_log_dir_prefix "/server/log/yarn"
./configs.sh  set 10.1.5.79 willcluster zookeeper-env zk_log_dir "/server/log/zookeeper"
```

刷新amabri的UI的配置页就可以看到，对应配置已经修改成功，等待重启生效了。

然后tail一下各个服务对应的日志，都没有问题。

参考：
- https://cwiki.apache.org/confluence/display/AMBARI/Modify+configurations
