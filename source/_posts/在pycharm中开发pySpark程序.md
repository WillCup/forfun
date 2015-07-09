title: "在pycharm中开发pySpark程序"
date: 2015-05-08 01:47:17
tags:
---
Apache spark是一个用来分析和处理大数据的有力工具。
虽然知道怎样运行pySpark shell，但是开发的时候我们总会喜欢常用的IDE。
![Figure1 - PySpark Reference](/imgs/pyspark-reference.png)
开始我尝试使用pycharm 的preference setting，添加pySpark module到external library(Figure 1).。但是编辑器里还是不能找到引用的对象(Figure 2)。最后，只能在python的运行脚本里添加相应引用了。

![Figure 2 - Reference Problem](/imgs/reference-problem.png)




通过下面的代码，添加适当的SPARK HOME目录和PYSPARK目录来引入Apache spark module
```python
import os
import sys
# Path for spark source folder
os.environ['SPARK_HOME']="your_spark_home_folder"
# Append pyspark  to Python Path
sys.path.append("your_pyspark_folder ")
try:
    from pyspark import SparkContext
    from pyspark import SparkConf
    print("Successfully imported Spark Modules")
except ImportError as e:
    print("Can not import Spark Modules",e)
    sys.exit(1)
```
成功引入后会打印：“Successfully imported Spark Modules” (Figure 3).
![Figure 3 - Successfully import pyspark](/imgs/succ-import-pyspark.png)
再来写个小程序验证PySpark是否能够正常工作. 引入pySpark module后，添加以下代码 (Figure 4).
```python
# Initialize SparkContext
sc=SparkContext('local')
words=sc.parallelize(["scala","java","hadoop","spark","akka"])
printwords.count()
```
![Figure 4 - Test pyspark](/imgs/run-pyspark.png)


转自：http://renien.github.io/blog/accessing-pyspark-pycharm
