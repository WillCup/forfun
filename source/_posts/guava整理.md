title: guava整理 - Basic utilities
date: 2015-11-19 15:04:51
tags: guava
---

总是看到guava的身影，今天来大概浏览一下这个Google项目。

### 1. 对于null的思考
null值的特点：
 
 - 不能清晰表达任何业务模型
    就是说在很多情况下，我们看到null的时候通常要分析一下context才能知道这里是什么情况
 - 容易被人忽视
    很多时候coder会专注于自己正在思考的核心业务场景，编写代码逻辑。容易忘记处理null对象的情况，所以一些nullPointerException通常是由于不可避免的粗心遗漏造成的。

Optional<T>
    当获取某个值的时候，尽量使用Optional<T>, T能够帮助我们理解当前的业务模型，而Optional.get()这一个不是很方便的调用则会时刻提醒我们充分考虑当前场景下，如果主对象为null应当采取的措施。
    有异曲同工之妙的还有scala中的Option<T>

API概览：
    http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/base/Optional.html

### 2. 前置条件

就是一些常用的、通用的、简单的条件判断，
API:http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/base/Preconditions.html

java7其实也添加了一些方法：
http://docs.oracle.com/javase/7/docs/api/java/util/Objects.html#requireNonNull(java.lang.Object,java.lang.String)

如果不满足这些条件，就会抛出Exception

### 3. 通用对象方法

对于Object默认的toString hashCode等方法提供简单实现
```
Objects.equal("a", "a"); // returns true
Objects.equal(null, "a"); // returns false
Objects.equal("a", null); // returns false
Objects.equal(null, null); // returns true
```
equals方法给处理了null值情况。

java7也是用了类似guava的http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/base/Objects.html#hashCode(java.lang.Object...)的
http://docs.oracle.com/javase/7/docs/api/java/util/Objects.html#hash(java.lang.Object...)

可读的toString
```
// Returns "ClassName{x=1}"
   MoreObjects.toStringHelper(this)
       .add("x", 1)
       .toString();

   // Returns "MyObject{x=1}"
   MoreObjects.toStringHelper("MyObject")
       .add("x", 1)
       .toString();
```

http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/base/Objects.html#equal(java.lang.Object, java.lang.Object)

### 4. ComparisonChain

任何一个条件返回非0值，就比较结束。

```
public int compareTo(Foo that) {
     return ComparisonChain.start()
         .compare(this.aString, that.aString)
         .compare(this.anInt, that.anInt)
         .compare(this.anEnum, that.anEnum, Ordering.natural().nullsLast())
         .result();
   }
```

### 5. 异常传播
简化Throwable的处理
有时候我们catch住一个Exception，但是并不想在这个catch块里处理，就会throw出去给下个块。最常见的例子是RuntimeException和Error，本来我们并不想catch它们，但是通常它们又会被catch住。

具体使用没看太明白



集合类使用

### 1. 不可变集合

- 安全
- 不存在并发问题
- 空间可预测

注：目前guava的不可变集合都不许有null元素，如果要使用带有null的不可变集合，那么选用JDK自带的Collections.unmodifiableXXX

使用不可变集合

- copyOf：如ImmutableSet.copyOf(set);
- of：如ImmutableSet.of(“a”, “b”, “c”)或 ImmutableMap.of(“a”, 1, “b”, 2);
- builder：public static final ImmutableSet<Color\> GOOGLE_COLORS =
        ImmutableSet.<Color\>builder()
            .addAll(WEBSAFE_COLORS)
            .add(new Color(0, 191, 255))
            .build();
asList视图

所有不可变集合都有一个asList()方法提供ImmutableList视图，来帮助你用列表形式方便地读取集合元素。例如，你可以使用sortedSet.asList().get(k)从ImmutableSortedSet中读取第k个最小元素。

asList()返回的ImmutableList通常是——并不总是——开销稳定的视图实现，而不是简单地把元素拷贝进List。也就是说，asList返回的列表视图通常比一般的列表平均性能更好，比如，在底层集合支持的情况下，它总是使用高效的contains方法。

### 2. 新的集合类型
JDK没有，但是又被非常广泛使用的集合类型。

- MultiSet
```
 private void outMuiltSet() {
    System.out.println("-------------------");
    System.out.println(strs.size());
    System.out.println(strs.count(new Object())); //count(Object element): 返回给定参数元素的个数
    System.out.println(strs.count("sdfsd")); //count(Object element): 返回给定参数元素的个数
    System.out.println(strs);
  }

  public void test3() throws Throwable {
    strs = TreeMultiset.create();
    strs.add("sdfsd", 2); //add(E element,int occurrences): 向其中添加指定个数的元素
    strs.add("sdfsd");  // add(E element): 向其中添加单个元素
    strs.add("dlkjlkj");
    System.out.println(strs.elementSet());  // elementSet(): 将不同的元素放入一个Set中
    Set<Multiset.Entry<Comparable>> entries = strs.entrySet();  //类似与Map.entrySet 返回Set。包含的Entry支持使用getElement()和getCount()
    for (Multiset.Entry entry: entries) {
      System.out.println(entry.getElement());
      System.out.println(entry.getCount());
    }
    outMuiltSet();
    strs.remove("sdfsd"); // remove(E element): 移除一个元素，其count值 会响应减少
    outMuiltSet();
    strs.setCount("sdfsd", 10); // 设定某一个元素的重复次数
    outMuiltSet();
    strs.remove("sdfsd", 2); //remove(E element,int occurrences): 移除相应个数的元素
    outMuiltSet();
    strs.setCount("sdfsd", 8, 20); //setCount(E element,int oldCount,int newCount): 将符合原有重复个数的元素修改为新的重复次数
    strs.setCount("sdfsd", 8, 20);
    strs.setCount("dlkjlkj", 8, 20);
    outMuiltSet();
  }
```
- SortedMultiset

