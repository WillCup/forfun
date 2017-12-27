
title: django模板语言
date: 2016-12-05 17:08:33
tags: [youdaonote]
---

本文从技术角度解释一下Django template系统，看一下它是怎样运行的，以及怎样继承它。

希望你已经看了前面的文章。

概览
---
在python中使用template系统包含三个步骤：
1. 配置一个[Engine](https://docs.djangoproject.com/en/1.9/ref/templates/api/#django.template.Engine)；
2. 把template代码编译成[Template](https://docs.djangoproject.com/en/1.9/ref/templates/api/#django.template.Template)；
3. 在[Context](https://docs.djangoproject.com/en/1.9/ref/templates/api/#django.template.Context)中渲染这个Template

Django项目一般都是使用[高级API](https://docs.djangoproject.com/en/1.9/topics/templates/#template-engines)来实现上面的步骤。
1. 对于[TEMPLATES](https://docs.djangoproject.com/en/1.9/ref/settings/#std:setting-TEMPLATES)配置的每个[ DjangoTemplates](https://docs.djangoproject.com/en/1.9/topics/templates/#django.template.backends.django.DjangoTemplates)，Django都实例化一个`Engine`. DjangoTemplates包装了一个`Engine`，使它能够适配后台的通用模板API。
2. [django.template.loader](https://docs.djangoproject.com/en/1.9/topics/templates/#module-django.template.loader) moduel有一个`get_template()`方法用来加载template。返回一个包装了[django.template.Template](https://docs.djangoproject.com/en/1.9/ref/templates/api/#django.template.Template)的`django.template.backends.django.Template`.
3. 上一步骤中的`Template`有一个`render()`方法，它又包含一个context，有时也包含一个针对`Context`的request，负责在指定Template进行渲染。


配置engine
---
**class Engine(dirs=None, app_dirs=False, allowed_include_roots=None, context_processors=None, debug=False, loaders=None, string_if_invalid='', file_charset='utf-8', libraries=None, builtins=None)[source](https://docs.djangoproject.com/en/1.9/_modules/django/template/engine/#Engine)**

- **dirs** template文件所在的文件夹们，默认是空
- **loaders** 多个template loader类。[详见](https://docs.djangoproject.com/en/1.9/ref/templates/api/#template-loaders)
- **app_dirs** 只在**loaders**里配置了'django.template.loaders.app_directories.Loader' 才有效果
- **libraries**


加载Template
---
创建`Template`的推荐方式是调用`Engine`的`get_template()`, `select_template()`或者`from_string()`.

在django项目里是只能配置一个Template的，但我们也可以脱离配置直接使用Template。
```python
from django.template import Template

template = Template("My name is {{ my_name }}.")
```

渲染context
---
编译好Template之后，就可以用它来渲染任意的context了。
```python
>>> from django.template import Context, Template
>>> template = Template("My name is {{ my_name }}.")

>>> context = Context({"my_name": "Adrian"})
>>> template.render(context)
"My name is Adrian."

>>> context = Context({"my_name": "Dolores"})
>>> template.render(context)
"My name is Dolores."
```
调用Template的render方法来渲染context。

变量
---
变量只能是字母、数字、下划线、点。
点有个特殊的含义，叫做lookup。每次遇到点的时候，template系统就会按照以下此次序进行lookup：
- 字典lookup。例子：foo['sdf']
- 属性lookup。例子：foo.bar
- 数组索引lookup。例子：frr[sdf]
```python
>>> from django.template import Context, Template
>>> t = Template("My name is {{ person.first_name }}.")
>>> d = {"person": {"first_name": "Joe", "last_name": "Johnson"}}
>>> t.render(Context(d))
"My name is Joe."

>>> class PersonClass: pass
>>> p = PersonClass()
>>> p.first_name = "Ron"
>>> p.last_name = "Nasty"
>>> t.render(Context({"person": p}))
"My name is Ron."

>>> t = Template("The first stooge in the list is {{ stooges.0 }}.")
>>> c = Context({"stooges": ["Larry", "Curly", "Moe"]})
>>> t.render(c)
"The first stooge in the list is Larry."
```
当某个变量可以被调用的时候，都会被尝试调用。例如
```python
>>> class PersonClass2:
...     def name(self):
...         return "Samantha"
>>> t = Template("My name is {{ person.name }}.")
>>> t.render(Context({"person": PersonClass2}))
"My name is Samantha."
```
可以被调用的变量比直接查找lookup的变量稍微复杂点儿。记住下面几条
- 如果返回异常的话，会被传递上去，除非设置了`silent_variable_failure `为True，设置这个属性的话，当异常发生时，就会使用engine配置的`string_if_invalid `选项；
```python
>>> t = Template("My name is {{ person.first_name }}.")
>>> class PersonClass3:
...     def first_name(self):
...         raise AssertionError("foo")
>>> p = PersonClass3()
>>> t.render(Context({"person": p}))
Traceback (most recent call last):
...
AssertionError: foo

>>> class SilentAssertionError(Exception):
...     silent_variable_failure = True
>>> class PersonClass4:
...     def first_name(self):
...         raise SilentAssertionError
>>> p = PersonClass4()
>>> t.render(Context({"person": p}))
"My name is ."
```
- 只能调用无参函数
- 显然如果前端能调用的话，就可能发生side effect，所以要确保措施。比如{{ data.delete }}

要解决这个问题，就需要在敏感方法上加上`alters_data`属性，django默认动态生成的model对象都自动设置了这个属性
```python
def sensitive_function(self):
    self.database_record.delete()
sensitive_function.alters_data = True
```
- 有时我们想关闭这个属性，那么久告诉template系统不要调用。在这个属性上设置`do_not_call_in_templates `为True。跟上面的区别是，上面会直接返回`string_if_invalid`，而这个还是可以访问，但是不能调用，也就是当变量而不是当方法使用。



