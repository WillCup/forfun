
title: django使用记录
date: 2016-12-05 17:05:00
tags: [youdaonote]
---

#### Ubuntu安装失败
```
root@will-vm:/usr/local/metamap/metamap_js# pip install Django
Downloading/unpacking Django
  Downloading Django-1.9.7-py2.py3-none-any.whl (6.6MB): 6.6MB downloaded
Cleaning up...
Exception:
Traceback (most recent call last):
  File "/usr/lib/python2.7/dist-packages/pip/basecommand.py", line 122, in main
    status = self.run(options, args)
  File "/usr/lib/python2.7/dist-packages/pip/commands/install.py", line 278, in run
    requirement_set.prepare_files(finder, force_root_egg_info=self.bundle, bundle=self.bundle)
  File "/usr/lib/python2.7/dist-packages/pip/req.py", line 1259, in prepare_files
    )[0]
IndexError: list index out of range

Storing debug log for failure in /root/.pip/pip.log

```
查了一下，貌似只有Ubuntu有这个问题，需要先安装distribute
```
pip install --no-use-wheel --upgrade distribute
```

之后再安装django就解决了
```
root@will-vm:/usr/local/metamap/metamap_js# python -c "import django; print(django.get_version())"
1.9.7
```

参考：
https://www.zhihu.com/question/38484255
