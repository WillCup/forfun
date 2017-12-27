
title: metamap部署
date: 2016-12-05 17:10:52
tags: [youdaonote]
---

vituralEnv
---

先创建执行环境
```
virtualenv metamap
```
这个命令会在当前目录下生成一个metamap目录，后来查实了一下，其实后面是指定要生成的目标记录的(`virtualenv /path/to/tartget/metamap`)，便于统一管理所有的env环境。

进入我们刚刚创建的环境:
```
root@will-vm:/usr/local/metamap# source metamap/bin/activate
(metamap) root@will-vm:/usr/local/metamap# 
```

我这里没有使用绝对路径，所以就成了当前目录。目录结构如下：
```
(metamap) root@will-vm:/usr/local/metamap# tree -L 2 metamap
metamap
├── bin
│   ├── activate
│   ├── activate.csh
│   ├── activate.fish
│   ├── activate_this.py
│   ├── django-admin
│   ├── django-admin.py
│   ├── django-admin.pyc
│   ├── easy_install
│   ├── easy_install-2.7
│   ├── pip
│   ├── pip2
│   ├── pip2.7
│   ├── python
│   ├── python2 -> python
│   ├── python2.7 -> python
│   ├── python-config
│   └── wheel
├── include
│   └── python2.7 -> /usr/include/python2.7
├── lib
│   └── python2.7
├── local
│   ├── bin -> /usr/local/metamap/metamap/bin
│   ├── include -> /usr/local/metamap/metamap/include
│   └── lib -> /usr/local/metamap/metamap/lib
└── pip-selfcheck.json


```
另外还会在新的env中安装三个基础依赖包：
```
(metamap) root@will-vm:/usr/local/metamap# pip list
pip (8.1.2)
setuptools (25.1.3)
wheel (0.29.0)

```


使用pip freeze > requirements.txt来导出当前环境中的所有python依赖。

然后使用pip install -r path/to/requirements.txt安装，发现有一些是跟os有关的，比如包含Ubuntu的，根本就没有必要安装。所以就只保留了有印象的几个依赖。

最后requirements.txt只剩下：
```
tlr4-python2-runtime==4.5.3
Django==1.9.7
MySQL-python==1.2.5
pyhs2==0.6.0
```

如果要退出virtualenv，就使用deactivate。


Gunicorn
---
在vituralenv里安装
```
pip install gunicorn
```

进入我们的django项目根目录，也就是manage.py所在的目录：
```
(metamap) root@will-vm:/usr/local/metamap/metamap_django# gunicorn_django -b 0.0.0.0:8089
!!!
!!! WARNING: This command is deprecated.
!!! 
!!!     You should now run your application with the WSGI interface
!!!     installed with your project. Ex.:
!!! 
!!!         gunicorn myproject.wsgi:application
!!! 
!!!     See https://docs.djangoproject.com/en/1.8/howto/deployment/wsgi/gunicorn/
!!!     for more info.
!!!

[2016-08-03 19:13:32 +0000] [13020] [INFO] Starting gunicorn 19.6.0
[2016-08-03 19:13:32 +0000] [13020] [INFO] Listening at: http://0.0.0.0:8089 (13020)
[2016-08-03 19:13:32 +0000] [13020] [INFO] Using worker: sync
[2016-08-03 19:13:32 +0000] [13025] [INFO] Booting worker with pid: 13025
[2016-08-03 19:13:32 +0000] [13025] [ERROR] Exception in worker process
Traceback (most recent call last):
  File "/usr/local/metamap/metamap/local/lib/python2.7/site-packages/gunicorn/arbiter.py", line 557, in spawn_worker
    worker.init_process()
  File "/usr/local/metamap/metamap/local/lib/python2.7/site-packages/gunicorn/workers/base.py", line 126, in init_process
    self.load_wsgi()
  File "/usr/local/metamap/metamap/local/lib/python2.7/site-packages/gunicorn/workers/base.py", line 136, in load_wsgi
    self.wsgi = self.app.wsgi()
  File "/usr/local/metamap/metamap/local/lib/python2.7/site-packages/gunicorn/app/base.py", line 67, in wsgi
    self.callable = self.load()
  File "/usr/local/metamap/metamap/local/lib/python2.7/site-packages/gunicorn/app/djangoapp.py", line 105, in load
    mod = util.import_module("gunicorn.app.django_wsgi")
  File "/usr/lib/python2.7/importlib/__init__.py", line 37, in import_module
    __import__(name)
  File "/usr/local/metamap/metamap/local/lib/python2.7/site-packages/gunicorn/app/django_wsgi.py", line 21, in <module>
    from django.core.management.validation import get_validation_errors
ImportError: No module named validation
[2016-08-03 19:13:32 +0000] [13025] [INFO] Worker exiting (pid: 13025)
[2016-08-03 19:13:32 +0000] [13020] [INFO] Shutting down: Master
[2016-08-03 19:13:32 +0000] [13020] [INFO] Reason: Worker failed to boot.

```

