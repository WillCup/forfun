
title: mondrian-定义schema
date: 2016-12-05 17:11:20
tags: [youdaonote]
---

一个schema定义了一个多维数据库。它包含一个逻辑模型，这个逻辑模型中包含了多个立方体、层级、成员、以及将这个模型映射到物理模型的映射。

逻辑模型包含了MDX语言中的构造组件：: cubes, dimensions, hierarchies, levels, and members.
物理模型是物理数据源。一般是星型模型，就是RDBMS中的一堆表。

Scheme 文件
---
mondrian的schema是定义在xml文件中的。有一个比较全面的例子demo/FoodMart.xml。目前，创建schema只能通过编辑xml文件来实现。下面是xml结构：
```xml
<Schema>
<Cube>
<Table>
<AggName>
aggElements
<AggPattern>
aggElements
<Dimension>
<Hierarchy>
relation
<Closure/>
<Level>
<KeyExpression>
<SQL/>
<NameExpression>
<SQL/>
<CaptionExpression>
<SQL/>
<OrdinalExpression>
<SQL/>
<ParentExpression>
<SQL/>
<Property>
<PropertyExpression>
<SQL/>
<DimensionUsage>
<Measure>
<MeasureExpression>
<SQL/>
<CalculatedMemberProperty/>
<CalculatedMember>
<Formula/>
<CalculatedMemberProperty/>
<NamedSet>
<Formula/>
<VirtualCube>
<CubeUsages>
<CubeUsage>
<VirtualCubeDimension>
<VirtualCubeMeasure>
<Role>
<SchemaGrant>
<CubeGrant>
<DimensionGrant>
<HierarchyGrant>
<MemberGrant/>
<Union>
<RoleUsage/>
<UserDefinedFunction/>
<Parameter/>

relation ::=
<Table>
<SQL/>
<View>
<SQL/>
<InlineTable>
<ColumnDefs>
<ColumnDef>
<Rows>
<Row>
<Value>
<Join>
relation

aggElement ::=
<AggExclude>
<AggFactCount>
<AggIgnoreColumn>
<AggForeignKey>
<AggMeasure>
<AggLevel>
```

注意：xml元素的顺序。<UserDefinedFunction> 必须在 <Cube>, <VirtualCube>, <NamedSet> and <Role> 元素后面。

#### 2.1 注解
主要的元素(schema, cube, virtual cube, shared dimension, dimension, hierarchy, level, measure, calculated member)都支持注解。注解能够把用户定义的属性和元数据元素联系起来，允许工具在不继承mondrian schema的前提下添加元数据。

下面例子，为Schema添加了 Author和Date两个注解。
```xml
<Schema name="Rock Sales">
<Annotations>
<Annotation name="Author">Fred Flintstone</Annotation>
<Annotation name="Date">10,000 BC</Annotation>
</Annotations>
<Cube name="Sales">
...
```

3 逻辑模型
---
最重要的三个组件： cubes, measures, dimensions。
- cube。 维度和指标在指定子区域的集合。
- measure。量化的指标。
- dimension。可切分指标的属性。

下面是一个简单的schema:
```xml
<Schema>
    <Cube name="Sales">
        <Table name="sales_fact_1997"/>
        <Dimension name="Gender" foreignKey="customer_id">
            <Hierarchy hasAll="true" allMemberName="All Genders" primaryKey="customer_id">
                <Table name="customer"/>
                <Level name="Gender" column="gender" uniqueMembers="true"/>
            </Hierarchy>
        </Dimension>
        <Dimension name="Time" foreignKey="time_id">
            <Hierarchy hasAll="false" primaryKey="time_id">
                <Table name="time_by_day"/>
                <Level name="Year" column="the_year" type="Numeric" uniqueMembers="true"/>
                <Level name="Quarter" column="quarter" uniqueMembers="false"/>
                <Level name="Month" column="month_of_year" type="Numeric" uniqueMembers="false"/>
            </Hierarchy>
        </Dimension>
        <Measure name="Unit Sales" column="unit_sales" aggregator="sum" formatString="#,###"/>
        <Measure name="Store Sales" column="store_sales" aggregator="sum" formatString="#,###.##"/>
        <Measure name="Store Cost" column="store_cost" aggregator="sum" formatString="#,###.00"/>
        <CalculatedMember name="Profit" dimension="Measures" formula="[Measures].[Store Sales] - [Measures].[Store Cost]">
            <CalculatedMemberProperty name="FORMAT_STRING" value="$#,##0.00"/>
        </CalculatedMember>
    </Cube>
</Schema>
```
这个schema只有一个立方体，叫做'sales'。两个维度，time和gender。四个指标。

