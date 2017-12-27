
title: druid设计
date: 2017-11-15 14:28:59
tags: [youdaonote]
---

segment
---

Druid的索引是存在segment文件中的，segment文件是按照时间分区的。一个segment文件对应一个时间间隔，时间间隔根据配置项`granularitySpec`里的`segmentGranularity `进行配置。推荐segment文件大小设置在300~700M之间。如果你的segment文件太大，那么请考虑修改时间间隔，或者为数据做分区，修改`partitioningSpec`的`targetPartitionSize`参数(可以试试五百万行)。

#### 文件数据结构
segment是列式存储的：每一列都在不同的数据结构中。每个列都分开存储，druid就可以只访问查询应用到的列的数据。有三个基本的列类型：timestamp、维度、metric。

![实例](http://druid.io/docs/img/druid-column-types.png)


timestamp和metric列比较简单：都是int或者float数组，使用LZ4算法压缩。一个query只要知道他需要查询那些行，就解压相关列的这部分文件，抽取对应的行，然后执行聚合计算操作即可。

dimension列有些不同，它要支持filter和groupby操作，所以需要下面三个数据结构
- 一个字典，把值映射成int类型的id
- 使用上面字典编码的列值的list
- 列中每个不同的列指，都要弄一个bitmap对应所有的行，说明哪些行带有这个列指


把原本的值映射成int的id是为了节省空间。上面第三个里的bitmap也就是倒排索引(inverted indexes)可以提供快速的过滤工作。最后，上面第二个结构里的值，是用来处理group by和topN拆线呢的。也就是说，基于filter的单独聚合指标不要要接触到第二个数据结构中的维度值。

基于上图中的数据，看一下例子，下面是针对列`Page`的
```
1: 映射编码的字典
  {
    "Justin Bieber": 0,
    "Ke$ha":         1
  }

2: 按照1中映射后，这一列的字段值
  [0,
   0,
   1,
   1]

3: 每一个或者说每一种列值都有自己的Bitmaps
  value="Justin Bieber": [1,1,0,0]
  value="Ke$ha":         [0,0,1,1]
```

注意bitmap数据结构与前面个不同，前两个是随数据量线性增长的，而bitmap是数据量 * 每列各种值的个数。


#### 多值的列

假设`Page`的第二行数据为`Ke$ha,Justin Bieber`，那么数据结构变化为下面：
```
1: Dictionary that encodes column values
  {
    "Justin Bieber": 0,
    "Ke$ha":         1
  }

2: Column data
  [0,
   [0,1],  <--Row value of multi-value column can have array of values
   1,
   1]

3: Bitmaps - one for each unique value
  value="Justin Bieber": [1,1,0,0]
  value="Ke$ha":         [0,1,1,1]
                            ^
                            |
                            |
    Multi-value column has multiple non-zero entries
```

#### 命名约定
segment文件的标识符一般是由数据源、开始时间、结束时间、版本构成的。如果数据在某个时间范围外还sharded了，那么也会带有一个分区数字。

例子：datasource_intervalStart_intervalEnd_version_partitionNum

#### segment组件

一个segment由几个文件组成
- version.bin。  4个字节，代表segment的version。
- meta.smoosh。带有其他smoosh文件内容元数据(文件名、offset)的文件。
- xxxx.smoosh。多个这种文件，都是二进制数据。

