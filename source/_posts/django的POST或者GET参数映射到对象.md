title: django的POST或者GET参数映射到对象
date: 2016-07-21 19:50:59
tags:
---


原理及其简单，就算利用python的自省，有点儿类似java的反射调用。
```python
etl = ETL()
        dic = dict(request.POST)
        for k,v in dic.iteritems():
            if hasattr(etl, k):
                setattr(etl, k, v)
        print etl
```
就这几行代码就可以搞定了。为了公用方便，可以抽出来作为一个module使用。


简单封装了以下
```python
# -*- coding: utf-8 -*
import logging

'''
http的一些常用的工具方法
'''


def post2obj(obj, post, *excludes):
    '''
    将post里的参数对应copy到obj里，主要是方便http接口直接使用obj进行操作
    :param obj:
    :param post:
    :return:
    '''
    dic = dict(post)
    for k, v in dic.iteritems():
        if k not in excludes:
            if hasattr(obj, k):
                v = ''.join(v)
                setattr(obj, k, v)
    logging.info("post params has been copied to  obj -> %s" % obj)

...
...
get 同理
```

`注意`：上面我们使用了''.join(v)。因为从dict里遍历出来的所有value都是list类型的。而`request.POST['xxx']`取出来的就是正常的unicode类型。list会在后面的处理中出现很多问题，所以这里转成str再使用。

再想想，其实可以通过python的装饰器【对应java里的注释】实现，但是目前不太确定python是否支持java的泛型类的东西，可以用来自动生成操作对象的实例。
