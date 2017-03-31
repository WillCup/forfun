title: hue metastore引出的HDFS权限问题
date: 2017-03-31 16:32:29
tags: [hdfs, hue]
---

需求
---
支持的某个事业部提出一个需求：**在hue里开放修改表或者字段的权限**。


在hue里就给她一个人开了相应的权限，账户名是wcq。因为不知道wcq的密码，所以使用的mm的测试账户登陆的，**在hue里给了两个人相同的权限组：lv和metastoremanger**。

问题
---
然而奇怪的现象就发生了：**wcq在hue的metastore管理界面修改表的字段之后，看网络请求是成功了的。然而，刷新页面以后就没有了。mm这个账户确都可以使用。**


追查
---
经查验，mm这个用户在HDFS中除了是lv组的用户，而且还是hdfs组的用户，而**hdfs组是超级用户组，拥有任何权限**。所以这两个用户所在的组并不是对等的，只是在hue里对等而已。


理论上如果hue直接操作hive元数据库的话是不会有此问题的。

#### 查一下网络请求有什么异常？
发送的request包摘要：
```
Request URL:http://10.1.5.83/metastore/table/dim_tinyv/data_dict/alter_column
Request Method:POST

column:id
comment:ss
```

返回的response内容：
```json
{
    "status": 0,
    "message": "",
    "data": {
        "comment": "",
        "type": "int",
        "name": "id"
    }
}
```


#### hue源码确认逻辑
以url为线索查看hue相关源码`apps/metastore/src/metastore/urls.py`中找到`alter_table`，在`apps/metastore/src/metastore/views.py`里：

```py
check_has_write_access_permission
@require_http_methods(["POST"])
def alter_column(request, database, table):
  db = dbms.get(request.user)
  response = {'status': -1, 'message': ''}
  try:
    column = request.POST.get('column', None)

    if column is None:
      raise PopupException(_('alter_column requires a column parameter'))

    column_obj = db.get_column(database, table, column)
    if column_obj:
      new_column_name = request.POST.get('new_column_name', column_obj.name)
      new_column_type = request.POST.get('new_column_type', column_obj.type)
      comment = request.POST.get('comment', None)
      partition_spec = request.POST.get('partition_spec', None)

      column_obj = db.alter_column(database, table, column, new_column_name, new_column_type, comment=comment, partition_spec=partition_spec)

      response['status'] = 0
      response['data'] = {
        'name': column_obj.name,
        'type': column_obj.type,
        'comment': column_obj.comment
      }
    else:
      raise PopupException(_('Column `%s`.`%s` `%s` not found') % (database, table, column))
  except Exception, ex:
    response['status'] = 1
    response['message'] = _("Failed to alter column `%s`.`%s` `%s`: %s") % (database, table, column, str(ex))

  return JsonResponse(response)
```

看到是使用了dbms里返回的数据库执行的语句，找到`apps/beeswax/src/beeswax/server/dbms.py`，对应代码：
```py
def get(user, query_server=None):
  global DBMS_CACHE
  global DBMS_CACHE_LOCK

  if query_server is None:
    query_server = get_query_server_config()

  DBMS_CACHE_LOCK.acquire()
  try:
    DBMS_CACHE.setdefault(user.username, {})

    if query_server['server_name'] not in DBMS_CACHE[user.username]:
      # Avoid circular dependency
      from beeswax.server.hive_server2_lib import HiveServerClientCompatible

      if query_server['server_name'] == 'impala':
        from impala.dbms import ImpalaDbms
        from impala.server import ImpalaServerClient
        DBMS_CACHE[user.username][query_server['server_name']] = ImpalaDbms(HiveServerClientCompatible(ImpalaServerClient(query_server, user)), QueryHistory.SERVER_TYPE[1][0])
      else:
        from beeswax.server.hive_server2_lib import HiveServerClient
        DBMS_CACHE[user.username][query_server['server_name']] = HiveServer2Dbms(HiveServerClientCompatible(HiveServerClient(query_server, user)), QueryHistory.SERVER_TYPE[1][0])

    return DBMS_CACHE[user.username][query_server['server_name']]
  finally:
    DBMS_CACHE_LOCK.release()
```

可以看到是使用hiveserver2建立连接，搞定的。并不是最开始猜测的直接操作hive的元数据库。

修改数据字段的方法也在这里：
```py
def alter_column(self, database, table_name, column_name, new_column_name, column_type, comment=None,
                   partition_spec=None, cascade=False):
    hql = 'ALTER TABLE `%s`.`%s`' % (database, table_name)

    if partition_spec:
      hql += ' PARTITION (%s)' % partition_spec

    hql += ' CHANGE COLUMN `%s` `%s` %s' % (column_name, new_column_name, column_type.upper())

    if comment:
      hql += " COMMENT '%s'" % comment

    if cascade:
      hql += ' CASCADE'

    timeout = SERVER_CONN_TIMEOUT.get()
    query = hql_query(hql)
    handle = self.execute_and_wait(query, timeout_sec=timeout)

    if handle:
      self.close(handle)
    else:
      msg = _("Failed to execute alter column statement: %s") % hql
      raise QueryServerException(msg)

    return self.get_column(database, table_name, new_column_name)

```

#### 执行相应逻辑
这样的话，我们直接在命令行里，切换到wcq用户执行hive命令行，应该是一样的。
```
hive> alter table default.batting CHANGE COLUMN dtdontquery dtdontquery string COMMENT 'ss';
FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.DDLTask. Unable to alter table. java.security.AccessControlException: Permission denied: user=wcq, access=WRITE, inode="/apps/hive/warehouse/batting":root:jlc:drwxr-xr-x

```

此阶段结论：**hue通过hiveserver执行hql语句，而且hive本身修改表元数据的时候也会验证是否有对HDFS目录有写权限**。



解决
---
那么应该怎样解决呢？


当前的目录权限都是给的：`drwxr-x---`，也就是只有owner有写权限, 指定组的用户有读权限，匿名用户没有任何权限。

这样授权的理由在于，以事业部为组，所有同一个事业部的人都可读，但是只能通过我们的数据平台用户来进行写操作【当前是root】。


既不想违背这个初衷，又要满足新的需求，再不进行任何自主研发的前提下，只能**启用HDFS的ACL**，才能添加额外的权限。

hue中也支持*ACL修改，而且执行遍历目录修改.

步骤如下：
![](/imgs/hue_acl/hue_dfs_acl_1.png)
![](/imgs/hue_acl/hue_dfs_acl_2.png)
![](/imgs/hue_acl/hue_dfs_acl_3.png)
![](/imgs/hue_acl/hue_dfs_acl_4.png)


遗留
---
对官网说的mask不太理解。字面意思好像是用来过滤所有用户、组、匿名组的权限的。

以官网例子来看：
```
  user::rw-
   user:bruce:rwx                  #effective:r--
   group::r-x                      #effective:r--
   group:sales:rwx                 #effective:r--
   mask::r--
   other::r--
```
因为上面指定的mask是只读，所以即便其他指定了额外的权限也是白费的，生效的只有只读。所以这样理解，mask就是这个文件可以有的所有权限的总和。如果不单独设置mask的话，其默认值也是这样算出来的。

参考
---
- https://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-hdfs/HdfsPermissionsGuide.html#The_Super-User
- https://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-common/FileSystemShell.html#setfacl
