
title: celery队列路由
date: 2016-12-05 17:10:27
tags: [youdaonote]
---

同一个app里如果有多种不同的任务，也就是分由不同的task进行处理。假设只有一个worker的话，task1耗时特别长，有可能会阻塞短平快的task2、task3、task4的执行，也就是面临着小任务饿死的情况。

这时候我们就需要使用队列路由，将不同类型的任务路由至不同的queue，并且启动多个worker，分别负责消费指定queue的任务。这样，长少慢的任务就不会影响短频快的小任务的执行了。

自动路由
---
CELERY_CREATE_MISSING_QUEUES开启后，如果queue没有在CELERY_QUEUES中的话会自动创建。

假设我们有两个server：x y。他俩负责普通的task。然后z server只处理特定的跟种子有关的task，那么可以这样配置：
```
CELERY_ROUTES = {'feed.tasks.import_feed': {'queue': 'feeds'}}
```

所有feed.tasks.import_feed的任务都会被放进feeds队列，其他的就放进默认的队列【celery】。

然后启动z server的时候指定一下队列名称：
```
user@z:/$ celery -A proj worker -Q feeds
```

修改默认队列的名称
---
```py
from kombu import Exchange, Queue

CELERY_DEFAULT_QUEUE = 'default'
CELERY_QUEUES = (
    Queue('default', Exchange('default'), routing_key='default'),
)
```

队列定义
---
celery隐藏了后面负责的AMQP协议，但是有人也想知道怎样声明队列。

下面是创建一个video队列：
```
{'exchange': 'video',
 'exchange_type': 'direct',
 'routing_key': 'video'}
```
非AMQP协议的后端不支持exchange。


手动配置
---
还是上面一样，xy处理普通任务，z处理feed相关任务：
```py
from kombu import Queue

CELERY_DEFAULT_QUEUE = 'default'
CELERY_QUEUES = (
    Queue('default',    routing_key='task.#'),
    Queue('feed_tasks', routing_key='feed.#'),
)
CELERY_DEFAULT_EXCHANGE = 'tasks'
CELERY_DEFAULT_EXCHANGE_TYPE = 'topic'
CELERY_DEFAULT_ROUTING_KEY = 'task.default'
```

CELERY_QUEUES里定义了一串队列的实例。如果没有设置exchange或者没给key设置exchange类型，会按照默认的CELERY_DEFAULT_EXCHANGE、CELERY_DEFAULT_EXCHANGE_TYPE。

要指定某个task的路由，需要在CELERY_ROUTES里添加一个入口
```py
CELERY_ROUTES = {
        'feeds.tasks.import_feed': {
            'queue': 'feed_tasks',
            'routing_key': 'feed.import',
        },
}
```

也可以在代码调用的时候指定：
```py
>>> from feeds.tasks import import_feed
>>> import_feed.apply_async(args=['http://cnn.com/rss'],
...                         queue='feed_tasks',
...                         routing_key='feed.import')
```

启动z：
```py
user@z:/$ celery -A proj worker -Q feed_tasks --hostname=z@%h
```

还要配置x y启动的时候只消费默认队列default：
```
user@x:/$ celery -A proj worker -Q default --hostname=x@%h
user@y:/$ celery -A proj worker -Q default --hostname=y@%h
```

这些队列也可以混搭在一起，如果你的server都能处理的话，也可以这么做：
```
user@z:/$ celery -A proj worker -Q feed_tasks,default --hostname=z@%h
```

要是想添加一个在其他exchange上的队列，自己定义好exchange以及exchange类型就行：
```py
from kombu import Exchange, Queue

CELERY_QUEUES = (
    Queue('feed_tasks',    routing_key='feed.#'),
    Queue('regular_tasks', routing_key='task.#'),
    Queue('image_tasks',   exchange=Exchange('mediatasks', type='direct'),
                           routing_key='image.compress'),
)
```
要是对这部分内容有疑问，可以自行查阅一下AMQP。



初识AMQP
---

#### message