SortedMultiset是Multiset 接口的变种，它支持高效地获取指定范围的子集。比方说，你可以用 latencies.subMultiset(0,BoundType.CLOSED, 100, BoundType.OPEN).size()来统计你的站点中延迟在100毫秒以内的访问，然后把这个值和latencies.size()相比，以获取这个延迟水平在总体访问中的比例。

TreeMultiset实现SortedMultiset接口。

- Multimap
可以用两种方式思考Multimap的概念:

"键-单个值映射"的集合: a->1, a->2, a->4, b->3, c->5
"键-值集合映射"的映射: a->[1,2,4], b->3, c->5
一般情况下都会使用ListMultimap或SetMultimap接口，它们分别把键映射到List或Set。
Multimap.get(key)以集合形式返回键所对应的值视图, 即使没有任何对应的值，也会返回空集合。
对值视图集合进行的修改最终都会反映到底层的Multimap。

```
public void testMap() {
    ImmutableListMultimap<String, Integer> of = ImmutableListMultimap.of("a", 1, "a", 2, "b", 1);
    System.out.println(of.asMap());
    ListMultimap map = ArrayListMultimap.create(of);
    map.put("a", 234);
    map.putAll("c", ImmutableSet.of(123,3243));
    System.out.println(map);
  }
```

- BiMap
实现键值对的双向映射,并保持它们间的同步。

```
 BiMap<String, Integer> map = HashBiMap.create();
    map.put("foo", 1);
    map.put("bar", 2);
    map.put("quux", 3);

    map.inverse().forcePut(1, "quux");  // 会导致foo被替换成quux，而quux的value也成了1，就是{bar=2, quux=1}
    System.out.println(map);
```

- Table

```
String v1 = "a";
    String v2 = "b";
    String v3 = "c";
    Table<String, String, Integer> weightedGraph = HashBasedTable.create();
    weightedGraph.put(v1, v2, 4);
    weightedGraph.put(v1, v3, 20);
    weightedGraph.put(v2, v3, 5);
    System.out.println(weightedGraph);

    System.out.println(weightedGraph.row(v1) + "........sdfsd"); // 按照rowKey查询
    System.out.println(weightedGraph.column(v3)); // 按照Columnkey查询
```

- ClassToInstanceMap

```
ClassToInstanceMap<Number> numberDefaults=MutableClassToInstanceMap.create();
    numberDefaults.putInstance(Integer.class, Integer.valueOf(0));
    numberDefaults.putInstance(Integer.class, Float.valueOf(0)); // 编译报错
```

- RangeSet
RangeSet描述了一组不相连的、非空的区间。当把一个区间添加到可变的RangeSet时，所有相连的区间会被合并，空区间会被忽略。
```
RangeSet<Integer> rangeSet = TreeRangeSet.create();
    System.out.println(rangeSet);
    rangeSet.add(Range.closed(1, 10)); // {[1,10]}
    System.out.println(rangeSet);
    rangeSet.add(Range.closedOpen(11, 15));//不相连区间:{[1,10], [11,15)}
    System.out.println(rangeSet);
    rangeSet.add(Range.closedOpen(15, 20)); //相连区间; {[1,10], [11,20)}
    System.out.println(rangeSet);
    rangeSet.add(Range.openClosed(0, 0)); //空区间; {[1,10], [11,20)}
    System.out.println(rangeSet);
    rangeSet.remove(Range.open(5, 10)); //分割[1, 10]; {[1,5], [10,10], [11,20)}
    System.out.println(rangeSet);
```

- RangeMap
RangeMap描述了”不相交的、非空的区间”到特定值的映射。和RangeSet不同，RangeMap不会合并相邻的映射，即便相邻的区间映射到相同的值。
```
RangeMap<Integer, String> rangeMap = TreeRangeMap.create();
    rangeMap.put(Range.closed(1, 10), "foo"); //{[1,10] => "foo"}
    System.out.println(rangeMap);
    rangeMap.put(Range.open(3, 6), "bar"); //{[1,3] => "foo", (3,6) => "bar", [6,10] => "foo"}
    System.out.println(rangeMap);
    rangeMap.put(Range.open(10, 20), "foo"); //{[1,3] => "foo", (3,6) => "bar", [6,10] => "foo", (10,20) => "foo"}
    System.out.println(rangeMap);
    rangeMap.remove(Range.closed(5, 11)); //{[1,3] => "foo", (3,5) => "bar", (11,20) => "foo"}
    System.out.println(rangeMap);
```