查了一下，是版本问题，改用
```
(metamap) root@will-vm:/usr/local/metamap/metamap_django# gunicorn metamap_django.wsgi:application -b 0.0.0.0:8089
[2016-08-03 19:16:10 +0000] [13097] [INFO] Starting gunicorn 19.6.0
[2016-08-03 19:16:10 +0000] [13097] [INFO] Listening at: http://0.0.0.0:8089 (13097)
[2016-08-03 19:16:10 +0000] [13097] [INFO] Using worker: sync
[2016-08-03 19:16:10 +0000] [13102] [INFO] Booting worker with pid: 13102
```
wsgi为我们生成django项目时，跟settings.py同目录的wsgi.py。然后就可以到浏览器访问了。

看一下我们django自动生成的这个文件内容，可以看到还有：
```python
import os

from django.core.wsgi import get_wsgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "metamap_django.settings")

application = get_wsgi_application()

def get_wsgi_application():
    """
    The public interface to Django's WSGI support. Should return a WSGI
    callable.

    Allows us to avoid making django.core.handlers.WSGIHandler public API, in
    case the internal WSGI implementation changes or moves in the future.
    """
    django.setup()
    return WSGIHandler()
```

晚上整体浏览了一遍Gunicorn官网，整理以下几点注意的地方：
 - 可以指定django的settings文件
 - gunicorn 官网中指定的几个管理master和worker的signal其实都是Linux的kill命令的参数 `kill -TERM cat /tmp/xx.pid`
 - worker和thread是不一样的概念，前者是进程，后者是线程，worker中包含thread。具体并发数需要压测结果来确定
 - 为了防止出现内存泄露问题，可以为worker指定类似max_request之类的参数在一定阶段重启当前worker
 - Gunicorn可以动态增加worker实现调优
 - 命令行可以加入运行目录并生成pid
 - 环境中添加python插件setproctitle之后就可以为gunicorn进程命名了
 ```
 (metamap) root@will-vm:/usr/local/metamap/metamap_django# ps -ef | grep gunicorn
root      2258 10709  0 11:38 pts/26   00:00:00 tailf /tmp/gunicorn_error.log
root      2494  2264  0 11:38 pts/27   00:00:00 tailf /tmp/gunicorn_access.log
root      3190  2444  0 11:57 ?        00:00:00 gunicorn: master [will's metamap]                                                                                                    
root      3195  3190  2 11:57 ?        00:00:00 gunicorn: worker [will's metamap]                                                                                                    
root      3198  3190  2 11:57 ?        00:00:00 gunicorn: worker [will's metamap]                                                                                                    
root      3199  3190  2 11:57 ?        00:00:00 gunicorn: worker [will's metamap]                                                                                                    
root      3205  3190  2 11:57 ?        00:00:00 gunicorn: worker [will's metamap]                                                                                                    
root      3207  3190  2 11:57 ?        00:00:00 gunicorn: worker [will's metamap]                                                                                                    
root      3223 17309  0 11:57 pts/25   00:00:00 grep --color=auto gunicorn

 ```
 - --statsd-host=localhost:8125会启动一个针对gunicorn的监控系统
 - master只负责管理worker，不处理请求，worker是真正处理请求的进程
 - 可接受配置文件-c, 具体可配置项，参考[这里](http://docs.gunicorn.org/en/latest/settings.html)

```python
# !/usr/bin/env python
# -*- coding: utf-8 -*
'''
created by will
'''

import multiprocessing

print('gunicorn config is running....')

bind = "0.0.0.0:8088"
workers = multiprocessing.cpu_count() * 2 + 1
pidfile = '/tmp/gunicorn.pid'
accesslog = '/tmp/gunicorn_access.log'
errorlog = '/tmp/gunicorn_error.log'
loglevel = 'info'
capture_output = True
statsd_host = '0.0.0.0:8888'
proc_name = 'will\'s metamap'
daemon = True
```

然后运行的时候指定此文件的位置即可。

注意：
 - 启动指定wsgi位置的时候，需要是`gunicorn metamap_django.wsgi:application`，要注意的是metamap_django与wsgi之间是`.`，而不是能是文件路径`/`，否则启动gunicorn的时候报错:import by filename is not supported gunicorn
 - 执行 kill -HUP cat /tmp/xx/pid来管理master和worker进程
 - 如果要使用supervisord管理进程的话，最好就不要设置`daemon`为True了
 - 有些东西跟官网说的不一致，比如: statsd_host，django_settings指定settings文件位置，access-log等




参考：
- http://ju.outofmemory.cn/entry/105202
- http://zqpythonic.qiniucdn.com/data/20130901152951/index.html


Django
---
注意一下几点：

1. 为django创建多个settings文件，如prod.py,test.py等,之后在gunicorn的配置文件里指定需要的配置文件即可
```python
# !/usr/bin/env python
# -*- coding: utf-8 -*
'''
created by will
'''

import multiprocessing, os

print('gunicorn config is running....')

bind = "0.0.0.0:8088"
workers = multiprocessing.cpu_count() * 2 + 1
pidfile = '/tmp/gunicorn.pid'
# accesslog = '/tmp/gunicorn_access.log'
errorlog = '/tmp/gunicorn_error.log'
loglevel = 'info'
capture_output = True
# statsd_host = 'localhost:8077'
proc_name = 'will\'s metamap'
daemon = True
# 测试确定，这里设置这个参数不生效
# django_settings = 'metamap.config.prod'

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "metamap.config.prod")

```
2. 修改settings里的几个选项

参数|解释
---|---
DEBUG | 改为False，不然程序出错的方法栈会直接展示在前端用户那里，不友好。
ALLOWED_HOSTS | 添加可以访问你的项目的几个域名或主机地址，防止[ HTTP Host header attacks](https://docs.djangoproject.com/es/1.9/topics/security/#host-headers-virtual-hosting)
ADMINS | 当程序出错的时候django会发送错误给这些邮件列表

参考：https://docs.djangoproject.com/es/1.9/topics/settings/

3. 确认有处理view里抛出的异常的逻辑，如果没有的话，自定义一个处理异常的middleware，注册到setting离去。

参考：https://docs.djangoproject.com/ja/1.9/topics/http/middleware/

4. 将django的所有的静态文件整理到指定文件夹
在settings.py里设置static_root，然后执行`./manage.py collectstatic`，这个命令就会把所有的静态资源整理到static_root里面。

参考：https://docs.djangoproject.com/en/1.10/howto/static-files/#deployment

Nginx
---
nginx的主要功能是分发动态请求，以及处理静态资源请求。

配置文件
```

#user  nobody;
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /tmp/nginx.access.log combined;

    sendfile        on;
    
    keepalive_timeout  65;

    #gzip  on;

     upstream app_server {

        # 指定后台地址
        server 192.168.244.128:8088 fail_timeout=0;
      }

      server {
        listen 80;
        client_max_body_size 4G;

        # 指定域名
        server_name 192.168.244.128;

        keepalive_timeout 5;

        # static文件夹所在的根目录
        root /usr/local/metamap;

        location / {
          # 如果是静态资源就自己处理，如果不是就发送给proxy_to_app
          # try_files->按顺序检查文件是否存在，返回第一个找到的文件或文件夹（结尾加斜线表示为文件夹），如果所有的文件或文件夹都找不到，会进行一个内部重定向到最后一个参数
          try_files $uri @proxy_to_app;
        }

        location @proxy_to_app {
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          # enable this if and only if you use HTTPS
          # proxy_set_header X-Forwarded-Proto https;
          proxy_set_header Host $http_host;
          # we don't want nginx trying to do something clever with
          # redirects, we set the Host: header above already.
          proxy_redirect off;
          proxy_pass http://app_server;
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
          root html;
        }
      }



}

```




未解决的问题
---
不定时出现找不到静态资源的问题：
```
2016-08-04 10:04:10,128 [MainThread:-1219164416] [django.request:182] [base:get_response] [WARNING]- Not Found: /static/js/bootstrap.js
2016-08-04 10:04:10,126 [MainThread:-1219164416] [django.request:182] [base:get_response] [WARNING]- Not Found: /static/js/jquery.js
2016-08-04 10:04:10,148 [MainThread:-1219164416] [django.request:182] [base:get_response] [WARNING]- Not Found: /static/css/bootstrap.css
2016-08-04 10:04:10,140 [MainThread:-1219164416] [django.request:182] [base:get_response] [WARNING]- Not Found: /static/style.css
```
