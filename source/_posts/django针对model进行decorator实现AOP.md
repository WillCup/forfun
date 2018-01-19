title: django针对model进行decorator实现AOP
date: 2018-01-19 16:14:51
tags: [django, 架构，jaeger]
---
#### 1. 期望

通过decorator实现对django中所有Model操作的AOP，可以用来记录所以model操作的log，也可用来进行调用追踪。

#### 2. 思路

想到django是可以直接配置logger为`django.db.backends`来打印具体SQL的，所以我们就可以以这里为突破口，进行实际SQL的AOP的trace。经过源码追踪，发现关键类`django.db.backends.mysql.base.CursorWrapper`，主要包含两个方法`execute`和`executemany`，但是这个东西只有在settings.DEBUG为True的时候才生效。

找下django有没有提供一些进行这个部分custom的方法，么有搜到，但是搜到了一个更大更全的custom选项，就是直接把整个backend engine都进行定制。
```py
DATABASES = {
    'default': {
        'ENGINE': 'metamap.will_backends',
        'NAME': result['MAIN_DB_NAME'],
        'PASSWORD': result['MAIN_DB_PWD'],
        'USER': result['MAIN_DB_USER'],
        'HOST': result['MAIN_DB_HOST'],
        'PORT': result['MAIN_DB_PORT'],
        'TEST': {
            'NAME': 'metamap1',
        },
    }
}

```

虽然官网没有提及自定义database backend的，但是搜到了一些相关的文章，有兴趣的同学可以看一下。通过上面的配置也可以很容易看出`ENGINE`是可以使用自定义的东西的。因为我们这里只是要针对Wrapper弄个AOP，所以直接找到django的mysql base，整体复制下来，加上我们的tracer装饰器就可以了。

#### 3. 实施

复制django原生mysql引擎代码到自己的项目中，目录结构如下：
```
app_dir
├── admin.py
......
.....
└── will_backends   # mysql引擎目录
    ├── base.py     # 目标所在文件
    ├── client.py
    ├── compiler.py
    ├── creation.py
    ├── features.py
    ├── __init__.py
    ├── introspection.py
    ├── operations.py
    ├── schema.py
    └── validation.py

```

我们的tracer装饰器代码在上一篇文章《django中使用threadlocal存放tracer》已经提到了。自定义的mysql的`CursorWrapper`代码如下：
```py

from will_common.decorators import jaeger_tracer
class CursorWrapper(object):
    def __init__(self, cursor):
        self.cursor = cursor

    @jaeger_tracer('mysql')
    def execute(self, query, args=None):
        ....

    @jaeger_tracer('mysql')
    def executemany(self, query, args):
        .....

```

很简单，加上我们的装饰器。看下效果


![产出的jaeger trace效果图](/imgs/trace/django_jaeger_mysql_tracer.png)

#### 参考
- [python反射相关](https://docs.python.org/2/library/inspect.html#inspect.getsourcefile)
- [监听django model的操作](https://gist.github.com/ur001/3e86d43f1e80f17fbfd9)
- [基于decorator的元编程](https://www.ibm.com/developerworks/cn/linux/l-cpdecor.html)
- [django日志](https://docs.djangoproject.com/en/2.0/topics/logging/)
