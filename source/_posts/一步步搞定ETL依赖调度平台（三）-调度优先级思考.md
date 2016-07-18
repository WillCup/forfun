title: 一步步搞定ETL依赖调度平台（三）-调度优先级思考
date: 2016-06-17 11:21:50
tags: [ETL,调度平台]
---

从《azkaban源码追踪（三）-上传zip的坑》可以看到azkaban本身的调度基本是广度优先执行的。

### 1.猎聘

![猎聘的调度tree](/imgs/azkaban/schedule-lp-tree.png)

优先级生成策略是从叶子节点向上遍历，每个节点的优先级=所有子节点优先级+edge数(也就是子节点数)。其实思想就是谁下面的小弟多，谁的优先级就越大。

但这通常并不完全适用于对于指定的单个叶子节点最优先的需求，极端点儿想的话，可能出现老的ETL有100多个相关成一个flow，可是新增的优先级特别高的需求super_etl在另一个只有1个节点的flow，或者在同一个flow，但是它没有小弟（这种其实蛮常见的）。那么super_etl的默认priority是1，根本没有地位！！！

> LP的人说了：我们可以给它人工指定优先级啊！
> 额。。。。是的，但是其他的tree是在一直生长的，按照你们的策略来看，这个super_etl的优先级要一直改变。
> LP的人又说了：我们可以动态指定！
> 额。。。。是的，但是后面出现了super_etl01,super_etl02的时候，可能就会特别特别麻烦了。

再看一下，计划顺序与实际执行顺序的对比。本来C的优先级是比B高的，可猎聘给出的实际执行却是先执行B后执行c，音频中苏总说是并发问题，个人看来稍稍有些含糊其辞了。推测猎聘的实际调度的时候并不完全以priority为指导，而仍然是azkaban广度遍历优先，同一个level之内才是priority优先执行。

参考:http://mp.weixin.qq.com/s?src=3&timestamp=1466131251&ver=1&signature=YfS7PgVR0SjjLt4KwIQ297t4SDiLrrJu5C7uuMXpkCvuhvjXk-DU*eJGfIBSfjDm-WlVuwaIbGtbfda-k-5rRgUewAlvjSUQBBNY4jnjuvQD1puqdaVzUqIq9ds*x1r-6dUnU6OiSERzBt5UWfjdUkekpmc7hNqTm1pklClkSkE=

### 2.美团
![美团的调度tree](/imgs/azkaban/schedule-mt-tree.png)

调度策略就不该是递归调度，而应该完全按照优先级别去调度。想来想去，这才是真正的优先级调度。东家靠谱儿！


参考：http://mp.weixin.qq.com/s?src=3&timestamp=1466131231&ver=1&signature=YfS7PgVR0SjjLt4KwIQ297t4SDiLrrJu5C7uuMXpkCtKEZcZ58gspxJhkgDlgXAkqQvXpI3IbmE4HzkI33W*4PJCOUoKyzSna56Q8uSQgUNnSl7vsxqLcREVfDBU0nPguiAiwnosiXyE9wB7T-edr3QrRn8rKMhB3oxjQDrS1RQ=

综上：后面本人要开发的调度系统优先级策略会借用美团的思想。
