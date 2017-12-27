
title: celery笔记
date: 2016-12-05 17:07:37
tags: [youdaonote]
---

需求需要异步调用后台执行时间长的任务，而且任务结束后回写一些status到数据库。

开始的时候，在django中使用自己的threadpool：
```py
# !/usr/bin/env python
# -*- coding:utf-8 -*-
'''
created by will 
'''
import Queue
import logging
import subprocess
import threading
import time, os
from django.utils import timezone
from subprocess import Popen

from metamap.models import Executions

from metamap.utils import enums

logger = logging.getLogger('django')
class WorkManager(object):
    def __init__(self, work_num=1000, thread_num=2):
        self.work_queue = Queue.Queue()
        self.threads = []
        # self.__init_work_queue(work_num)
        self.__init_thread_pool(thread_num)

    """
      初始化线程
    """

    def __init_thread_pool(self, thread_num):
        for i in range(thread_num):
            self.threads.append(Work(self.work_queue))

    """
      初始化工作队列
    """

    def __init_work_queue(self, jobs_num):
        for i in range(jobs_num):
            self.add_job(do_job, i)

    """
      添加一项工作入队
    """

    def add_job(self, func, *args):
        self.work_queue.put((func, list(args)))  # 任务入队，Queue内部实现了同步机制

    """
      检查剩余队列任务
    """

    def check_queue(self):
        return self.work_queue.qsize()

    """
      等待所有线程运行完毕
    """

    def wait_allcomplete(self):
        for item in self.threads:
            if item.isAlive(): item.join()


class Work(threading.Thread):

    def __init__(self, work_queue):
        threading.Thread.__init__(self)
        self.work_queue = work_queue
        self.start()

    def run(self):
        # 死循环，从而让创建的线程在一定条件下关闭退出
        while True:
            location = ''
            try:
                do, args = self.work_queue.get_nowait()  # 任务异步出队，Queue内部实现了同步机制
                logger.info('%s method ' % do)
                logger.info('%s find job ....... %s ' % (self.getName(), ''.join(args[0])))
                returncode = do(args)
                self.work_queue.task_done()  # 通知系统任务完成
                location = ''.join(args[1])
                execution = Executions.objects.get(logLocation=location)
                execution.end_time = timezone.now()
                logger.info('%s return code is %d' % (self.getName(), returncode))
                if returncode == 0:
                    execution.status = enums.EXECUTION_STATUS.DONE
                else:
                    execution.status = enums.EXECUTION_STATUS.FAILED
                execution.save()
            except Queue.Empty:
                time.sleep(3)
            except Executions.DoesNotExist:
                logger.error('cannot find execution from db for location %s' % location)
            except Exception, e:
                logger.error('got error :%s' % str(e))
                break


# 具体要做的任务
def do_job(*args):
    logger.debug(args)
    log = args[0][1]
    command = args[0][0]
    p = Popen([''.join(command)], stdout=open(log, 'a'), stderr=subprocess.STDOUT, shell=True, universal_newlines=True)
    p.wait()
    return p.returncode


if __name__ == '__main__':
    start = time.time()
    # work_manager = WorkManager(10, 3)
    # work_manager.add_job(do_job, "sh /usr/local/metamap/test.sh")
    # work_manager.add_job(do_job, "sh /usr/local/metamap/test.sh")
    print "doneeeeeeeeeeeeee"
    end = time.time()
    print "cost all time: %s" % (end - start)

```

遇到两个问题：
 - django项目的manage.py运行的时候，连接池会随着view代码的加载而初始化运行，导致命令不退出，一直hang着
 - 结合gunicorn使用的时候，即便已经完成了django的一次request和response过程，重定向到了任务的状态页面。但是gunicorn任务此次request并未结束，导致worker自认为请求超时自杀，另起worker。比如后台异步任务在连接池中需要执行400s，而gunicorn的请求超时时间是300s，那么到了300s的时候当前worker的进程就被kill，然后另起一个新的worker。那么后台的线程池里执行的这个任务就也随着worker的退出被中止了。
 

问了一些大神的意见， 推荐给我celery。尝试django-celery未成功，所以就使用了原生的celery，解决了以上问题。
