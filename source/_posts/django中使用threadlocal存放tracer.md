title: django中使用threadlocal存放tracer
date: 2018-01-19 15:46:07
tags: [django, jaeger, 架构]
---

基于上篇《服务trace的一些理解》，想办法把django的view里的tracer和model.pre_save的tracer通过threadlocal统一起来。


结合gunicorn考虑一下，其实如果能保证gunicorn的每个worker都是单线程的话，就不存在这种问题，只要弄一个静态的tracer就可以了。

#### 思路
- middleware从threadlocal引入tracer，打在request层面，这里会有第一个span
- redis/db的关键API通过`jaeger_tracer`装饰器方式获得threadlocal里的tracer，新增一个child span


#### 代码

middleware沿用`django_opentracing`的使用方式，修改生成tracer的方式为每个thread一个tracer，因为worker可能是多线程的，也方便后面处理逻辑获取当前线程中既有的tracer：
```py
try:
    # Django >= 1.10
    from django.utils.deprecation import MiddlewareMixin
except ImportError:
    # Not required for Django <= 1.9, see:
    # https://docs.djangoproject.com/en/1.10/topics/http/middleware/#upgrading-pre-django-1-10-style-middleware
    MiddlewareMixin = object


class WillOpenTracingMiddleware(MiddlewareMixin):
    '''
    __init__()只会被调用一次，没有参数，只有server对第一个request生成response的时候被调用一次
    '''

    def __init__(self, get_response=None):
        '''
        TODO: ANSWER Qs
        - Is it better to place all tracing info in the settings file, or to require a tracing.py file with configurations?
        - Also, better to have try/catch with empty tracer or just fail fast if there's no tracer specified
        '''
        self.get_response = get_response

    def process_view(self, request, view_func, view_args, view_kwargs):
        # 如果配置中并不是trace所有request，这个middleware就不会有任何作为，实际trace应该交由decorator来做
        if not threadlocals.get_current_tracer()._trace_all:
            return None

        if hasattr(settings, 'OPENTRACING_TRACED_ATTRIBUTES'):
            traced_attributes = getattr(settings, 'OPENTRACING_TRACED_ATTRIBUTES')
        else:
            traced_attributes = []
        threadlocals.set_thread_variable('request', request)
        threadlocals.get_current_tracer()._apply_tracing(request, view_func, traced_attributes)

    def process_response(self, request, response):
        threadlocals.get_current_tracer()._finish_tracing(request)
        return response

```

`django_opentracing`中的`DjangoTracer`剖析一下:
```py
class DjangoTracer(object):
    '''
    是对一种tracer的wrapper
    '''
    def __init__(self, tracer=None):
        self._tracer_implementation = None
        if tracer:
            self._tracer_implementation = tracer
        self._current_spans = {}
        if not hasattr(settings, 'OPENTRACING_TRACE_ALL'):
            self._trace_all = False
        elif not getattr(settings, 'OPENTRACING_TRACE_ALL'):
            self._trace_all = False
        else:
            self._trace_all = True

    @property
    def _tracer(self):
        if self._tracer_implementation:
            return self._tracer_implementation
        else:
            return opentracing.tracer

    def get_span(self, request):
        '''
        @param request
        返回trace给定request的span
        '''
        return self._current_spans.get(request, None)
        
    def trace(self, *attributes):
        '''
        这是上面提到的Function decorator
        注意:必须在 @app.route decorator后面
        @param attributes any number of flask.Request attributes
        (strings) to be set as tags on the created span
        '''
        def decorator(view_func):
            # TODO: do we want to provide option of overriding trace_all_requests so that they
            # can trace certain attributes of the request for just this request (this would require
            # to reinstate the name-mangling with a trace identifier, and another settings key)
            # 如果是_trace_all，就直接返回，原因参考上面的middleware的逻辑
            if self._trace_all:
                return view_func
            # otherwise, execute decorator
            def wrapper(request):
                span = self._apply_tracing(request, view_func, list(attributes))
                r = view_func(request)
                self._finish_tracing(request)
                return r
            return wrapper
        return decorator
    
    # 实际trace逻辑
    def _apply_tracing(self, request, view_func, attributes):
        '''
        避免middleware和decorator之间出现重写。返回request的新的span和view_func的正确的operation名字
        '''
        # strip headers for trace info
        headers = {}
        for k,v in request.META.iteritems():
            k = k.lower().replace('_','-')
            if k.startswith('http-'):
                k = k[5:]
            headers[k] = v

        # start new span from trace info
        span = None
        operation_name = view_func.__name__
        try:
            span_ctx = self._tracer.extract(opentracing.Format.HTTP_HEADERS, headers)
            span = self._tracer.start_span(operation_name=operation_name, child_of=span_ctx)
        except (opentracing.InvalidCarrierException, opentracing.SpanContextCorruptedException) as e:
            span = self._tracer.start_span(operation_name=operation_name)
        if span is None:
            span = self._tracer.start_span(operation_name=operation_name)

        # add span to current spans
        # 把request与span的对应关系存进这个DjangoTracer的一个字典里
        # 按执行过程推理，如果有request并发的话，保证span不会错乱，跟threadlocal的作用差不多，但是需要额外的request参数才行
        self._current_spans[request] = span

        # log any traced attributes
        for attr in attributes:
            if hasattr(request, attr):
                payload = str(getattr(request, attr))
                if payload:
                    span.set_tag(attr, payload)

        return span

    # 将所有span结束
    def _finish_tracing(self, request):
        span = self._current_spans.pop(request, None)
        if span is not None:
            span.finish()
    
    
```
我们的思路是使用threadlocal实现，原因上面也提到了，threadlocal不用给方法传递类似request这种额外的参数。对于后端redis/mysql等会很方便。

