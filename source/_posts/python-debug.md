title: python debug
date: 2016-06-23 19:24:52
tags: [python]
---

今儿想着快些让hive view的java代码跟远程程序能够一起调试了，想要远程调试就得在远程ambari server启动的地方添加jdwp设置才行。

当我们安装好ambari-server之后，它默认是在/usr/sbin/ambari-server的一个shell文件，start命令导流到/usr/sbin/ambari-server.py脚本中。跟tomcat、jetty啥的不一样，并不提供静态的类似JAVA_OPTS之类的变量。ambari server是作为python脚本的子进程启动的，所以启动的参数都是通过python程序生成后执行的。自己沿着入口找了半天，越找越蒙圈，回头想了一下，还是debug来的更真实。

查下python的debug方式，最快能实施的就是使用**pdb**了。到ambari server的宿主机上开始进行调试。

### pdb基础

##### 设置断点
在要调试的python文件import pdb，然后在断点代码的上方添加pdb.set_trace()
```python
import pdb
....

  options.exit_message = "Ambari Server '%s' completed successfully." % action
  options.exit_code = None


  pdb.set_trace()
  try:
    action_obj.execute()

    if action_obj.need_restart:
      pstatus, pid = is_server_runing()
      if pstatus:
        print 'NOTE: Restart Ambari Server to apply changes' + \
              ' ("ambari-server restart|stop+start")'

```
到命令行执行此python文件，到pdb.set_trace()的位置就会停下，跳出(pdb) 

##### pdb命令
命令|解释
---|---
help | 显示所有命令
n | 继续执行
step | 进入当前方法的方法块
c | 忽略调试，继续执行
p | p str 打印某个对象

我就用了以上命令，完成了对于ambari的调试。如果有其他的要求，可以逐个试一下。

#### ambari调试过程

##### 设置断点 
 在ambari-server.py 引入pdb，在main方法上面添加pdb.set_trace()

