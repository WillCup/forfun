
title: 自制python的安装包
date: 2017-09-28 14:15:09
tags: [youdaonote]
---


起步
---
#### 名字
- 小写
- pypi中没有重复的


#### 目录结构
```
funniest/
    funniest/
        __init__.py
    setup.py
```

在__init__.py中添加
```py
def joke():
    return (u'Wenn ist das Nunst\u00fcck git und Slotermeyer? Ja! ... '
            u'Beiherhund das Oder die Flipperwaldt gersput.')
```

setup.py里应该包含一个方法setuptools.setup():
```
from setuptools import setup

setup(name='funniest',
      version='0.1',
      description='The funniest joke in the world',
      url='http://github.com/storborg/funniest',
      author='Flying Circus',
      author_email='flyingcircus@example.com',
      license='MIT',
      packages=['funniest'],
      zip_safe=False)
```

然后就可以本地安装一下了。
```
pip install .
```

用下面命令的话，可以让这个module的使用者实时更新变化
```
root@will-vm:~/gitLearning/funiest# pip install -e .
Obtaining file:///root/gitLearning/funiest
Installing collected packages: funiest
  Running setup.py develop for funiest
Successfully installed funiest
```

引用：
```
>>> import funiest
>>> print funiest.joke()
Wenn ist das Nunstück git und Slotermeyer? Ja! ... Beiherhund das Oder die Flipperwaldt gersput.
```

#### 发布到pypi
setup.py同样也是我们在PYPI上注册包名，并上传源码的入口。

##### 打包：
```
root@will-vm:~/gitLearning/funiest# python setup.py sdist
running sdist
running egg_info
writing funiest.egg-info/PKG-INFO
writing top-level names to funiest.egg-info/top_level.txt
writing dependency_links to funiest.egg-info/dependency_links.txt
reading manifest file 'funiest.egg-info/SOURCES.txt'
writing manifest file 'funiest.egg-info/SOURCES.txt'
warning: sdist: standard file not found: should have one of README, README.rst, README.txt

running check
creating funiest-0.1
creating funiest-0.1/funiest
creating funiest-0.1/funiest.egg-info
making hard links in funiest-0.1...
hard linking setup.py -> funiest-0.1
hard linking funiest/__init__.py -> funiest-0.1/funiest
hard linking funiest.egg-info/PKG-INFO -> funiest-0.1/funiest.egg-info
hard linking funiest.egg-info/SOURCES.txt -> funiest-0.1/funiest.egg-info
hard linking funiest.egg-info/dependency_links.txt -> funiest-0.1/funiest.egg-info
hard linking funiest.egg-info/not-zip-safe -> funiest-0.1/funiest.egg-info
hard linking funiest.egg-info/top_level.txt -> funiest-0.1/funiest.egg-info
Writing funiest-0.1/setup.cfg
creating dist
Creating tar archive
removing 'funiest-0.1' (and everything under it)
root@will-vm:~/gitLearning/funiest# ll dist/
总用量 12
drwxr-xr-x 2 root root 4096 9月  28 14:35 ./
drwxr-xr-x 5 root root 4096 9月  28 14:35 ../
-rw-r--r-- 1 root root  912 9月  28 14:35 funiest-0.1.tar.gz
```
这个包并没有经过build，在使用pip安装的时候需要再执行个build的步骤，尽管是纯python的也要有这个步骤。
##### wheels
wheel是一个已经build过的包，安装的时候不用走build步骤了，相对终端用户安装过程会快一些。

- 如果我们的项目是纯python(不包含编译的扩展)，本身就支持python2和python3，那么就叫做universal wheel
- 如果是纯python，但是不是原生支持python2和3，就叫做pure python wheel
- 如果包含便宜扩展，就叫 platform wheel


在把我们的project构建wheel之前，先安装wheel
```
pip install wheel
```


###### universal wheel
```
python setup.py bdist_wheel --universal
```
可以在setup.cfg中指定参数
```
[bdist_wheel]
universal=1
```
只有在你的project对于python2和python3都支持，而且不包含c扩展的时候才能用universal。

###### pure python wheel
```
python setup.py bdist_wheel
```
bdist_wheel会检测是不是纯python，会生成一个与当前python版本兼容的结果。要是想支持多个版本，就需要在2、3的版本python环境下分别构建。

###### platform wheel
用来指定运行平台：linux, windows等，通常都带着C扩展
```
python setup.py bdist_wheel
```

##### 上传到PYPI


因为我还没有过pypi的用户，所以要根据提示先注册账户。

可以把注册信息加入~/.pypirc中：
```
[pypi]
username = <username>
password = <password>
```

上传：
```
pip install twine
```
```
root@will-vm:~/gitLearning/funiest# twine upload dist/*
Uploading distributions to https://upload.pypi.org/legacy/
Enter your username: willcup
Enter your password: 
Uploading funiest-0.1-py2.py3-none-any.whl
Uploading funiest-0.1.tar.gz      
```

然后就可以看到了 https://pypi.python.org/pypi/funiest/0.1

至此，已经可以到别的机器上下载并使用了


参考：
- https://packaging.python.org/tutorials/distributing-packages/#setup-py

- https://github.com/pypa/sampleproject/blob/master/setup.py

- https://python-packaging.readthedocs.io/en/latest/
