
title: R数据结构
date: 2017-02-08 15:23:53
tags: [youdaonote]
---

类型
=
- NULL表示没有任何值
- NA 表示缺失值


因子
=
用类别值来代表特征的属性成为**名义属性**。尽管可以用字符型向量存储名义属性，但是R提供了因子的专用数据结构。

一个是省空间、另外一些机器学习算法是用特别的程序来处理分类变量的，把分类变量编码为因子可以确保模型能够合理地处理这类数据。

```
> gender <- factor(c("MALE", "FAMALE","FAMALE"))
> gender
[1] MALE   FAMALE FAMALE
Levels: FAMALE MALE
> blood <- factor(c("A", "B"), levels = c("A", "B", "O"))
> blood
[1] A B
Levels: A B O
```
levels是说这个字段可能会有多少值

集合
= 
向量必须是同样类型的元素，而列表可以包含不同类型的元素，还可以指定元素的key。
```
> obj <- list(name="will", age=21, job="coder")
> obj
$name
[1] "will"

$age
[1] 21

$job
[1] "coder"

> obj$age
[1] 21
> obj[2]
$age
[1] 21

> obj[c("name", "age")]
$name
[1] "will"

$age
[1] 21
```


DataFrame可以看作是带有每列含义的matrix矩阵。


文件操作
=
```
> p_data <- read.csv("D:/qdsj-20170204094000")
> typeof(p_data)
[1] "list"
> p_data$用户id[1]
[1] 0f1d4b839353400998b7b289f895e369
67788 Levels: 000188cb8d044291bd9984b8e04a224a ... ffff69f594cf48eb8867e8edcdb21e87
```

可以看到有67788个level，也就是说这是已经把用户id这个字段自动因子化了。

```
> p_data <- read.csv("D:/qdsj-20170204094000", stringsAsFactors = FALSE)
> p_data$用户id[1]
[1] "0f1d4b839353400998b7b289f895e369"
> typeof(p_data)
[1] "list"
```

探索数据结构
=

进行可读化输出：
```
> str(p_data)
'data.frame':	67788 obs. of  13 variables:
 $ 用户id      : chr  "0f1d4b839353400998b7b289f895e369" "d57eb5a615004c00af1476471577bcf8" "8ffd3d4e84db4cad841d0ed8c31e0fb5" "49cdd310f2854af09ccfde82e3ea4e06" ...
 $ 用户渠道    : chr  "laiqiang_cpa_01" "laiqiang_cpa_01" "laiqiang_cpa_01" "laiqiang_cpa_01" ...
 $ 用户手机号  : chr  "186****5084" "134****3636" "188****2253" "134****3632" ...
 $ 用户姓名    : chr  "马先生" "张先生" "" "" ...
 $ 注册时间    : chr  "2016-10-19 17:46:28" "2016-10-20 16:07:05" "2016-10-20 16:08:37" "2016-10-20 19:21:22" ...
 $ 开户时间    : chr  "2016-10-19 17:51:24" "2016-10-20 16:09:13" "null" "null" ...
 $ 首投时间    : chr  "2016-10-19 17:51:54" "2016-10-20 16:29:34" "null" "null" ...
 $ 首投金额    : num  1000 1000 0 0 1000 1000 1000 0 1000 1000 ...
 $ 二投时间    : chr  "null" "null" "null" "null" ...
 $ 二投金额    : num  0 0 0 0 1208 ...
 $ 首投定期时间: chr  "2016-10-19 17:52:38" "2016-10-20 16:30:27" "null" "null" ...
 $ 首投定期金额: num  1000 1000 0 0 0 1000 1000 0 1000 1000 ...
 $ 凌晨前存量  : chr  "\\N" "\\N" "\\N" "\\N" ...
```

探索数值型变量
```
> summary(p_data$首投金额)
   Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
    0.0     0.0     0.0   128.1     0.0 50000.0 
```

可以找到DataFrame中这列的数值属性：最小值、第一四分位数、中位数、平均数、第三四分位数、最大值。

也可以同时看两列的：
```
> summary(p_data[c("首投金额", "二投金额")])
    首投金额          二投金额       
 Min.   :    0.0   Min.   :     0.0  
 1st Qu.:    0.0   1st Qu.:     0.0  
 Median :    0.0   Median :     0.0  
 Mean   :  128.1   Mean   :   228.7  
 3rd Qu.:    0.0   3rd Qu.:     0.0  
 Max.   :50000.0   Max.   :200000.0  
```