```python

...

def mainBody():
  parser = optparse.OptionParser(usage="usage: %prog [options] action [stack_id os]",)
  init_parser_options(parser)
  (options, args) = parser.parse_args()

  pdb.set_trace()

  # check if only silent key set
  default_options = parser.get_default_values()
  silent_options = default_options
  silent_options.silent = True

  if options == silent_options:
    options.only_silent = True
  else:
    options.only_silent = False

  # set verbose
  set_verbose(options.verbose)
  if options.verbose:
    main(options, args, parser)

...


```
##### 执行命令
```python
[root@data-test01 ~]# ambari-server start
Using python  /usr/bin/python2
Starting ambari-server
> /usr/sbin/ambari-server.py(584)main()
-> try:
(Pdb) n
> /usr/sbin/ambari-server.py(585)main()
-> action_obj.execute()
(Pdb) step
--Call--
> /usr/sbin/ambari-server.py(59)execute()
-> def execute(self):
(Pdb) n
> /usr/sbin/ambari-server.py(60)execute()
-> self.fn(*self.args, **self.kwargs)
(Pdb) p *self.args
*** SyntaxError: SyntaxError('invalid syntax', ('<string>', 1, 1, '*self.args'))
(Pdb) p self.args
(<Values at 0xfbee60: {'only_silent': False, 'jdbc_driver': None, 'verbose': False, 'init_script_file': '/var/lib/ambari-server/resources/Ambari-DDL-Postgres-EMBEDDED-CREATE.sql', 'dbms': None, 'silent': False, 'warnings': [], 'exit_code': None, 'cluster_name': None, 'database_username': None, 'drop_script_file': '/var/lib/ambari-server/resources/Ambari-DDL-Postgres-EMBEDDED-DROP.sql', 'upgrade_script_file': '/var/lib/ambari-server/resources/upgrade/ddl/Ambari-DDL-Postgres-UPGRADE-1.3.0.sql', 'java_home': None, 'ldap_sync_existing': False, 'force_repo_version': False, 'debug': False, 'ldap_sync_groups': None, 'must_set_database_options': True, 'sqla_server_name': None, 'ldap_sync_users': None, 'database_name': None, 'jdbc_db': None, 'ldap_sync_all': False, 'upgrade_stack_script_file': '/var/lib/ambari-server/resources/upgrade/dml/Ambari-DML-Postgres-UPGRADE_STACK.sql', 'desired_repo_version': None, 'suspend_start': False, 'database_port': None, 'sid_or_sname': 'sname', 'database_password': None, 'exit_message': "Ambari Server 'start' completed successfully.", 'database_host': None, 'postgres_schema': None}>,)
(Pdb) p self.kwargs
{}
(Pdb) step
--Call--
> /usr/lib/python2.6/site-packages/ambari_commons/os_family_impl.py(82)thunk()
-> def thunk(*args, **kwargs):
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_commons/os_family_impl.py(83)thunk()
-> fn_id_base = func.__module__ + "." + func.__name__
(Pdb) jn
*** NameError: name 'jn' is not defined
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_commons/os_family_impl.py(84)thunk()
-> fn_id = fn_id_base + "." + OSCheck.get_os_family()
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_commons/os_family_impl.py(85)thunk()
-> if fn_id not in self._func_impls:
(Pdb) p fn_id
'__main__.start.redhat'
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_commons/os_family_impl.py(86)thunk()
-> fn_id = fn_id_base + "." + OsFamilyImpl.DEFAULT
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_commons/os_family_impl.py(88)thunk()
-> fn = self._func_impls[fn_id]
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_commons/os_family_impl.py(89)thunk()
-> return fn(*args, **kwargs)
(Pdb) p args
(<Values at 0xfbee60: {'only_silent': False, 'jdbc_driver': None, 'verbose': False, 'init_script_file': '/var/lib/ambari-server/resources/Ambari-DDL-Postgres-EMBEDDED-CREATE.sql', 'dbms': None, 'silent': False, 'warnings': [], 'exit_code': None, 'cluster_name': None, 'database_username': None, 'drop_script_file': '/var/lib/ambari-server/resources/Ambari-DDL-Postgres-EMBEDDED-DROP.sql', 'upgrade_script_file': '/var/lib/ambari-server/resources/upgrade/ddl/Ambari-DDL-Postgres-UPGRADE-1.3.0.sql', 'java_home': None, 'ldap_sync_existing': False, 'force_repo_version': False, 'debug': False, 'ldap_sync_groups': None, 'must_set_database_options': True, 'sqla_server_name': None, 'ldap_sync_users': None, 'database_name': None, 'jdbc_db': None, 'ldap_sync_all': False, 'upgrade_stack_script_file': '/var/lib/ambari-server/resources/upgrade/dml/Ambari-DML-Postgres-UPGRADE_STACK.sql', 'desired_repo_version': None, 'suspend_start': False, 'database_port': None, 'sid_or_sname': 'sname', 'database_password': None, 'exit_message': "Ambari Server 'start' completed successfully.", 'database_host': None, 'postgres_schema': None}>,)
(Pdb) p kwargs
{}
(Pdb) step
--Call--
> /usr/sbin/ambari-server.py(103)start()
-> @OsFamilyFuncImpl(OsFamilyImpl.DEFAULT)
(Pdb) n
> /usr/sbin/ambari-server.py(105)start()
-> status, pid = is_server_runing()
(Pdb) n
> /usr/sbin/ambari-server.py(106)start()
-> if status:
(Pdb) p status
False
(Pdb) n
> /usr/sbin/ambari-server.py(110)start()
-> server_process_main(args)
(Pdb) p args
<Values at 0xfbee60: {'only_silent': False, 'jdbc_driver': None, 'verbose': False, 'init_script_file': '/var/lib/ambari-server/resources/Ambari-DDL-Postgres-EMBEDDED-CREATE.sql', 'dbms': None, 'silent': False, 'warnings': [], 'exit_code': None, 'cluster_name': None, 'database_username': None, 'drop_script_file': '/var/lib/ambari-server/resources/Ambari-DDL-Postgres-EMBEDDED-DROP.sql', 'upgrade_script_file': '/var/lib/ambari-server/resources/upgrade/ddl/Ambari-DDL-Postgres-UPGRADE-1.3.0.sql', 'java_home': None, 'ldap_sync_existing': False, 'force_repo_version': False, 'debug': False, 'ldap_sync_groups': None, 'must_set_database_options': True, 'sqla_server_name': None, 'ldap_sync_users': None, 'database_name': None, 'jdbc_db': None, 'ldap_sync_all': False, 'upgrade_stack_script_file': '/var/lib/ambari-server/resources/upgrade/dml/Ambari-DML-Postgres-UPGRADE_STACK.sql', 'desired_repo_version': None, 'suspend_start': False, 'database_port': None, 'sid_or_sname': 'sname', 'database_password': None, 'exit_message': "Ambari Server 'start' completed successfully.", 'database_host': None, 'postgres_schema': None}>
(Pdb) step
--Call--
> /usr/sbin/ambari_server_main.py(204)server_process_main()
-> def server_process_main(options, scmStatus=None):
(Pdb) n
> /usr/sbin/ambari_server_main.py(206)server_process_main()
-> try:
(Pdb) n
> /usr/sbin/ambari_server_main.py(207)server_process_main()
-> set_debug_mode_from_options(options)
(Pdb) n
> /usr/sbin/ambari_server_main.py(211)server_process_main()
-> if not check_reverse_lookup():
(Pdb) n
> /usr/sbin/ambari_server_main.py(216)server_process_main()
-> check_database_name_property()
(Pdb) n
> /usr/sbin/ambari_server_main.py(217)server_process_main()
-> parse_properties_file(options)
(Pdb) p options
<Values at 0xfbee60: {'only_silent': False, 'jdbc_driver': None, 'verbose': False, 'init_script_file': '/var/lib/ambari-server/resources/Ambari-DDL-Postgres-EMBEDDED-CREATE.sql', 'dbms': None, 'silent': False, 'warnings': [], 'exit_code': None, 'cluster_name': None, 'database_username': None, 'drop_script_file': '/var/lib/ambari-server/resources/Ambari-DDL-Postgres-EMBEDDED-DROP.sql', 'upgrade_script_file': '/var/lib/ambari-server/resources/upgrade/ddl/Ambari-DDL-Postgres-UPGRADE-1.3.0.sql', 'java_home': None, 'ldap_sync_existing': False, 'force_repo_version': False, 'debug': False, 'ldap_sync_groups': None, 'must_set_database_options': True, 'sqla_server_name': None, 'ldap_sync_users': None, 'database_name': None, 'jdbc_db': None, 'ldap_sync_all': False, 'upgrade_stack_script_file': '/var/lib/ambari-server/resources/upgrade/dml/Ambari-DML-Postgres-UPGRADE_STACK.sql', 'desired_repo_version': None, 'suspend_start': False, 'database_port': None, 'sid_or_sname': 'sname', 'database_password': None, 'exit_message': "Ambari Server 'start' completed successfully.", 'database_host': None, 'postgres_schema': None}>
(Pdb) n
> /usr/sbin/ambari_server_main.py(219)server_process_main()
-> ambari_user = read_ambari_user()
(Pdb) step
--Call--
> /usr/lib/python2.6/site-packages/ambari_server/serverConfiguration.py(501)read_ambari_user()
-> def read_ambari_user():
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_server/serverConfiguration.py(505)read_ambari_user()
-> properties = get_ambari_properties()
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_server/serverConfiguration.py(506)read_ambari_user()
-> if properties != -1:
(Pdb) p properties
<ambari_server.properties.Properties object at 0xfe4910>
(Pdb) dir(properties)
['_Properties__parse', '__class__', '__delattr__', '__dict__', '__doc__', '__format__', '__getattr__', '__getattribute__', '__getitem__', '__hash__', '__init__', '__module__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', '_keymap', '_origprops', '_props', 'bspacere', 'fileName', 'getPropertyDict', 'get_property', 'load', 'othercharre', 'othercharre2', 'process_pair', 'propertyNames', 'removeOldProp', 'removeProp', 'sort_origprops', 'sort_props', 'store', 'store_ordered', 'unescape']
(Pdb) type(properties)
<class 'ambari_server.properties.Properties'>
(Pdb) p properties fileName
*** SyntaxError: SyntaxError('unexpected EOF while parsing', ('<string>', 1, 19, 'properties fileName'))
(Pdb) p properties.fileName()
*** TypeError: TypeError("'str' object is not callable",)
(Pdb) p properties.fileName
'/etc/ambari-server/conf/ambari.properties'
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_server/serverConfiguration.py(507)read_ambari_user()
-> user = properties[NR_USER_PROPERTY]
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_server/serverConfiguration.py(508)read_ambari_user()
-> if user:
(Pdb) p user
'root'
(Pdb) p
*** SyntaxError: SyntaxError('unexpected EOF while parsing', ('<string>', 0, 0, ''))
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_server/serverConfiguration.py(509)read_ambari_user()
-> return user
(Pdb) n
--Return--
> /usr/lib/python2.6/site-packages/ambari_server/serverConfiguration.py(509)read_ambari_user()->'root'
-> return user
(Pdb) n
> /usr/sbin/ambari_server_main.py(220)server_process_main()
-> current_user = ensure_can_start_under_current_user(ambari_user)
(Pdb) n
> /usr/sbin/ambari_server_main.py(222)server_process_main()
-> print_info_msg("Ambari Server is not running...")
(Pdb) p current_user
'root'
(Pdb) n
> /usr/sbin/ambari_server_main.py(224)server_process_main()
-> jdk_path = find_jdk()
(Pdb) n
> /usr/sbin/ambari_server_main.py(225)server_process_main()
-> if jdk_path is None:
(Pdb) p jdk_path
'/server/java/jdk1.8.0_60'
(Pdb) n
> /usr/sbin/ambari_server_main.py(231)server_process_main()
-> properties = get_ambari_properties()
(Pdb) n
> /usr/sbin/ambari_server_main.py(234)server_process_main()
-> if is_root():
(Pdb) n
> /usr/sbin/ambari_server_main.py(235)server_process_main()
-> print configDefaults.MESSAGE_SERVER_RUNNING_AS_ROOT
(Pdb) print configDefaults.MESSAGE_SERVER_RUNNING_AS_ROOT
Ambari Server running with administrator privileges.
(Pdb) n
Ambari Server running with administrator privileges.
> /usr/sbin/ambari_server_main.py(237)server_process_main()
-> ensure_jdbc_driver_is_installed(options, properties)
(Pdb) n
> /usr/sbin/ambari_server_main.py(239)server_process_main()
-> ensure_dbms_is_running(options, properties, scmStatus)
(Pdb) n
> /usr/sbin/ambari_server_main.py(241)server_process_main()
-> if scmStatus is not None:
(Pdb) p scmStatus
None
(Pdb) n
> /usr/sbin/ambari_server_main.py(244)server_process_main()
-> refresh_stack_hash(properties)
(Pdb) p properties
<ambari_server.properties.Properties object at 0xfe4e50>
(Pdb) step
--Call--
> /usr/lib/python2.6/site-packages/ambari_server/serverUtils.py(85)refresh_stack_hash()
-> def refresh_stack_hash(properties):
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_server/serverUtils.py(86)refresh_stack_hash()
-> resources_location = get_resources_location(properties)
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_server/serverUtils.py(87)refresh_stack_hash()
-> stacks_location = get_stack_location(properties)
(Pdb) p resources_location
'/var/lib/ambari-server/resources'
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_server/serverUtils.py(88)refresh_stack_hash()
-> resource_files_keeper = ResourceFilesKeeper(resources_location, stacks_location)
(Pdb) p stacks_location
'/var/lib/ambari-server/resources/stacks'
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_server/serverUtils.py(90)refresh_stack_hash()
-> try:
(Pdb) p resource_files_keeper
<ambari_server.resourceFilesKeeper.ResourceFilesKeeper instance at 0x103c050>
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_server/serverUtils.py(91)refresh_stack_hash()
-> print "Organizing resource files at {0}...".format(resources_location,
(Pdb) n
> /usr/lib/python2.6/site-packages/ambari_server/serverUtils.py(92)refresh_stack_hash()
-> verbose=get_verbose())
(Pdb) n
Organizing resource files at /var/lib/ambari-server/resources...
> /usr/lib/python2.6/site-packages/ambari_server/serverUtils.py(93)refresh_stack_hash()
-> resource_files_keeper.perform_housekeeping()
(Pdb) p verbose
*** NameError: NameError("name 'verbose' is not defined",)
(Pdb) n
--Return--
> /usr/lib/python2.6/site-packages/ambari_server/serverUtils.py(93)refresh_stack_hash()->None
-> resource_files_keeper.perform_housekeeping()
(Pdb) n
> /usr/sbin/ambari_server_main.py(246)server_process_main()
-> if scmStatus is not None:
(Pdb) n
> /usr/sbin/ambari_server_main.py(249)server_process_main()
-> ensure_server_security_is_configured()
(Pdb) n
> /usr/sbin/ambari_server_main.py(251)server_process_main()
-> if scmStatus is not None:
(Pdb) n
> /usr/sbin/ambari_server_main.py(254)server_process_main()
-> java_exe = get_java_exe_path()
(Pdb) n
> /usr/sbin/ambari_server_main.py(256)server_process_main()
-> serverClassPath = ServerClassPath(properties, options)
(Pdb) n
> /usr/sbin/ambari_server_main.py(258)server_process_main()
-> debug_mode = get_debug_mode()
(Pdb) p serverClassPath
<ambari_server.serverClassPath.ServerClassPath instance at 0x103cfc8>
(Pdb) n
> /usr/sbin/ambari_server_main.py(259)server_process_main()
-> debug_start = (debug_mode & 1) or SERVER_START_DEBUG
(Pdb) n
> /usr/sbin/ambari_server_main.py(260)server_process_main()
-> suspend_start = (debug_mode & 2) or SUSPEND_START_MODE
(Pdb) n
> /usr/sbin/ambari_server_main.py(261)server_process_main()
-> suspend_mode = 'y' if suspend_start else 'n'
(Pdb) p debug_start
False
(Pdb) n
> /usr/sbin/ambari_server_main.py(263)server_process_main()
-> param_list = generate_child_process_param_list(ambari_user, java_exe,
(Pdb) n
> /usr/sbin/ambari_server_main.py(264)server_process_main()
-> serverClassPath.get_full_ambari_classpath_escaped_for_shell(), debug_start,
(Pdb) p param_list
*** NameError: NameError("name 'param_list' is not defined",)
(Pdb) n
> /usr/sbin/ambari_server_main.py(265)server_process_main()
-> suspend_mode)
(Pdb) p param_list
*** NameError: NameError("name 'param_list' is not defined",)
(Pdb) n
> /usr/sbin/ambari_server_main.py(266)server_process_main()
-> environ = generate_env(options, ambari_user, current_user)
(Pdb) p param_list
['/bin/sh', '-c', "ulimit -n 10000 ; /server/java/jdk1.8.0_60/bin/java -server -XX:NewRatio=3 -XX:+UseConcMarkSweepGC -XX:-UseGCOverheadLimit -XX:CMSInitiatingOccupancyFraction=60 -Dsun.zip.disableMemoryMapping=true   -Xms512m -Xmx2048m -Djava.security.auth.login.config=/etc/ambari-server/conf/krb5JAASLogin.conf -Djava.security.krb5.conf=/etc/krb5.conf -Djavax.security.auth.useSubjectCredsOnly=false -cp '/etc/ambari-server/conf:/usr/lib/ambari-server/*:/usr/share/java/mysql-connector-java.jar' org.apache.ambari.server.controller.AmbariServer > /var/log/ambari-server/ambari-server.out 2>&1 || echo $? > /var/run/ambari-server/ambari-server.exitcode &"]
(Pdb) n
> /usr/sbin/ambari_server_main.py(268)server_process_main()
-> if not os.path.exists(configDefaults.PID_DIR):
(Pdb) n
> /usr/sbin/ambari_server_main.py(271)server_process_main()
-> print_info_msg("Running server: " + str(param_list))
(Pdb) n
> /usr/sbin/ambari_server_main.py(272)server_process_main()
-> procJava = subprocess.Popen(param_list, env=environ)
(Pdb) p str(param_list)
'[\'/bin/sh\', \'-c\', "ulimit -n 10000 ; /server/java/jdk1.8.0_60/bin/java -server -XX:NewRatio=3 -XX:+UseConcMarkSweepGC -XX:-UseGCOverheadLimit -XX:CMSInitiatingOccupancyFraction=60 -Dsun.zip.disableMemoryMapping=true   -Xms512m -Xmx2048m -Djava.security.auth.login.config=/etc/ambari-server/conf/krb5JAASLogin.conf -Djava.security.krb5.conf=/etc/krb5.conf -Djavax.security.auth.useSubjectCredsOnly=false -cp \'/etc/ambari-server/conf:/usr/lib/ambari-server/*:/usr/share/java/mysql-connector-java.jar\' org.apache.ambari.server.controller.AmbariServer > /var/log/ambari-server/ambari-server.out 2>&1 || echo $? > /var/run/ambari-server/ambari-server.exitcode &"]'
(Pdb) n
> /usr/sbin/ambari_server_main.py(274)server_process_main()
-> pidJava = procJava.pid
(Pdb) p procJava.pid
2216
(Pdb) n
> /usr/sbin/ambari_server_main.py(275)server_process_main()
-> if pidJava <= 0:
(Pdb) n
> /usr/sbin/ambari_server_main.py(288)server_process_main()
-> try:
(Pdb) n
> /usr/sbin/ambari_server_main.py(289)server_process_main()
-> os.setpgid(pidJava, 0)
(Pdb) n
OSError: (13, 'Permission denied')
> /usr/sbin/ambari_server_main.py(289)server_process_main()
-> os.setpgid(pidJava, 0)
(Pdb) n
> /usr/sbin/ambari_server_main.py(290)server_process_main()
-> except OSError, e:
(Pdb) n
> /usr/sbin/ambari_server_main.py(291)server_process_main()
-> print_warning_msg('setpgid({0}, 0) failed - {1}'.format(pidJava, str(e)))
(Pdb) n
WARNING: setpgid(2216, 0) failed - [Errno 13] Permission denied
> /usr/sbin/ambari_server_main.py(292)server_process_main()
-> pass
(Pdb) n
> /usr/sbin/ambari_server_main.py(293)server_process_main()
-> pidfile = os.path.join(configDefaults.PID_DIR, PID_NAME)
(Pdb) n
> /usr/sbin/ambari_server_main.py(294)server_process_main()
-> save_pid(pidJava, pidfile)
(Pdb) n
> /usr/sbin/ambari_server_main.py(295)server_process_main()
-> print "Server PID at: "+pidfile
(Pdb) n
Server PID at: /var/run/ambari-server/ambari-server.pid
> /usr/sbin/ambari_server_main.py(296)server_process_main()
-> print "Server out at: "+configDefaults.SERVER_OUT_FILE
(Pdb) n
Server out at: /var/log/ambari-server/ambari-server.out
> /usr/sbin/ambari_server_main.py(297)server_process_main()
-> print "Server log at: "+configDefaults.SERVER_LOG_FILE
(Pdb) n
Server log at: /var/log/ambari-server/ambari-server.log
> /usr/sbin/ambari_server_main.py(299)server_process_main()
-> wait_for_server_start(pidfile, scmStatus)
(Pdb) n
Waiting for server start....................
> /usr/sbin/ambari_server_main.py(301)server_process_main()
-> if scmStatus is not None:
(Pdb) n
> /usr/sbin/ambari_server_main.py(304)server_process_main()
-> return procJava
(Pdb) p procJava
<subprocess.Popen object at 0x1038690>
(Pdb) dir(procJava)
['__class__', '__del__', '__delattr__', '__dict__', '__doc__', '__format__', '__getattribute__', '__hash__', '__init__', '__module__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', '_check_timeout', '_child_created', '_close_fds', '_communicate', '_communicate_with_poll', '_communicate_with_select', '_communication_started', '_execute_child', '_get_handles', '_handle_exitstatus', '_input', '_internal_poll', '_remaining_time', '_set_cloexec_flag', '_translate_newlines', 'communicate', 'kill', 'pid', 'poll', 'returncode', 'send_signal', 'stderr', 'stdin', 'stdout', 'terminate', 'universal_newlines', 'wait']
(Pdb) n
--Return--
> /usr/sbin/ambari_server_main.py(304)server_process_main()-><subproc...x1038690>
-> return procJava
(Pdb) n
--Return--
> /usr/sbin/ambari-server.py(110)start()->None
-> server_process_main(args)
(Pdb) n
--Return--
> /usr/lib/python2.6/site-packages/ambari_commons/os_family_impl.py(89)thunk()->None
-> return fn(*args, **kwargs)
(Pdb) n
--Return--
> /usr/sbin/ambari-server.py(60)execute()->None
-> self.fn(*self.args, **self.kwargs)
(Pdb) n
> /usr/sbin/ambari-server.py(587)main()
-> if action_obj.need_restart:
(Pdb) n
> /usr/sbin/ambari-server.py(593)main()
-> if options.warnings:
(Pdb) n
> /usr/sbin/ambari-server.py(608)main()
-> if options.exit_message is not None:
(Pdb) n
> /usr/sbin/ambari-server.py(609)main()
-> print options.exit_message
(Pdb) n
Ambari Server 'start' completed successfully.
> /usr/sbin/ambari-server.py(611)main()
-> if options.exit_code is not None:  # not all actions may return a system exit code
(Pdb) n
--Return--
> /usr/sbin/ambari-server.py(611)main()->None
-> if options.exit_code is not None:  # not all actions may return a system exit code
(Pdb) n
--Return--
> /usr/sbin/ambari-server.py(635)mainBody()->None
-> main(options, args, parser)
(Pdb) n
--Return--
> /usr/sbin/ambari-server.py(644)<module>()->None
-> mainBody()
(Pdb) n
[root@data-test01 ~]# which ambari-server
/usr/sbin/ambari-server
[root@data-test01 ~]# vim /usr/sbin/ambari-server.py 
[root@data-test01 ~]# 
         
```

