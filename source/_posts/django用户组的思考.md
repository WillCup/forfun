
title: django用户组的思考
date: 2017-05-10 17:24:10
tags: [youdaonote]
---

我们有两个不同的事业部A, B， 目前是用django的group来做的。。。都在我的平台上编辑自己的内容OBJ，我给OBJ指定了唯一一个group_id外键来保证AB事业部内容的隔离。。。。现在遇到问题，因为要控制一些权限，当加group的时候.....自己就懵逼了


不该这样处理AB，对吧。。。应该额外建立一个oaganization来作为AB隔离的。django的group用来做操作权限控制更好，而不是对象隔离。


所以现在的代码结构是：
```python
class OrgGroup(models.Model):
    name = models.CharField(max_length=200, verbose_name=u"组织名称")
    owners = models.CharField(max_length=100, verbose_name=u"负责人", blank=True, default='')
    hdfs_path = models.CharField(max_length=100, verbose_name=u"HDFS临时路径", blank=True, default='')
    def __str__(self):
        return self.name

class UserProfile(models.Model):
    user = models.OneToOneField(User)
    phone = models.BigIntegerField(default=110)
    org_group = models.ForeignKey(OrgGroup, on_delete=models.DO_NOTHING, related_name='user_cgroup', null=True)

    def __str__(self):
        return self.user.username
```
重新定义了一个组织的model，然后每个人都有属于某个特定的组织。

```py
class JarApp(models.Model):
    .....
    cgroup = models.ForeignKey(OrgGroup, on_delete=models.DO_NOTHING, 
    .....
```

上面的cgroup原来是
```
cgroup = models.ForeignKey(auth.Group, on_delete=models.DO_NOTHING, related_name='jar_cgroup', null=True)
```

然后还要重构JarApp的新建与查询的一些代码。


这样，使用自定义的组织来划分人员与面向的内容。使用django的group来控制是否组织内特定人员组具有的权限【ETL开发人员、与报表开发人员】。

自定义的组织用来限定报表和ETL所属的组织。而django的auth.group可以管理一组权限【对ETL、报表的访问权限、操作权限等】。


