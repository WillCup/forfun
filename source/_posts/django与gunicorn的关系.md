
title: django与gunicorn的关系
date: 2016-12-05 17:05:24
tags: [youdaonote]
---

为什么需要有gunicorn呢？

总的来说很简单：你需要一些东西来执行python，但是python并不是对于所有类型的request都很适合。


[说明一下: 我是 Gunicorn 的developer]

稍微复杂点儿解释:
不管你用什么样的app server，都会有一些upstream都是你的django app不处理的，比如一些静态资源(images/css/js)

这就出现了经典的'三层架构'，可以Google一下['three tier architecture'](https://upload.wikimedia.org/wikipedia/commons/thumb/5/51/Overview_of_a_three-tier_application_vectorVersion.svg/2000px-Overview_of_a_three-tier_application_vectorVersion.svg.png)。就是webserver(比如nginx)负责处理许多静态资源的request。需要动态处理的request会被发送给app server(比如Gunicorn)。最内层的，也就是第三层就是数据层了。

纵观历史典型应用，每一层都应该在不同的主机上。（前两层或许还会占用多个主机，负载均衡之类，比如：5个web server分发请求到两个app server上，查询一个数据库）

最近来说，各种应用形态已经各有各的特点了。小项目其实并非完全用到这三层架构，或者也不用分别运行在很多机器上。也有些方案会把app server与web server集成起来(apache httpd + mod_wsgi, nginx + mod_uwsgi等)。

对于Gunicorn来说，是从Ruby的Unicorn抽取出来，只利用nginx里的代理功能。如果Gunicorn不会直接从互联网接收request，那么我们就不用担心客户端会比较慢。这意味着Gunicorn的处理模型十分的简单。

Gunicorn与nginx的功能分离也允许Gunicorn在不影响总体性能的前提下，使用纯python重写。

具体到底谁处理了HTTP request呢，就是Gunicorn了。其实是Nginx和Gunicorn一起处理的。Nginx接收request，如果他是一个动态的request，就传递给Gunicorn来处理，然后返回一个response给Nginx，然后Nginx再传递给client。


所以，最好是使用nginx和Gunicorn一起部署django项目。如果你只是想让nginx做代理，那么也可以使用Gunicorn、mod_uwsgi、CherryPy等作为django的依赖server。


参考：
- http://serverfault.com/questions/331256/why-do-i-need-nginx-and-something-like-gunicorn
- http://www.cnblogs.com/ArtsCrafts/p/gunicorn.html
