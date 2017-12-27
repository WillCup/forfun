
title: R语言简记
date: 2017-01-18 11:10:55
tags: [youdaonote]
---

为新生成的data.frame指定列名： 
```
colnames(df) <- c("a","b")
```

```
## 聚合one ~ one, one ~ many, many ~ one, and many ~ many:
aggregate(weight ~ feed, data = chickwts, mean)
aggregate(breaks ~ wool + tension, data = warpbreaks, mean)
aggregate(cbind(Ozone, Temp) ~ Month, data = airquality, mean)
aggregate(cbind(ncases, ncontrols) ~ alcgp + tobgp, data = esoph, sum)
```

set.seed
=
set.seed()是为了设置一个随机数的种子，当想产生同样的一组随机数字的时候，就设置一下。这在统计分析中是经常用到的。


dim
=
dim()返回一个data.table的行、列数量。
```
> x <- 1:12 ; dim(x) <- c(3,4)
> x
     [,1] [,2] [,3] [,4]
[1,]    1    4    7   10
[2,]    2    5    8   11
[3,]    3    6    9   12
> dim(x)[1]
[1] 3
> dim(x)[2]
[1] 4
```