从以上debug堆栈中我们可以看到ambari-server启动过程中python脚本的所有准备工作。其中java进程的参数主要在ambari_server_main.py的server_process_main()的param_list中。

那么param_list来源在哪儿呢？ 是在server_process_main的子方法generate_child_process_param_list()中的command_base变量，SERVER_START_CMD_DEBUG或者SERVER_START_CMD的形式如下。
> {0} -server -XX:NewRatio=3 -XX:+UseConcMarkSweepGC -XX:-UseGCOverheadLimit -XX:CMSInitiatingOccupancyFraction=60 -Dsun.zip.disableMemoryMapping=true {1} {2} -cp {3} org.apache.ambari.server.controller.AmbariServer > {4} 2>&1 || echo $? > {5} & 

可以看到留下了很多占位符。还是看一下generate_child_process_param_list的code吧
```python
@FamilyFuncImpl(OsFamilyImpl.DEFAULT)                                          
def generate_child_process_param_list(ambari_user, java_exe, class_path,         
                                      debug_start, suspend_mode):                
  from ambari_commons.os_linux import ULIMIT_CMD                                 
                                                                                 
  properties = get_ambari_properties()                                           
                                                                                 
  command_base = SERVER_START_CMD_DEBUG if debug_start else SERVER_START_CMD     
                                                                                 
  ulimit_cmd = "%s %s" % (ULIMIT_CMD, str(get_ulimit_open_files(properties)))    
  command = command_base.format(java_exe,                                        
          ambari_provider_module_option,                                         
          jvm_args,                                                              
          class_path,                                                            
          configDefaults.SERVER_OUT_FILE,                                        
          os.path.join(configDefaults.PID_DIR, EXITCODE_NAME),                   
          suspend_mode)                                                          
                                                                                 
  # required to start properly server instance                                   
  os.chdir(configDefaults.ROOT_FS_PATH)                                          
                                                                                 
  #For properly daemonization server should be started using shell as parent     
  param_list = [locate_file('sh', '/bin'), "-c"]                                 
  if is_root() and ambari_user != "root":                                        
    # To inherit exported environment variables (especially AMBARI_PASSPHRASE),  
    # from subprocess, we have to skip --login option of su command. That's why  
    # we change dir to / (otherwise subprocess can face with 'permission denied' 
    # errors while trying to list current directory                              
    cmd = "{ulimit_cmd} ; {su} {ambari_user} -s {sh_shell} -c '{command}'".format(ulimit_cmd=ulimit_cmd, 
                                                                                su=locate_file('su', '/bin'), ambari_user=ambari_user,
                                                                                sh_shell=locate_file('sh', '/bin'), command=command)
  else:                                                                          
    cmd = "{ulimit_cmd} ; {command}".format(ulimit_cmd=ulimit_cmd, command=command)
                                                                                 
  param_list.append(cmd)                                                         
  return param_list     
```

这个方法把java进程的相关的东西全部拼进cmd，之后放进param_list，其中放了一个jvm_args的参数就是jvm参数了。在当前python文件中追溯此数据的来源：jvm_args = os.getenv('AMBARI_JVM_ARGS', '-Xms512m -Xmx2048m') ，所以它是一个环境变量。依赖eclipse的索引，搜一下哪个脚本里设置了这个环境变量：ambari-env.sh。


### 参考
http://www.ibm.com/developerworks/cn/linux/l-cn-pythondebugger/
