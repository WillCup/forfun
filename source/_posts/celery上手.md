
title: celery上手
date: 2016-12-05 17:09:24
tags: [youdaonote]
---


首先我们需要有一个celery实例，也就是celery app。有了这个实例我们才能创建task、管理worker，只需要把这个实例作为moduel import进来就可以了。

创建实例，并创建task：
```py
from celery import Celery

app = Celery('tasks', broker='amqp://guest@localhost//')

@app.task
def add(x, y):
    return x + y
```

启动worker：
```
 celery -A tasks worker --loglevel=info
```

```
-------------- celery@halcyon.local v3.1 (Cipater)
---- **** -----
--- * ***  * -- [Configuration]
-- * - **** --- . broker:      amqp://guest@localhost:5672//
- ** ---------- . app:         __main__:0x1012d8590
- ** ---------- . concurrency: 8 (processes)
- ** ---------- . events:      OFF (enable -E to monitor this worker)
- ** ----------
- *** --- * --- [Queues]
-- ******* ---- . celery:      exchange:celery(direct) binding:celery
--- ***** -----

[2012-06-08 16:23:51,078: WARNING/MainProcess] celery@halcyon.local has started.
```

- concurrency 是worker进程的并发数据，默认是cpu个数。需要视情况而定，cpu密集型的就少一些，IO密集型的就多一些。除了默认的prefork 池，celery还支持eventlet、gevent以及threads。
- event。 启用后celery就发送这个worker里的监控event。这可以被celery events、flower使用。
- queue。代表这个worker会消费的队列。


调用task：
```py
>>> from tasks import add
>>> add.delay(4, 4)
```
使用delay()方法调用task，它是apply_async()方法的简写。然后我们可以去worker的console观察任务执行了。

调用task放回的对象是AsyncResult实例，它可以用来获取当前task的状态，等待task结束后，获取task的返回值。默认是没有启用的，要使用的话需要给celery配置一个result backend。

```py
app = Celery('tasks', backend='redis://localhost', broker='amqp://')
```

这里是使用redis作为task状态和结果的存储介质。再次调用
```py
>>> result = add.delay(4, 4)
>>> result.ready()
False
# 一般并不会直接获取这个结果，这样就把异步调用弄成同步调用一样了
>>> result.get(timeout=1)
8
## 当出现异常的时候，get方法默认也会抛出这个异常，可以使用propagate来阻止
>>> result.get(propagate=False)
## 获取异常堆栈
>>> result.traceback
>>> res.failed()
True

>>> res.successful()
False
# 一个task的典型status历程：PENDING -> STARTED -> SUCCESS
# 重试的task：PENDING -> STARTED -> RETRY -> STARTED -> RETRY -> STARTED -> SUCCESS
>>> res.state
'FAILURE'
```



要使配置马上生效的话，可以调用update方法
```py
app.conf.update(
    CELERY_TASK_SERIALIZER='json',
    CELERY_ACCEPT_CONTENT=['json'],  # Ignore other content
    CELERY_RESULT_SERIALIZER='json',
    CELERY_TIMEZONE='Europe/Oslo',
    CELERY_ENABLE_UTC=True,
)
```
要是大型项目，弄一个特定的配置module比较好。最好不要硬编码周期任务和task路由，把这些放在一个集中的位置，方便控制。比如系统管理员在系统故障的时候做一些简单的微调什么的。

使用`app.config_from_object()`方法，使celery实例使用一个配置module。
```py
app.config_from_object('celeryconfig')
```
celeryconfig.py:
```py
BROKER_URL = 'amqp://'
CELERY_RESULT_BACKEND = 'rpc://'

CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_ACCEPT_CONTENT=['json']
CELERY_TIMEZONE = 'Europe/Oslo'
CELERY_ENABLE_UTC = True
```
注意，这个module应该可以import到才对。配置文件中可以指定路由、注解等很多东西。
```py
CELERY_ANNOTATIONS = {
    'tasks.add': {'rate_limit': '10/m'}
}

CELERY_ROUTES = {
    'tasks.add': 'low-priority',
}
```

要是使用rabbitMQ或者Redis作为broker的话，还可以直接通过命令进行配置：
```
celery -A tasks control rate_limit tasks.add 10/m
```

---

实际使用celery
---

项目结构：
```
proj/__init__.py
    /celery.py
    /tasks.py
```
celery.py文件内容：
```py
from __future__ import absolute_import

from celery import Celery

app = Celery('proj',
             broker='amqp://',
             backend='amqp://',
             include=['proj.tasks'])

# Optional configuration, see the application user guide.
app.conf.update(
    CELERY_TASK_RESULT_EXPIRES=3600,
)

if __name__ == '__main__':
    app.start()
```
在celery中可以创建自己的实例，然后在其他地方import这个实例就可以使用了。

