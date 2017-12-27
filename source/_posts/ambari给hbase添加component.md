
title: ambari给hbase添加component
date: 2017-04-05 14:53:26
tags: [youdaonote]
---

因为原生的ambari并没有原生支持habse的thriftserver在UI端操作。而且业务部门需要使用这个，我们运维的过程中，总是需要去到指定机器启动才行，不太方便，总会遗漏。所以添加到ambari的hbase service里。

ps:
按照参考的位置`/var/lib/ambari-server/resources/stacks/HDP/2.4/services/HBASE/metainfo.xml `里面空空如也。

添加component
---
编辑/var/lib/ambari-server/resources/common-services/HBASE/0.96.0.2.0/metainfo.xml文件，添加对应component
```xml
        <component>
          <name>HBASE_THRIFT_SERVER</name>
          <displayName>ThriftServer</displayName>
          <category>MASTER</category>
          <cardinality>1+</cardinality>
          <versionAdvertised>true</versionAdvertised>
          <timelineAppid>HBASE</timelineAppid>
          <commandScript>
            <script>scripts/hbase_thrift_server.py</script>
            <scriptType>PYTHON</scriptType>
          </commandScript>
      </component>

```

添加对应脚本
---
`/var/lib/ambari-server/resources/common-services/HBASE/0.96.0.2.0/package/scripts/hbase_thrift_server.py`:
```py
import sys
from resource_management import *
from resource_management.libraries.functions.security_commons import build_expectations, \
  cached_kinit_executor, get_params_from_filesystem, validate_security_config_properties, \
  FILE_TYPE_XML
from hbase import hbase
from hbase_service import hbase_service
import upgrade
from ambari_commons import OSCheck, OSConst
from ambari_commons.os_family_impl import OsFamilyImpl


class HbaseThriftServer(Script):
  def install(self, env):
   # import params
   # self.install_packages(env, params.exclude_packages)
    print "Decommission not yet implemented!"
   
  def configure(self, env):
   # import params
   # env.set_params(params)
   # hbase(name='thrift')
    print "Decommission not yet implemented!"

  def decommission(self, env):
    print "Decommission not yet implemented!"



@OsFamilyImpl(os_family=OsFamilyImpl.DEFAULT)
class HbaseThriftServerDefault(HbaseThriftServer):
  def get_stack_to_component(self):
    return {"HDP": "hbase-thrift"}

  def pre_upgrade_restart(self, env, upgrade_type=None):
    import params
    env.set_params(params)
    upgrade.prestart(env, "hbase-thrift")

  def post_upgrade_restart(self, env, upgrade_type=None):
    import params
    env.set_params(params)
    upgrade.post_thrift(env)

  def start(self, env, upgrade_type=None):
    import params
    env.set_params(params)
    self.configure(env) # for security
    hbase_service( 'thrift',
      action = 'start'
    )

  def stop(self, env, upgrade_type=None):
    import params
    env.set_params(params)

    hbase_service( 'thrift',
      action = 'stop'
    )

  def status(self, env):
    import status_params
    env.set_params(status_params)
    pid_file = format("{pid_dir}/hbase-{hbase_user}-thrift.pid")
    check_process_status(pid_file)

  def security_status(self, env):
    import status_params

    env.set_params(status_params)
    if status_params.security_enabled:
      props_value_check = {"hbase.security.authentication" : "kerberos",
                           "hbase.security.authorization": "true"}
      props_empty_check = ['hbase.thrift.keytab.file',
                           'hbase.thrift.kerberos.principal']
      props_read_check = ['hbase.thrift.keytab.file']
      hbase_site_expectations = build_expectations('hbase-site', props_value_check, props_empty_check,
                                                   props_read_check)

      hbase_expectations = {}
      hbase_expectations.update(hbase_site_expectations)

      security_params = get_params_from_filesystem(status_params.hbase_conf_dir,
                                                   {'hbase-site.xml': FILE_TYPE_XML})
      result_issues = validate_security_config_properties(security_params, hbase_expectations)
      if not result_issues:  # If all validations passed successfully
        try:
          # Double check the dict before calling execute
          if ( 'hbase-site' not in security_params
               or 'hbase.thrift.keytab.file' not in security_params['hbase-site']
               or 'hbase.thrift.kerberos.principal' not in security_params['hbase-site']):
            self.put_structured_out({"securityState": "UNSECURED"})
            self.put_structured_out(
              {"securityIssuesFound": "Keytab file or principal are not set property."})
            return

          cached_kinit_executor(status_params.kinit_path_local,
                                status_params.hbase_user,
                                security_params['hbase-site']['hbase.thrift.keytab.file'],
                                security_params['hbase-site']['hbase.thrift.kerberos.principal'],
                                status_params.hostname,
                                status_params.tmp_dir)
          self.put_structured_out({"securityState": "SECURED_KERBEROS"})
        except Exception as e:
          self.put_structured_out({"securityState": "ERROR"})
          self.put_structured_out({"securityStateErrorInfo": str(e)})
      else:
        issues = []
        for cf in result_issues:
          issues.append("Configuration file %s did not pass the validation. Reason: %s" % (cf, result_issues[cf]))
        self.put_structured_out({"securityIssuesFound": ". ".join(issues)})
        self.put_structured_out({"securityState": "UNSECURED"})
    else:
      self.put_structured_out({"securityState": "UNSECURED"})


if __name__ == "__main__":
  HbaseThriftServer().execute()
```

