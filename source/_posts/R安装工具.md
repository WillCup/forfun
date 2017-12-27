
title: R安装工具
date: 2017-02-08 14:33:05
tags: [youdaonote]
---

安装R的packages的时候官方提供了两种方式：
 - 交互式REPL里执行install.packages()
 - 命令行执行R CMD INSTALL XXX.tar.gz

由于是要初始化docker的R运行环境，如果要安装多个package，这两种方法都不太方便。


找到一个不错的工具R-littler, 安装完成后，可以通过执行自己定义的脚本进行R环境甚至程序的执行。

下面是实际使用的例子：

```
#!/usr/bin/env r

repos <- "http://mirrors.tuna.tsinghua.edu.cn/CRAN/"

lib.loc <- "/usr/lib64/R/library"

install.packages("data.table", lib.loc, repos)
install.packages("rJava", lib.loc, repos)
install.packages("dplyr", lib.loc, repos)
install.packages("bit64", lib.loc, repos)
install.packages("stringr", lib.loc, repos)
install.packages("ggplot2", lib.loc, repos)
install.packages("RODBC", lib.loc, repos)
install.packages("RJDBC", lib.loc, repos)
install.packages("timeSeries", lib.loc, repos)
install.packages("fUnitRoots", lib.loc, repos)
install.packages("forecast", lib.loc, repos)
install.packages("sqldf", lib.loc, repos)
install.packages("lubridate", lib.loc, repos)
```

把以上文件命名为install.r，赋予可执行权限即可。

这只是给准备环境而已，其实也可以尝试直接写自己的业务逻辑的。

参考：
 - http://dirk.eddelbuettel.com/code/littler.examples.html
