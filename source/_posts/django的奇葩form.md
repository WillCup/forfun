title: django的奇葩form
date: 2016-07-21 18:11:22
tags:
---
记录一下遇到的最奇葩的组件：django的form插件。官方文档是：https://docs.djangoproject.com/ja/1.9/ref/forms/widgets/。

弄这个东西的初衷场景：python里暂时没有找到类似java里类似下面的功能。

```java
@RequestMapping(value = "save",method = RequestMethod.POST, produces = MediaType.APPLICATION_JSON)
    public @ResponseBody Object addETL(ETL etl) throws Exception{
        etl.setAuthor("will");
        etlService.addETL(etl);
        return "{\"message\" :\"success\"}";
    }
```
调用这个接口的时候，只需要传递ETL的字段就可以了，不管是POST还是GET方法，都会自己序列化进入ETL对象，然后就可以操作client端传送上来的ETL对象了 。
但是在python或者django里本人暂时没有找到这种方便的方式，只能通过request.POST['name']之类的方法，挨个从POST对象里面取出来，再自己用这些属性去初始化ETL对象，之后再去执行save或者等等之类的操作....身为一个优秀的程序员，必须将“懒”的特性发回出来。所以就去django官网找能够实现类似功能的例子，所以就找到了django的form组件。

额外发现是，当我们需要编辑一个已经存在的ETL的时候，可以自动把所有字段渲染到前端。

那么满怀期待地开始了恶心的form探索之旅。

前端
---
```html
<form action="etls/add" method="POST">
    {% csrf_token %}
        {{ form.as_p }}
        <input type="submit"/>
    </form>
```
简直是简单的不要不要的。

后端
views文件里定义一个Form。
---
自定义一个form
```python

class ETLForm(forms.ModelForm):
    preSql = forms.CharField(widget=forms.Textarea(attrs={'class': "form-control", "size": 10}))

    class Meta:
        model = ETL
        exclude = ['id', 'ctime']
```
上面这段，定义了preSql这个字段的显示类型为form的textarea，然后定义来这个textarea的属性，包括class样式。
Meta里定义了model的数据模型，还有要去除不显示的字段。也可以指定fields，就是要显示的字段。 怎么方便怎么来就是了。

其实恶心的东西就是这里写的样式了。好，就算能够接受后端把逻辑和样式一起写了。可是对于不是form的样式怎么办呢，有很多页面就不是form啊，比如列表～会有很多样式必须需要在前端写。那样就是前端有样式，后端也有样式，麻蛋，想想头就大。

其实是有种更方便的做法
```python
Form = modelform_factory(Author, form=AuthorForm, localized_fields=("birth_date",))
```
鉴于本人对上面这种后端写样式就已经感觉不能接受了，就没有再继续搞这个东西。

```python
@transaction.atomic
def edit(request, pk):
    if request.method == 'POST':
        form = ETLForm(request.POST)
        if form.is_valid():


            new_etl = form.save()
            logger.info('ETL has been created successfully : ' + new_etl)
            return HttpResponseRedirect(reverse('metamap:index'))
    else:
        etl = ETL.objects.get(pk=pk)
        form = ETLForm(instance=etl)
        return render(request, 'etl/edit.html', {'form': form})
```

路由配置是
```python
 url('etls/(?P<pk>[0-9]+)/$', etls.edit),
```


![前端效果图](/imgs/django/django-form.png)


记录以下这个奇葩组件，然后使用传统form进行逻辑编写，就算是要到POST里逐个取值，也认了！！！！

widget相关：https://docs.djangoproject.com/ja/1.9/ref/forms/widgets/