message包含header和body。celery使用head来存储内容类型和内容编码。内容类型用来序列化message。body中是要执行的task的name，id，参数以及其他元数据信息。

下面是一个例子：
```py
{'task': 'myapp.tasks.add',
 'id': '54086c5e-6193-4575-8308-dbab76798756',
 'args': [4, 4],
 'kwargs': {}}
```

#### Producers, consumers， brokers
发送message的客户端叫做publisher，也叫producer。收到message的就叫做consumer。

broker是message所在的server，把message从producer路由到consumer。

#### Exchanges, queues， routing keys
1. message被发送给exchange
2. exchange路由message到一或多个队列。有多种exchange类型，提供不同的路由方式，实现不同的消息场景
3. message在队列中等着consumer消费
4. 当message被ack之后就会从队列中移除

下面是发送与接收message的必要步骤：
1. 创建一个exchange
2. 创建一个队列
3. 把队列绑定到exchange上

celery自动为CELERY_QUEUES创建了以上这些对象【除非你关闭了自动创建不存在队列的配置】。

下面是一个包含三个队列的配置，一个video，一个image，还一个默认的：
```py
from kombu import Exchange, Queue

CELERY_QUEUES = (
    Queue('default', Exchange('default'), routing_key='default'),
    Queue('videos',  Exchange('media'),   routing_key='media.video'),
    Queue('images',  Exchange('media'),   routing_key='media.image'),
)
CELERY_DEFAULT_QUEUE = 'default'
CELERY_DEFAULT_EXCHANGE_TYPE = 'direct'
CELERY_DEFAULT_ROUTING_KEY = 'default'
```

#### Exchange类型
标准的有：direct, topic, fanout, headers。非标准的就是一些rabbitMQ的插件了。

##### direct
匹配精确的routing key，绑定到routing key video的队列只会接收带有video routing key的message。

##### topc
支持对号隔开、带有通配符的routing key。

#### 相关API
```
exchange.declare(exchange_name, type, passive,
durable, auto_delete, internal)
Declares an exchange by name.

See amqp:Channel.exchange_declare.

Parameters:	
    passive – Passive means the exchange won’t be created, but you can use this to check if the exchange already exists.
    durable – Durable exchanges are persistent. That is - they survive a broker restart.
    auto_delete – This means the queue will be deleted by the broker when there are no more queues using it.
    queue.declare(queue_name, passive, durable, exclusive, auto_delete)
    Declares a queue by name.

See amqp:Channel.queue_declare

Exclusive queues can only be consumed from by the current connection. Exclusive also implies auto_delete.

queue.bind(queue_name, exchange_name, routing_key)
Binds a queue to an exchange with a routing key.

Unbound queues will not receive messages, so this is necessary.

See amqp:Channel.queue_bind

queue.delete(name, if_unused=False, if_empty=False)
Deletes a queue and its binding.

See amqp:Channel.queue_delete

exchange.delete(name, if_unused=False)¶
Deletes an exchange.

See amqp:Channel.exchange_delete
```

#### 使用API

自带了一个工具 celery amqp，用来在命令行访问AMQP API，执行创建/删除队列或exchange的，清空队列或者正在发送的message。也能在没有AMQP broker的上面使用。

可以在celery amap后面指定参数来调用API, 或者不指定参数，就进入了shell模式：
```
$ celery -A proj amqp
-> connecting to amqp://guest@localhost:5672/.
-> connected.
1>
```

这里的1是你执行命令个数的编号。可以输入help，这个模式也支持自动补全。

下面是创建队列的例子
```
$ celery -A proj amqp
1> exchange.declare testexchange direct
ok.
2> queue.declare testqueue
ok. queue:testqueue messages:0 consumers:0.
3> queue.bind testqueue testexchange testkey
ok.
```

这里队列名称是testqueue，使用routing key testkey绑定了exchange为testexchange。

现在开始所有发送到exchange  testchange的message，只要她的routing key是testkey，那么他就会进入testqueue这个队列。我们可以使用basic.publish来发送一个message:
```
4> basic.publish 'This is a message!' testexchange testkey
ok.
```