我们可以基于这个schema写出如下MDX查询：
```sql
SELECT {[Measures].[Unit Sales], [Measures].[Store Sales]} ON COLUMNS,
  {descendants([Time].[1997].[Q1])} ON ROWS
FROM [Sales]
WHERE [Gender].[F]
```

#### 3.1 cube
cube是指标和维度的集合。指标和维度的共同处之一就是都在事实表上，这里就是 "sales_fact_1997表。指标基于事实表里的字段计算，事实表里也包含所有的维度。

事实表使用<Table>元素定义。如果事实表不在默认的schema中，你可以使用schema属性明确指定：
```xml
<Table schema=" dmart" name="sales_fact_1997"/>
```

还可以使用<View>构造更复杂的SQL语句。事实表不支持<Join>。

#### 3.2 measure
Sales cube定义了 "Unit Sales" 和 "Store Sales"几个指标。
```xml
<Measure name="Unit Sales" column="unit_sales" aggregator="sum" datatype="Integer" formatString="#,###"/>
<Measure name="Store Sales" column="store_sales" aggregator="sum" datatype="Numeric" formatString="#,###.00"/>
```
datatype是可选的属性，它指示着这个指标在mondrian缓存中存在的形式，也决定这些指标怎样通过XML返回。("String", "Integer", "Numeric", "Boolean", "Date", "Time", "Timestamp") 。 除了"count"或者 "distinct-count" 是"Integer"，其他的aggregator的默认值都是"Numeric"。