其实是copy的regionserver的脚本，因为install什么的都不用，所以就全设置为print了。


重启ambari-server
---

添加hbase服务
---


插曲
---
install的时候出现错误，启动的时候也会这样。。。看来是应该先注册一下自己的名字：
```
2017-04-05 09:32:30,375 - Could not determine HDP version for component hbase-thrift by calling '/usr/bin/hdp-select status hbase-thrift > /tmp/tmpa6nhtU'. Return Code: 1, Output: ERROR: Invalid package - hbase-thrift

Packages:
  accumulo-client
  accumulo-gc
  accumulo-master
  accumulo-monitor
  accumulo-tablet
  accumulo-tracer
  atlas-server
  falcon-client
  falcon-server
  flume-server
  hadoop-client
  hadoop-hdfs-datanode
  hadoop-hdfs-journalnode
  hadoop-hdfs-namenode
  hadoop-hdfs-nfs3
  hadoop-hdfs-portmap
  hadoop-hdfs-secondarynamenode
  hadoop-httpfs
  hadoop-mapreduce-historyserver
  hadoop-yarn-nodemanager
  hadoop-yarn-resourcemanager
  hadoop-yarn-timelineserver
  hbase-client
  hbase-master
  hbase-regionserver
  hive-metastore
  hive-server2
  hive-webhcat
  kafka-broker
  knox-server
  livy-server
  mahout-client
  oozie-client
  oozie-server
  phoenix-client
  phoenix-server
  ranger-admin
  ranger-kms
  ranger-usersync
  slider-client
  spark-client
  spark-historyserver
  spark-thriftserver
  sqoop-client
  sqoop-server
  storm-client
  storm-nimbus
  storm-slider-client
  storm-supervisor
  zeppelin-server
  zookeeper-client
  zookeeper-server
Aliases:
  accumulo-server
  all
  client
  hadoop-hdfs-server
  hadoop-mapreduce-server
  hadoop-yarn-server
  hive-server
.
```

找了老半天，问题竟然在shell里，服了。
在/usr/bin/hdp-select里的leaves字典里添加
```
 "hbase-thrift": "hbase",
```
然后还要创建一个目录才行
```
ln -s /usr/hdp/2.4.2.0-258/hbase-regionserver /usr/hdp/current/hbase-thrift
```

参考：

- https://cwiki.apache.org/confluence/display/AMBARI/Defining+a+Custom+Stack+and+Services
- 
