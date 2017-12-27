
title: django中模型之间的关系
date: 2016-12-05 17:05:17
tags: [youdaonote]
---

Many-to-One
---
使用`django.db.models.ForeignKey`来定义，像其他的字段类型一样，额外需要一个指定model类的位置。

例如，有一个Car model，它的生产厂家是Manufacturer，也就是一个厂家对应多个车，但是每个车只属于一个生产厂家。
```python
from django.db import models

class Manufacturer(models.Model):
    # ...
    pass

class Car(models.Model):
    manufacturer = models.ForeignKey(Manufacturer, on_delete=models.CASCADE)
    # ...
```

除此之外，也可以创建[递归关系](https://docs.djangoproject.com/en/1.9/ref/models/fields/#recursive-relationships)以及[未定义模型的关系](https://docs.djangoproject.com/en/1.9/ref/models/fields/#lazy-relationships)。详见[此处](https://docs.djangoproject.com/en/1.9/ref/models/fields/#ref-foreignkey)。

建议把`ForeignKey`字段设置成目标model的小写形式，弄成别的也不会出错，但是最好规范一些。

对应数据库里面生成的字段名是`manufacturer_id`，就是在我们定义的属性的名字后面缀上`_id`。

```python
class Car(models.Model):
    company_that_makes_it = models.ForeignKey(
        Manufacturer,
        on_delete=models.CASCADE,
    )
    # ...
```
更多[实例](https://docs.djangoproject.com/en/1.9/topics/db/examples/many_to_one/)

Many-To-Many
---
使用[`ManyToManyField`](https://docs.djangoproject.com/en/1.9/ref/models/fields/#django.db.models.ManyToManyField)关键字定义多对多关系。

例如，一个Pizza有多个Topping，一个Topping也可以放在多个pizza上。

```python
from django.db import models

class Topping(models.Model):
    # ...
    pass

class Pizza(models.Model):
    # ...
    toppings = models.ManyToManyField(Topping)
```

以上无论哪个对象里放置`ManyToManyFiled`都可以，但是只能放在其中一个里面，不能都放。

两者的互相访问
```python
pizza.toppings
topping.pizza_set
```
更多[实例](https://docs.djangoproject.com/en/1.9/topics/db/examples/many_to_many/)


