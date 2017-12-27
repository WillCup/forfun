
title: flask笔记1
date: 2016-12-05 17:05:28
tags: [youdaonote]
---

instance文件夹
---
有时你需要定义一些不能为人所知的配置变量。为此，你会想要把它们从config.py中的其他变量分离出来，并保持在版本控制之外。 你可能要隐藏类似数据库密码和API密钥的秘密，或定义特定于当前机器的参数。 为了让这更加轻松，Flask提供了一个叫instance文件夹的特性。

instance文件夹是根目录的一个子文件夹，包括了一个特定于当前应用实例的配置文件。我们不要把它提交到版本控制中。

以上是copy的别人的代码，引入instance中特定配置文件的方式是
```python
app = Flask(__name__)
app.config.from_object('config')
app.config.from_pyfile('instance/config.py')
```
这样弄的话，就没有instance的概念了
但是至少能找到这个配置文件
可是flask本身是内置了又instance的概念的............

我参考的网站都让我配置一个instance的相对路径`instance_relative_config`或者绝对路径`instance_path`.它们配置后的期望代码是
```python
app = Flask(__name__, instance_relative_config=True)
app.config.from_object('config')
app.config.from_pyfile('config.py')
```
就是会自己把自己置于instance目录下，取相对的config.py文件。

其实不然，我使用flask版本是0.11.1，看的文档是0.10.1的，不知道是不是版本问题。

借助instance中的配置，我们可以在根路径下创建给config包，然后下面分别放置global.py、prod.py、dev.py、test.py，看名字就知道了，一个是全局的，flask加载的时候一定要弄进配置里。但是对于后面的prod.py、dev.py、test.py要加载哪一个？


参考的资料是自己设置一个环境变量APP_CONFIG_FILE，就是在shell环境里去export.
```python
# yourapp/__init__.py

app = Flask(__name__, instance_relative_config=True)
app.config.from_object('config.default')
app.config.from_pyfile('instance/config.py') # 从instance文件夹中加载配置
app.config.from_envvar('APP_CONFIG_FILE')
```
最后这个`app.config.from_envvar`是从环境变量所代表的文件里加载配置。
然后运行的时候
```bash
APP_CONFIG_FILE=/var/www/app/config/prod.py
python run.py
```

我认为也直接在instance里进行配置了。
```python
app = Flask(__name__)
app.config.from_object('config')
app.config.from_pyfile('instance/config.py')
app.config.from_pyfile(app.config['APP_CONFIG_FILE'])
```



不错的学习网站：
- https://spacewander.github.io/explore-flask-zh/5-configuration.html
- http://docs.jinkan.org/docs/flask

蓝图
--- 

参考：
- http://docs.jinkan.org/docs/flask/blueprints.html#url
