
title: celery数据库事务问题
date: 2016-12-05 17:06:27
tags: [youdaonote]
---

我的场景跟官网的例子基本类似，就是单个事务中需要，先创建一个对象，之后执行异步任务，这个异步任务里又要更新这个新建的对象。

理论上，如果任何一步出问题的话，就应该回滚全部。但是官网给的解释，目前不是这样的：

view代码：
```py
from django import forms
from django.http import HttpResponseRedirect
from django.template.context import RequestContext
from django.shortcuts import get_object_or_404, render_to_response

from blog import tasks
from blog.models import Comment


class CommentForm(forms.ModelForm):

    class Meta:
        model = Comment


def add_comment(request, slug, template_name='comments/create.html'):
    post = get_object_or_404(Entry, slug=slug)
    remote_addr = request.META.get('REMOTE_ADDR')

    if request.method == 'post':
        form = CommentForm(request.POST, request.FILES)
        if form.is_valid():
            # 这里创建对象，对DB有了插入操作
            comment = form.save()
            # 异步方法的调用
            tasks.spam_filter.delay(comment_id=comment.id,
                                    remote_addr=remote_addr)
            return HttpResponseRedirect(post.get_absolute_url())
    else:
        form = CommentForm()

    context = RequestContext(request, {'form': form})
    return render_to_response(template_name, context_instance=context)
```

task代码：
```py
from celery import Celery

from akismet import Akismet

from django.core.exceptions import ImproperlyConfigured
from django.contrib.sites.models import Site

from blog.models import Comment


app = Celery(broker='amqp://')


@app.task
def spam_filter(comment_id, remote_addr=None):
    logger = spam_filter.get_logger()
    logger.info('Running spam filter for comment %s', comment_id)

     # 可以看到异步方法调用依赖于前面对象的创建
    comment = Comment.objects.get(pk=comment_id)
    current_domain = Site.objects.get_current().domain
    akismet = Akismet(settings.AKISMET_KEY,
                      'http://{0}'.format(current_domain))
    if not akismet.verify_key():
        raise ImproperlyConfigured('Invalid AKISMET_KEY')


    is_spam = akismet.comment_check(user_ip=remote_addr,
                        comment_content=comment.comment,
                        comment_author=comment.name,
                        comment_author_email=comment.email_address)
    if is_spam:
        comment.is_spam = True
        # 异步方法调用完成后，会基于前面创建的对象，再次操作数据库对象
        comment.save()

    return is_spam
```

这明显是已经破坏了这个事务的原子性的，虽然目前本人的代码也是这样的。


上面还有一段另外的写法：
```py
@transaction.commit_manually
def create_article(request):
    try:
        article = Article.objects.create(…)
    except:
        transaction.rollback()
        raise
    else:
        transaction.commit()
        expand_abbreviations.delay(article.pk)
```

为什么要这样呢？为了避免race condition。也就是异步方法调用先于数据库插入操作的话，异步方法里就直接报错了，所以必须在调用异步方法之前就commit当前的事务。针对性的代码示例如下：
```py
from django.db import transaction

@transaction.commit_on_success
def create_article(request):
    article = Article.objects.create(…)
    expand_abbreviations.delay(article.pk)
```

理论上来说，调用异步方法的时候这个article就是还没有创建的，因为是分离在两个进程的代码，所以异步方法中查不到这个article，就出现了race condition问题。

还有一点需要注意的是，在django中调用异步方法的时候，也不要以models里面的对象作为参数。如果需要这些对象，应该每次都去db里去查，否则也有可能面临race condition问题。


**但是事务性怎么办！！！**

参考：http://docs.celeryproject.org/en/latest/userguide/tasks.html#database-transactions