现在message已经发送出去了，你可以使用basic.get命令抽取它了(这个命令指示用来维护task的，正常服务应该使用basic.consume替代)。

从队列中pop出一条message：
```
5> basic.get testqueue
{'body': 'This is a message!',
 'delivery_info': {'delivery_tag': 1,
                   'exchange': u'testexchange',
                   'message_count': 0,
                   'redelivered': False,
                   'routing_key': u'testkey'},
 'properties': {}}
```

AMQP使用ack来确认某个message被消费并且成功执行。如果message还没有被ack，consumer channel就关闭了，那么这个message就会被传递给其他的consumer。

注意传递的tag在上面结构中已经有了。在一个连接channel内，每个接收到的message都有唯一的传递tag，这个tag用来ack这个message。还要注意，传递tag在多个连接之中并不是唯一的，也就是说连接1中的连接tag为1的message是A，连接2中的连接tag为1的message有可能就是B。

使用basic.ack来加上这个标记：
```
6> basic.ack 1
ok.
```

记得要在测试会话完成后清空刚才创建的message：
```
7> queue.delete testqueue
ok. 0 messages deleted.
8> exchange.delete testexchange
ok.
```

Routing Tasks
---

#### 定义队列
下面是一个包含三个队列定义的例子，一个是video，一个是images还有一个默认的队列
```py
default_exchange = Exchange('default', type='direct')
media_exchange = Exchange('media', type='direct')

CELERY_QUEUES = (
    Queue('default', default_exchange, routing_key='default'),
    Queue('videos', media_exchange, routing_key='media.video'),
    Queue('images', media_exchange, routing_key='media.image')
)
CELERY_DEFAULT_QUEUE = 'default'
CELERY_DEFAULT_EXCHANGE = 'default'
CELERY_DEFAULT_ROUTING_KEY = 'default'
```

若没有指定特定路由就都进入CELERY_DEFAULT_QUEUE里。

#### 指定task目的地

按照如下顺序决定message的目的地：
1. 在CELERY_ROUTES中定义的路由
2. Task.apply_async()中指定的路由参数
3. 在Task自身中定义的相关属性

最好不要硬编码这些配置，使用 Routers来读取配置，这是最灵活的方式。但是一些敏感的默认值还是可以被设置为任务属性。

####  Routers
Router类是用来决定某个task的路由选项的。

我们只需要创建一个新的路由类，带有一个route_for_task方法：
```py
class MyRouter(object):

    def route_for_task(self, task, args=None, kwargs=None):
        if task == 'myapp.tasks.compress_video':
            return {'exchange': 'video',
                    'exchange_type': 'topic',
                    'routing_key': 'video.compress'}
        return None
```

如果返回值中包含了 queue，那么会在CELERY_QUEUES里定义的配置上扩展，也就是会继承既有的配置。
```
{'queue': 'video', 'routing_key': 'video.compress'}
```
就变成了
```
{'queue': 'video',
 'exchange': 'video',
 'exchange_type': 'topic',
 'routing_key': 'video.compress'}
```

完事儿后把Router放进CELERY_ROUTES配置信息里就好了。
```
CELERY_ROUTES = (MyRouter(), )
```
或者
```
CELERY_ROUTES = ('myapp.routers.MyRouter', )
```

其实对于上面那种简单的路由规则， 也可以直接写在配置里：
```
CELERY_ROUTES = ({'myapp.tasks.compress_video': {
                        'queue': 'video',
                        'routing_key': 'video.compress'
                 }}, )
```
#### 广播

Celery也支持广播路由。
```
from kombu.common import Broadcast

CELERY_QUEUES = (Broadcast('broadcast_tasks'), )

CELERY_ROUTES = {'tasks.reload_cache': {'queue': 'broadcast_tasks'}}
```

这样，tasks.reload_cache任务就可以发送给每个消费这个队列的worker了。

注意celery的结果并没有定义如果两个任务都有同一个task_id。如果同样的task被分发给多个worker，那么state历史可能也不会被保存。

这样的话，设置task.ignore_result是一个不错的选择。