下面是一个简单的使用`DjangoTracer`的threadlocal实现简版：
```py
# -*- coding: utf-8 -*-
"""
__init__ module for the threadlocals package
"""
import opentracing
from django_opentracing import DjangoTracer
from jaeger_client import Config
from threading import local

__docformat__ = "restructuredtext"

_threadlocals = local()

def set_thread_variable(key, val):
    setattr(_threadlocals, key, val)


def get_thread_variable(key, default=None):
    return getattr(_threadlocals, key, default)

# 通过threadlocal获取当前线程的tracer
def get_current_tracer():
    if not hasattr(_threadlocals, 'tracer'):
        config = Config(
            config={
                'sampler': {
                    'type': 'const',
                    'param': 1
                },
                'local_agent': {
                    'reporting_host': '10.103.27.152'
                }
            },
            service_name='will-django'
        )
        set_request_variable('tracer', DjangoTracer(config.initialize_tracer()))
    return get_thread_variable('tracer')
```

自定义`jaeger_tracer`装饰器：
```py
def jaeger_tracer(service='will'):
    def _my_decorator(func):
        def _decorator(*args, **kwargs):
            with get_current_tracer()._tracer.start_span(os.path.basename(inspect.getsourcefile(func))[:-2] + func.func_name,
                                                         child_of=get_current_tracer().get_span(
                                                                 get_current_request())) as ttspan:
                # ttspan.set_tag(ext_tags.SPAN_KIND, ext_tags.SPAN_KIND_RPC_CLIENT)
                # ttspan.set_tag(ext_tags.PEER_SERVICE, service)
                for k, v in kwargs.items():
                    ttspan.set_tag(k, v)
                ttspan.set_tag('args', args[1:])
                ttspan.log_event('this is done by will')
                return func(*args, **kwargs)
        return _decorator
    return _my_decorator
```

使用例子：
```py
@jaeger_tracer('redis')
def get_keys():
    r = redis.StrictRedis(connection_pool=pool)
    return [key for key in r.keys() if '_' not in key and 'unack' not in key]
```

![产出的jaeger trace效果图](/imgs/trace/django_jaeger_redis_tracer.png)

#### 不足
- 只支持两层span，request一层，然后redis一层。如果还有多层，需要在current_span那里下点儿功夫了
- djangoTracer的字典是可有可无的，因为不再是针对request的，而是针对thread的



参考：
- https://github.com/shtalinberg/django-actions-logger/blob/master/actionslog/middleware.py
- https://github.com/benoitc/gunicorn/issues/1045