注意一下include，应该把自定义的task的module填写进来，这样celery实例启动的时候就会加载这些module。

我们的proj/tasks.py：
```py
from __future__ import absolute_import

from proj.celery import app


@app.task
def add(x, y):
    return x + y


@app.task
def mul(x, y):
    return x * y


@app.task
def xsum(numbers):
    return sum(numbers)
```

启动后要停掉worker的话，只需要直接ctrl + c就可以了。

生产环境中需要以守护进程的形式启动worker，使用`celery multi`任务就可以了，它可以启动一个或多个worker。

```
celery multi start w1 -A proj -l info
celery multi v3.1.1 (Cipater)
> Starting nodes...
    > w1.halcyon.local: OK

$ celery  multi restart w1 -A proj -l info
celery multi v3.1.1 (Cipater)
> Stopping nodes...
    > w1.halcyon.local: TERM -> 64024
> Waiting for 1 node.....
    > w1.halcyon.local: OK
> Restarting node w1.halcyon.local: OK
celery multi v3.1.1 (Cipater)
> Stopping nodes...
    > w1.halcyon.local: TERM -> 64052
    
celery multi stop w1 -A proj -l info
```

stop命令是异步的，这样就不用hang，等着worker停止了。为了我保证所有的task能够完成，也可以使用stopwait方法。
```
celery multi stopwait w1 -A proj -l info
```

下面是加上指定pid，和log的启动命令：
```
$ mkdir -p /var/run/celery
$ mkdir -p /var/log/celery
$ celery multi start w1 -A proj -l info --pidfile=/var/run/celery/%n.pid \
                                        --logfile=/var/log/celery/%n%I.log
```

```
 celery multi start 10 -A proj -l info -Q:1-3 images,video -Q:4,5 data \
    -Q default -L:4,5 debug
```

#### Canvas：工作流
使用subtask搞定，它封装了单个task的参数，可以传递给后面的方法。

```py
>>> add.subtask((2, 2), countdown=10)
tasks.add(2, 2)
```
可以替换为:
```py
>>> add.s(2, 2)
tasks.add(2, 2)
```

比较牛逼的地方在于，他竟然可以定义偏函数
```py
>>> s1 = add.s(2, 2)
>>> res = s1.delay()
>>> res.get()
4

# 空的是前面的参数，后面的是2
# incomplete partial: add(?, 2)
>>> s2 = add.s(2)

# resolves the partial: add(8, 2)
>>> res = s2.delay(8)
>>> res.get()
10

>>> s3 = add.s(2, 2, debug=True)
>>> s3.delay(debug=False)   # debug is now False.
```

#### primitive
- group
- map
- chain
- starmap
- chord
- chunks
这些primitive本身就是subtask，所以它们可以组合成工作流。

###### groups

group并行调用多个task，返回一串结果。
```py
>>> from celery import group
>>> from proj.tasks import add

>>> group(add.s(i, i) for i in xrange(10))().get()
[0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```
结合偏函数使用group：
```py
>>> g = group(add.s(i) for i in xrange(10))
>>> g(10).get()
[10, 11, 12, 13, 14, 15, 16, 17, 18, 19]
```

###### chain
有序调用一系列task
```py
>>> from celery import chain
>>> from proj.tasks import add, mul

# (4 + 4) * 8
>>> chain(add.s(4, 4) | mul.s(8))().get()
64
```
偏函数：
```py
# (? + 4) * 8
>>> g = chain(add.s(4) | mul.s(8))
>>> g(4).get()
64
```

```py
>>> (add.s(4, 4) | mul.s(8))().get()
64
```
###### chord
带有一个回调函数的一组task。
```py
>>> from celery import chord
>>> from proj.tasks import add, xsum

>>> chord((add.s(i, i) for i in xrange(10)), xsum.s())().get()
90
```
被chain到其他task的group自动转为chord：
```py
>>> (group(add.s(i, i) for i in xrange(10)) | xsum.s())().get()
90
```
结合多种subtask使用：
```py
>>> upload_document.s(file) | group(apply_filter.s() for filter in filters)
```


#### 路由

主要是针对consumer/worker和producer各自的队列

#### 远程控制

使用RabbitMQ、或者Redis作为broker的话可以在运行时控制与查看worker的状态。
```
$ celery -A proj inspect active
```
这是查看当前正在运行的task。





参考：
- http://docs.celeryproject.org/en/latest/getting-started/first-steps-with-celery.html
- http://docs.celeryproject.org/en/latest/getting-started/next-steps.html 
