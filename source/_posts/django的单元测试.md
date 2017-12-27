
title: django的单元测试
date: 2017-09-18 15:05:08
tags: [youdaonote]
---

编写测试
---

使用了unittest
```
from django.test import TestCase
from myapp.models import Animal

class AnimalTestCase(TestCase):
    def setUp(self):
        Animal.objects.create(name="lion", sound="roar")
        Animal.objects.create(name="cat", sound="meow")

    def test_animals_can_speak(self):
        """Animals that can speak are correctly identified"""
        lion = Animal.objects.get(name="lion")
        cat = Animal.objects.get(name="cat")
        self.assertEqual(lion.speak(), 'The lion says "roar"')
        self.assertEqual(cat.speak(), 'The cat says "meow"')
```


运行测试
---
```
# Run all the tests in the animals.tests module
$ ./manage.py test animals.tests

# Run all the tests found within the 'animals' package
$ ./manage.py test animals

# Run just one test case
$ ./manage.py test animals.tests.AnimalTestCase

# Run just one test method
$ ./manage.py test animals.tests.AnimalTestCase.test_animals_can_speak

$ ./manage.py test animals/

$ ./manage.py test --pattern="tests_*.py"
```

测试数据库
---
通过settings文件中的DATABASES里的TEST字段指定测试数据库的名称。要求对应的用户有创建测试数据库的权限。

执行顺序
---
为确保所有的TestCase代码都运行在一个clean的database之上，django的测试运行顺序如下：
 - 所有的django.test.TestCase子类
 - 所有SimpleTestCase的子类，包括TransactionTestCase。他们的运行顺序不能保证
 - unittest.TestCase的子类，这些类可能会修改db，而且不能恢复db的原始状态
 

你可以通过test --reverse选项，反转执行顺序。这样可以确保所有的TestCase之间都是无依赖的。



回滚
---
任何在migration里加载的初始数据都可以在TestCase中被使用，但是在TranactionTestCase里就不能使用，除非对于后台引擎支持事务才行，比如MyISAM引擎就不行。


django可以通过设置serialized_rollback 为true，为每个TestCase reload数据，但是注意这会让速度变慢。


第三方的app必须要设置这个，通常，我们都是用事务性数据库，就没有必要使用了。


初始序列化工作其实挺快的，但是如果有些APP并不需要序列化的话，可以通过TEST_NON_SERIALIZED_APPS进行配置。


加速测试
---
- 使用并行测试 test --parallel 3
- 密码hash。默认密码hasher比较慢，可以指定自己的。
- test --keepdb，测试完成后不删除测试数据库。跳过创建于删除动作

