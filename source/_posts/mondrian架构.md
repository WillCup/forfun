
title: mondrian架构
date: 2016-12-05 17:09:02
tags: [youdaonote]
---

1 mondrain系统的层次
---
- 表现层(the presentation layer)
- 维度层(the dimensional layer)
- 聚合层(the star layer)
- 存储层(the storage layer)


#### 1.1 表现层(the presentation layer)

表现层决定了最终用户将在他们的显示器上看到什么, 及他们如何同系统产生交互。
有许多方法可以用来向用户显示多维数据集, 有 pivot 表 (一种交互式的表), pie, line 和图表(bar charts)。它们可以用Swing 或 JSP来实现。
表现层以多维"文法(grammar)(维、度量、单元)”的形式发出查询，然后OLAP服务器返回结果。

#### 1.2 维度层(the dimensional layer)
维度层用来解析、验证和执行MDX查询请求。
一个MDX查询要通过几个阶段来完成：首先是计算坐标轴（axes），再者计算坐标轴axes 中cell的值。
 为了提高效率，维度层把要求查询的单元成批发送到集合层，查询转换器接受操作现有查询的请求，而不是对每个请求都建立一个MDX 声明。
 
#### 1.3 聚合层(the star layer)
集合层负责维护和创建聚合缓存，一个聚合是在内存中缓存一组单元值， 这些单元值由一组维的值来确定。
维度层对这些单元发出查询请求，如果所查询的单元值不在缓存中，则聚合管理器(aggregation manager)会向存储层发出查询请求

#### 1.4 存储层(the storage layer)
存储层是一个关系型数据库(RDBMS)。它负责创建集合的单元数据，和提供维表的成员。在后面后说一下直接使用RDBMS而不另外开发存储的原因。

这些层次组件可以都在同一个机器上，也可以分布开。第二层和第三层组成mondrian server，他俩必须在一个机器上。

2 存储与聚合策略
---
OLAP server通常都基于他们存储数据的方式被归为以下两类：
 - MOLAP（multidimensional OLAP） server。为了便于多维访问，它将所有的数据结构化存储在磁盘上。一般数据都是存储在稠密的数据中，每个cell只需要4或8个字节
 - ROLAP(relational OLAP) server。直接将数据存在RDBMS中。事实表里的每一行的字段都同时包含了维度和指标。
 
我们需要存储三种数据：事实表数据（事务记录）、聚合数据、维度。

MOLAP数据库是以多维格式存储事实数据的，但是如果维度很多的话，数据就会稀疏，这种存储格式表现就不太好了。 HOLAP (hybrid OLAP)系统的解决了这个问题，它在关系数据库中存储到最细粒度的数据，把聚合结果存在多维格式下。

对于大数据集来说，否则有些查询可能会需要遍历事实表里的所有数据。 MOLAP的聚合表通常是内存数据结构的一个快照，存储在磁盘上。ROLAP 聚合表是存在表里的。在一些 ROLAP 系统中，这些都需要OLAP server详细管理。对其他系统来说，这些表只是物化视图而已，只有在OLAP server被访问到指定查询时才会用到。

最极端的聚合策略就是缓存了，存在内存里。如果缓存里数据集在比较低的聚合level上，还可以上卷。

缓存可能是聚合策略中最重要的部分了，因为他是可以适配的。很难在不适用大量磁盘的前提下，选择聚合维度并预计算。要是维度较多，或者用户提交了某些一些不可预测的查询就更糟糕了。在数据实时变化的系统中，维护预计算聚合几乎是不可实现的。一个缓存的合理大小应该能够允许系统执行一些不可预估的查询，不适用任何聚合。

Mondrian的聚合策略如下：
 - 事实数据存在RDBMS中。我们没必要自己再开发一个存储了
 - 通过提交group by查询，将聚合数据读取进入缓存。既然RDBMS有了聚合操作，我们也直接使用
 - 如果RDBMS支持物化视图，数据库管理员为某些聚合创建了物化视图，然后mondrian就可以使用它们咯。理想情况下，mondrian的聚合管理员应该知道这些物化视图是否存在，知道怎样高效计算，甚至能够给数据库管理员一些调优建议。
 
总的原则就是能委托给数据库的就给数据库做。数据库的负载虽然增加了一些，但是数据库的客户端获益很大。多维存储应该减少IO，才能更快。

另外，因为mondrian并不需要什么存储，所以只要把jar文件加入classpath就可以运行它了。没有多余的数据管理，数据加载过程也很简单，mondrian很适合数据实时变化的OLAP。


3 API
---
mondrian为app提供了客户端api。

因为对于执行OLAP查询并没有统一规范，所以mondrian也是自己独有的API，但是熟悉jdbc的朋友们都会很容易上手。主要区别在于查询语言是MDX('Multi-Dimensional eXpressions')，而JDBC是使用标准SQL的。

下面的java代码是连接mondrian，执行查询，然后打印结果：
```java
import mondrian.olap.*;
import java.io.PrintWriter;

Connection connection = DriverManager.getConnection(
    "Provider=mondrian;" +
    "Jdbc=jdbc:odbc:MondrianFoodMart;" +
    "Catalog=/WEB-INF/FoodMart.xml;",
    null,
    false);
Query query = connection.parseQuery(
    "SELECT {[Measures].[Unit Sales], [Measures].[Store Sales]} on columns," +
    " {[Product].children} on rows " +
    "FROM [Sales] " +
    "WHERE ([Time].[1997].[Q1], [Store].[CA].[San Francisco])");
Result result = connection.execute(query);
result.print(new PrintWriter(System.out));
```

使用DriverManger创建Connection。Query对应于jdbc的Statement，通过解析MDX字符串创建。 Result对应于jdbc的ResultSet。既然我们这里处理的是多维数据，这个结果里就包含了坐标轴axes和单元格cell，而不是行row和列column了。还有，OLAP是用于数据探查的，你可以执行一些类似下钻drillDown、排序sort的操作修改parse tree，然后重新执行查询。

还有一些对象： Schema, Cube, Dimension, Hierarchy, Level, Member. 可以看一下mondrian的API了解详情。

4 MDX 
---

#### 4.1 参数
参数是在MDX查询中内嵌的一个变量。
```
Parameter(<name>, <type>, <defaultValue>[, <description>])
```
- name ,在query中唯一
- type， NUMERIC, STRING或者层次名称之一
- defaultValue。 默认值，是一个表达式，应该跟type类型一致，如果是层次，就得是层次的某个成员
- desc。 

要是想在同一个query中使用某个参数多次，就使用：
```
ParamRef(<name>)
```

下面的查询会查出California的前10名品牌，我们可以修改Count参数为前5名，或者修改Region参数为西雅图：
```
SELECT {[Measures].[Unit Sales]} on columns,
   TopCount([Product].[Brand].members,
     Parameter("Count", NUMERIC, 10, "Number of products to show"),
     (Parameter("Region", [Store], [Store].[USA].[CA]),
     [Measures].[Unit Sales])) on rows
FROM Sales
```
可以调用Query.getParameters()来获取某个查询的所有参数，然后调用Query.setParameter(String name, String value)修改参数值。


#### 4.2 内置函数
```
StrToSet(<String Expression>[, <Hierarchy>])
StrToTuple(<String Expression>[, <Hierarchy>])
```
