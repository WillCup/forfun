
title: hive中的权限-SQL-Standard-Based-Hive-Authorization
date: 2016-12-05 17:08:48
tags: [youdaonote]
---

## hive 0.13之前
默认hive的权限机制并不是为了防止一些恶意用户访问一些他们不该看的东西的，只是为了防止用户误操作。这套机制很不完善，甚至对于grant这种语句也不会做权限检查。权限检查是在hive query的编译阶段进行的。由于用户能够操作dfs、执行自定义函数、还能操作shell，跳过客户端的安全检查也不是不可能的。

hive还支持存储层的权限验证，主要用来给metastore server api添加权限验证（参考 Storage Based Authorization in the Metastore Server）。0.12版本之后，在client端也可以使用了。这样metastore就被保护起来了，但是对于细粒度访问权限还是控制不了（例如行、列）。

一般通过创建view来解决这个问题，只让用户访问到view层。

## 0.13之后
基于sql标准的检查机制,推荐使用此种方式。
这种认证方式可以在metastore server上结合storage based authorization一起使用。和目前目前hive的默认认证机制一样也算在hive query的编译阶段进行验证的。为了保证安全性，我们需要确认client端是安全的，让用户通过hiveserver2来访问数据，约束用户代码和非法sql命令。检查是针对提交query给hive server2的用户的，但是实际执行查询的时候使用hive server2用户执行，目录和文件也都只对hive server2的用户开放。对于不需要进行检查约束的用户，则可以直接开放给他们hive命令行执行权限，也就是查询的时候就使用提交的那个用户。

这项工作是尽量像SQL标准看齐的，不过也有一些有差异的地方。出发点一般都是为了让既有用户能够平滑的迁移到认证机制，或者考虑到易用性。

在这种认证机制下，所有可以访问到hive命令行、HDFS命令行、pig命令行、Hadoop jar等的用户都被认为是特权用户。团队中只有做ETL工作的人才能拥有这些权利。这些工具都不通过hive server访问数据，也就是说他们不会被检查认证情况。对于通过hive 客户端、pig、MR等访问hive table的用户可以控制他们启用metastore server的storage based authorization。

类似数据分析等使用sql或者jdbc通过hive server2访问数据的用户可以使用此套认证机制。

### hive 命令和语句的限制
启用此认证机制后
 - dfs\add\delete\compile\reset等命令都不可用。
 - hive运行时的set的配置参数只有少数能够被设置， hive.security.authorization.sqlstd.confwhitelist 控制这个东西，这里面是可以配置的参数的白名单。
 - 只有admin角色才能添加和删除function和macro。admin角色需要为普通用户添加permanent function，这样其他用户才能使用自定义的一些function。
 - transform clause不可用
 
### 权限
- select 
- insert
- update
- delete
- all

### 对象
 - table和view。不支持database
 - database所有权只对应部分动作
 - hive里还有特殊的对象URI。上面的权限对URI不适用，URI是指向系统的某个文件的。认证机制是通过用户的文件权限进行认证的。
 
### 对象所有权
针对每个操作，都会检查当前用户对于当前对象是否有权限。创建的人就是owner。对于table和view来说owner有所有权限。某个role也可以是一个database的owner，通过alter database来修改database的owner。


### user 和 role
权限可以给user，也可以给role。一个user可以有一个或多个role。
两个特殊的role：public以及admin。所有的user都是public role。You use this role in your grant statement to grant a privilege to all users.

当一个user运行hive命令的时候，就会检查这个用户的role以及它目前拥有的所有权限。使用"show current roles;"命令来查看当前用户的role。除了admin其他用户都可以在current role里查到，而且可以使用"set role xxx"命令来给当前用户设置一个role。

database的管理者应该使用admin role。这些用户可以执行类似"create role""drop role"等命令，也可以访问一切资源。但是，这些用户在成为admin role之前，必须先执行"set role admin"，因为admin默认是不再current roles里的。

### role相关命令
```sql
-- 只有admin角色才能执行的命令
CREATE ROLE role_name;
DROP ROLE role_name;
SHOW ROLES;
-- Show Principals 查看role的权限
SHOW PRINCIPALS role_name;

SHOW CURRENT ROLES;
SET ROLE (role_name|ALL|NONE);
--All 是当有新的role被赋予给当前user的时候，刷新一下current roles
--None 移除所有的current roles

-- Grant role
GRANT role_name [, role_name] ...
TO principal_specification [, principal_specification] ...
[ WITH ADMIN OPTION ];
 
principal_specification
  : USER user
  | ROLE role

-- 如果指定了'with admin option',那么这个用户就也能将这个role赋予其他user/role。如果出现role之间grant循环，就会直接报错。 

-- Revoke Role
REVOKE [ADMIN OPTION FOR] role_name [, role_name] ...
FROM principal_specification [, principal_specification] ... ;
 
principal_specification
  : USER user
  | ROLE role

-- Show Role Grant 只能查看自己的role的权限
SHOW ROLE GRANT (USER|ROLE) principal_name;

```

### 管理对象权限
```sql
-- 授予权限
GRANT
    priv_type [, priv_type ] ...
    ON table_or_view_name
    TO principal_specification [, principal_specification] ...
    [WITH GRANT OPTION];
    
-- 收回权限
REVOKE [GRANT OPTION FOR]
    priv_type [, priv_type ] ...
    ON table_or_view_name
    FROM principal_specification [, principal_specification] ... ;
 
principal_specification
  : USER user
  | ROLE role
 
priv_type
  : INSERT | SELECT | UPDATE | DELETE | ALL
  
-- Show Grant
SHOW GRANT [principal_name] ON (ALL| ([TABLE] table_or_view_name)
show grant user hive on all


 
```
注意revoke语句不能drop任何有依赖的权限。详见Postgres revoke documentation.


## 参考
- SQL Standards Based Authorization in HiveServer2
- https://cwiki.apache.org/confluence/display/Hive/SQL+Standard+Based+Hive+Authorization
- https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Authorization#LanguageManualAuthorization-2SQLStandardsBasedAuthorizationinHiveServer2