formatString也是可选的，它代表指标被打印的形式。[更多参考](http://mondrian.pentaho.com/documentation/mdx.php#Format_strings)

获取某个指标的名称可以调用 Member.getCaption() 。

指标使用cell reader，而不是读取column，或者也能使用sql表达式来计算。 "Promotion Sales"就是通过sql表达式计算的：
```xml
<Measure name="Promotion Sales" aggregator="sum" formatString="#,###.00">
	<MeasureExpression>
		<SQL dialect="generic">
			(case when sales_fact_1997.promotion_id = 0 then 0 else sales_fact_1997.store_sales end)
		</SQL>
	</MeasureExpression>
</Measure>
```

要格式化cell 值，[请参考](http://mondrian.pentaho.com/documentation/schema.php#Cell_formatter)

#### 3.3 dimension, hierarchy, level
先看一些定义：
- member。 指定维度中的一个点。性别hierarchy包含M和F两个member。
- hierarchy。一组形成某个结构的member。store hierarchy包含name, city, state, nation。这个hierarchy允许出现中间层次，一个state的子聚合（sub-total）是这个state的所有city的所有子聚合(sub-total)之和。
- level。同一个level的member距离hierarchy最上层root的距离相等，也就是说在同一层上
- dimemsion。一组hierarchy的集合，用来区分同一个事实表中的属性。

例子：
```xml
<Dimension name="Gender" foreignKey="customer_id">
	<Hierarchy hasAll="true" primaryKey="customer_id">
		<Table name="customer"/>
		<Level name="Gender" column="gender" uniqueMembers="true"/>
	</Hierarchy>
</Dimension>
```
这个维度包含了一个hierarchy，这个hierarchy包含了一个level。这个维度的值来源于customer表的gender字段。

对于任意一个订单，gender维度是这个顾客的性别，应该是事实表 "sales_fact_1997.customer_id" 和维度表"customer.customer_id" join的结果。

##### 3.3.1 将维度和hierarchy映射到表
一个维度通过一对儿字段join进入一个cube，一个字段来自事实表，另一个来自维度表。 <Dimension>元素有一个foreignKey属性，它是事实表里字段的名字。<Hierarchy>元素有一个primaryKey属性。

如果hierarchy涉及到多个表，我们可以使用primaryKey来区分。

column属性定义了level的key。必须是在level表中的名字。如果key是一个表达式，可以在level中使用<KeyExpression>元素。下面是一个例子
```xml
<Dimension name="Gender" foreignKey="customer_id">
	<Hierarchy hasAll="true" primaryKey="customer_id">
	<Table name="customer"/>
		<Level name="Gender" column="gender" uniqueMembers="true">
			<KeyExpression>
				<SQL dialect="generic">customer.gender</SQL>
			</KeyExpression>
		</Level>
	</Hierarchy>
</Dimension>
```
<Level>, <Measure>, <Property>还有一些嵌套属性：
Parent element	| 属性 |	Equivalent nested element |	描述
---|---|---|---
<Level>	| 	column		| <KeyExpression>		| Key of level.
<Level	| nameColumn	| 	<NameExpression>	| 	Expression which defines the name of members of this level. If not specified, the level key is used.
<Level>		| ordinalColumn	| 	<OrdinalExpression>	| 	Expression which defines the order of members. If not specified, the level key is used.
<Level>		| captionColumn		| <CaptionExpression>	| 	Expression which forms the caption of members. If not specified, the level name is used.
<Level>		| 	| 	<ParentExpression>	| 	Expression by which child members reference their parent member in a parent-child hierarchy. Not specified in a regular hierarchy.
<Measure>	| 	column	| 	<MeasureExpression>		| SQL expression to calculate the value of the measure (the argument to the SQL aggregate function).
<Property>		| column	| 	<PropertyExpression>	| 	SQL expression to calculate the value of the property.

uniqueMembers 属性用来优化SQL生成过程。如果知道某个level字段的值是唯一的，就设置uniqueMembers为true。 例如[Year].[Month] 在Month level就不是唯一的，因为在不同年里会出现相同的月份。而[Product Class].[Product Name] 就是唯一的，因为产品名字不会有相同的。顶层的都是唯一的。

highCardinality 属性通知mondrian当前维度会有很大的不可预估的值。

##### 3.3.2 'all' member
默认每个hierarchy都有一个顶层节点level，'all'。它是所有其他member的父节点，也是这个hierarchy的默认值，当hierarchy没有被包含在某个坐标系或者切片的时候，就使用这个值来计算cell值。allMemberName 和allLevelName 属性重写所有level和所有member的默认名称。

如果 <Hierarchy> 元素有属性 hasAll="false"，那么'all'这个level就失效了。维度的默认值就成了第一个level里的第一个元素，如果是time hierarchy的话，就是hierarchy的第一年。修改默认值的话，容易产生混淆，所以含是使用 hasAll="true"吧。

下面是覆盖默认值的例子
```xml
<Dimension name="Time" type="TimeDimension" foreignKey="time_id">
    <Hierarchy hasAll="false" primaryKey="time_id" defaultMember="[Time].[1997].[Q1].[1]"/>
    ....
```


##### 3.3.3 Time dimemsion
时间维度是基于年月周日的。与之相关的MDX的内置函数如下：
- ParallelPeriod([level[, index[, member]]])
- PeriodsToDate([level[, member]])
- WTD([member])
- MTD([member])
- QTD([member])
- YTD([member])
- LastPeriod(index[, member])


例子
```xml
<Dimension name="Time" type="TimeDimension">
	<Hierarchy hasAll="true" allMemberName="All Periods" primaryKey="dateid">
		<Table name="datehierarchy"/>
		<Level name="Year" column="year" uniqueMembers="true" levelType="TimeYears" type="Numeric"/>
		<Level name="Quarter" column="quarter" uniqueMembers="false" levelType="TimeQuarters"/>
		<Level name="Month" column="month" uniqueMembers="false" ordinalColumn="month" nameColumn="month_name" levelType="TimeMonths" type="Numeric"/>
		<Level name="Week" column="week_in_month" uniqueMembers="false" levelType="TimeWeeks"/>
		<Level name="Day" column="day_in_month" uniqueMembers="false" ordinalColumn="day_in_month" nameColumn="day_name" levelType="TimeDays" type="Numeric"/>
	</Hierarchy>
</Dimension>
```


##### 3.3.4 level的顺序与展示
注意时间hierarchy的例子中的ordinalColumn 和nameColumn 属性。这两个属性关系到level怎样在结果中展现。ordinalColumn 属性指定了hierarchy中的一个字段，根据这个字段排序，nameColumn 则指明了要展示的字段。

例如，在月份level中，datehierarchy有month (1 .. 12) ，和month_name (January, February, ...)两个字段。MDX内部使用的字段是month，此种形式：[Time].[2005].[Q1].[1]。Month level的元素会按照January, February...的顺序显示。

在父子hierarchy中，元素都是按照层级顺序排序的。ordinalColumn控制着同一个分支的兄弟节点之间的顺序。

level包含一个type属性，它的枚举有 "String", "Integer", "Numeric", "Boolean", "Date", "Time", "Timestamp".

##### 3.5 多个hierarchy
有的维度包含多个hierarchy：
```xml
<Dimension name="Time" foreignKey="time_id">
	<Hierarchy hasAll="false" primaryKey="time_id">
		<Table name="time_by_day"/>
		<Level name="Year" column="the_year" type="Numeric" uniqueMembers="true"/>
		<Level name="Quarter" column="quarter" uniqueMembers="false"/>
		<Level name="Month" column="month_of_year" type="Numeric" uniqueMembers="false"/>
	</Hierarchy>
	<Hierarchy name="Time Weekly" hasAll="false" primaryKey="time_id">
		<Table name="time_by_week"/>
		<Level name="Year" column="the_year" type="Numeric" uniqueMembers="true"/>
		<Level name="Week" column="week" uniqueMembers="false"/>
		<Level name="Day" column="day_of_week" type="String" uniqueMembers="false"/>
	</Hierarchy>
</Dimension>
```

注意，第一个hierarchy并没有name属性，默认是继承它的dimemsion。

以上两个hierarchy之间除了都是使用time_id作为join的key以外没有任何其他地方有相同点。把两个hierarchy放在同一个dimemsion中完全是为了迎合终端用户，因为终端用户认为为'Time' 和'Time Weekly'两个hierarchy弄两个坐标系完全是没有什么道理的。如果两个hierarchy在同一个dimemsion中，MDX语言就会强制你只能使用两者之中的一个。

##### 3.3.6 Degenerate dimensions 
退化维度，是指特别简单都不值当给他创建一个表的问题。例如下面的事实表:
product_id|	time_id|	payment_method|	customer_id	store_id|	item_count	dollars
---|---|---|---|---
55	20040106|	Credit	|123|	22|	3|	$3.54
78	20040106|	Cash|	89|	22|	1|	$20.00
199	20040107|	ATM|	3|	22	|2	|$2.99
55	20040106|	Cash|	122|	22	|1|	$1.18

假设我们为payment_method 字段创建了一个表。：
```
payment_method
Credit
Cash
ATM
```

它只有3个可能的值，建表没啥意义。我们应该创建一个退化维度。为此，我们要创建一个没有table的dimemsion，mondrian会认为这些字段来源于事实表：
```xml
<Cube name="Checkout">
	<!-- The fact table is always necessary. -->
	<Table name="checkout">
	<Dimension name="Payment method">
		<Hierarchy hasAll="true">
			<!-- No table element here. Fact table is assumed. -->
			<Level name="Payment method" column="payment_method" uniqueMembers="true"/>
		</Hierarchy>
	</Dimension>
	<!-- other dimensions and measures -->
</Cube>
```

由于没有join操作，所以foreignKey也米有必要了。hierarchy也没有table元素以及primaryKey属性。


##### 3.3.7 inline table
<InlineTable>用来在schema中定义一个简单的数据集。

```xml
<Dimension name="Severity">
	<Hierarchy hasAll="true" primaryKey="severity_id">
		<InlineTable alias="severity">
			<ColumnDefs>
				<ColumnDef name="id" type="Numeric"/>
				<ColumnDef name="desc" type="String"/>
			</ColumnDefs>
			<Rows>
				<Row>
					<Value column="id">1</Value>
					<Value column="desc">High</Value>
				</Row>
				<Row>
					<Value column="id">2</Value>
					<Value column="desc">Medium</Value>
				</Row>
				<Row>
					<Value column="id">3</Value>
					<Value column="desc">Low</Value>
				</Row>
			</Rows>
		</InlineTable>
		<Level name="Severity" column="id" nameColumn="desc" uniqueMembers="true"/>
	</Hierarchy>
</Dimension>
```

以上面数据集声明dimemsion：
```xml
<Dimension name="Severity">
	<Hierarchy hasAll="true" primaryKey="severity_id">
		<Table name="severity"/>
		<Level name="Severity" column="id" nameColumn="desc" uniqueMembers="true"/>
	</Hierarchy>
</Dimension>
```


##### 3.3.8 Member properties and formatters 
As we shall see later, a level definition can also define member properties and a member formatter.

##### 3.3.9 Approximate level cardinality 
The <Level> element allows specifying the optional attribute "approxRowCount". Specifying approxRowCount can improve performance by reducing the need to determine level, hierarchy, and dimension cardinality. This can have a significant impact when connecting to Mondrian via XMLA.

##### 3.3.10 Default Measure Attribute 