示例 | 解释
--- | ---
range(p_data$首投金额) | 返回最小值和最大值
diff(a, b) | a和b的差值
IQR(p_data$首投金额) | 四分位数中，中间的50%的差值（第三四分位数减去第一四分位数）
quantile(p_data$首投金额) | 返回四分位的五个位置上的数。 还可以通过probs指定获取百分位数中任意位置的数
boxplot(p_data$首投金额, main="sss", ylab="Money($)") | 将首投金额以箱图的形式绘图展示，名字是sss
hist(p_data$首投定期金额, main="首投定期金额", xlab= "Money($") | 直方图
var(p_data$首投金额) | 方差
sd(p_data$首投金额) | 标准差
table(p_data$首投金额) | 以表的形式输出数据分布情况，每种值的个数
prop.table(table(p_data$首投金额)) | 以表的形式输出数据分布情况，每种值占总数的比例
plot(x=p_data$首投金额, y =p_data$二投金额, main="变量关系", xlab="手头", ylab="二头") | 使用散点图展示两种特征值之间的关系， y是因变量，x是自变量
CrossTable(x=userdcars$model, y= userdcars$conservative) | 使用双向交叉表检验变量之间的关系


```
> max(p_data$首投金额)
[1] 50000
> min(p_data$首投金额)
[1] 0
> IQR(p_data$首投金额)
[1] 0
> range(p_data$首投金额)
[1]     0 50000
> diff(range(p_data$首投金额))
[1] 50000
> quantile(p_data$首投金额)
   0%   25%   50%   75%  100% 
    0     0     0     0 50000 
> quantile(p_data$首投金额, probs = c (0.25, 0.5, 0.75, 0.97, 0.99))
 25%  50%  75%  97%  99% 
   0    0    0 1000 3000
> quantile(p_data$首投金额, probs = c (0.25, 0.5, 0.75, 0.97, 0.9999))
25%     50%     75%     97%  99.99% 
0.0     0.0     0.0  1000.0 26106.5 
> quantile(p_data$首投金额, seq(from=0, to=1, by=0.1))
   0%   10%   20%   30%   40%   50%   60%   70%   80%   90%  100% 
    0     0     0     0     0     0     0     0     0   100 50000 
> table(p_data$首投金额)

      0     100     101  105.58     110     112     120     150     168     170 
  60435    1282       3       1       2       1       1       1       1       1 
> prop.table(table(p_data$首投金额))

           0          100          101       105.58          110          112 
8.915295e-01 1.891190e-02 4.425562e-05 1.475187e-05 2.950375e-05 1.475187e-05

> usedcars$conservative <-
+   usedcars$color %in% c("Black", "Gray", "Silver", "White")
> table(usedcars$conservative)

FALSE  TRUE 
   51    99 
> CrossTable(x = usedcars$model, y = usedcars$conservative)

 
   Cell Contents
|-------------------------|
|                       N |
| Chi-square contribution |
|           N / Row Total |
|           N / Col Total |
|         N / Table Total |
|-------------------------|

 
Total Observations in Table:  150 

 
               | usedcars$conservative 
usedcars$model |     FALSE |      TRUE | Row Total | 
---------------|-----------|-----------|-----------|
            SE |        27 |        51 |        78 | 
               |     0.009 |     0.004 |           | 
               |     0.346 |     0.654 |     0.520 | 
               |     0.529 |     0.515 |           | 
               |     0.180 |     0.340 |           | 
---------------|-----------|-----------|-----------|
           SEL |         7 |        16 |        23 | 
               |     0.086 |     0.044 |           | 
               |     0.304 |     0.696 |     0.153 | 
               |     0.137 |     0.162 |           | 
               |     0.047 |     0.107 |           | 
---------------|-----------|-----------|-----------|
           SES |        17 |        32 |        49 | 
               |     0.007 |     0.004 |           | 
               |     0.347 |     0.653 |     0.327 | 
               |     0.333 |     0.323 |           | 
               |     0.113 |     0.213 |           | 
---------------|-----------|-----------|-----------|
  Column Total |        51 |        99 |       150 | 
               |     0.340 |     0.660 |           | 
---------------|-----------|-----------|-----------|

 
```
根据每个人所属年级，添加对应的平均年龄

