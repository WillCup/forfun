
title: django-celery坑记
date: 2016-12-05 17:05:43
tags: [youdaonote]
---


安装django-celery花了好长时间，因为网上一些资料都不是最新版本，缺失一些配置。这里自己再整理一下整体的步骤。

#### 1.安装：
```
pip install django-celery
```

#### 2. settings文件
```py
INSTALLED_APPS = [
    'metamap',
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'djcelery',
]


import djcelery
djcelery.setup_loader()

# 用于存放task
BROKER_URL = 'redis://localhost:6379'


# Celery Beat 设置
CELERYBEAT_SCHEDULER = 'djcelery.schedulers.DatabaseScheduler'
```
注意一下，这里的INSTALLED_APPS，在[django1.9版本的官方文档](https://docs.djangoproject.com/en/1.9/intro/tutorial02/)中，是说写成`metamap.apps.MetamapConfig`的。但是写成这种的话，下面自动发现task的时候就会找不到。可以理解成是djcelery暂时还没有追上django版本的脚步，好在django兼容了以前版本，我们写成metamap也不会影响django1.9版本的运行。

#### 3. celery启动
启动celery的文件，有些博客少了这个主要部分。
```py
# !/usr/bin/env python
from __future__ import absolute_import

import os

from celery import Celery

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'metamap_django.settings')

from django.conf import settings

app = Celery('metamap_django')

# Using a string here means the worker will not have to
# pickle the object when using Windows.
# 可以发现，这里有一个自动发现task的配置，就是针对settings里的INSTALLED_APPS，这里我们的目标是metamap
# 找到metamap后，会自动找下面的tasks.py文件，扫描里面的task
app.config_from_object('django.conf:settings')
app.autodiscover_tasks(lambda: settings.INSTALLED_APPS) 


@app.task(bind=True)
def debug_task(self):
    print('Request: {0!r}'.format(self.request))


```

#### 4. 启动celery
在metamap/__init__.py里引入celery_app:
```py
from __future__ import absolute_import

# This will make sure the app is always imported when
# Django starts so that shared_task will use this app.
from .celery import app as celery_app
```


#### 5. tasks.py
放到app的根目录下面，我的是metamap/tasks.py：
```py
# !/usr/bin/env python
# -*- coding: utf-8 -*
'''
created by will
'''

from __future__ import absolute_import

import logging
import subprocess

from celery import shared_task, task
from django.utils import timezone

from metamap.models import ETL, Executions
from metamap.utils import enums

from celery.utils.log import get_task_logger

logger = get_task_logger(__name__)

@task
def xx():
    return 'sdfsdf'

@shared_task
def add(x, y):
    return x + y


@shared_task
def mul(x, y):
    return x * y


@shared_task
def xsum(numbers):
    return sum(numbers)

```



参考：
- http://www.weiguda.com/blog/73/
- https://github.com/hardvic/djcelery_proj
- https://github.com/celery/django-celery
