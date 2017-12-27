
title: 整合presto与hue
date: 2017-12-19 19:12:10
tags: [youdaonote]
---

传说是因为hue里没有直接对于presto的支持，所以大家都通过RDBMS间接的在hue上执行presto的查询。

#### 可行方案

一共找到两种方案：
- 借助postgresql创建中间的字典表，然后转化SQL进行执行。具体[参考](https://medium.com/@ilkkaturunen/integrating-presto-with-hue-61702b244839)
- presto jdbc

第一种方案部署相对复杂很多，选用了第二种方案。不过过程中发现了一些不错的项目，回头来看其实都是presto官网推荐的，可以看一下[这些resource](https://prestodb.io/resources.html)



#### 实际方案

最后确认的方案，是使用官方提供的[jdbc连接方式](https://prestodb.io/docs/0.184/installation/jdbc.html)。这种相对简单，架构也更清晰一些。

主要包含以下几个步骤
#####  1. 下载对应presto版本的jdbc包
#####  2. 修改本地java环境变量CLASSPATH

否则会报classNotFound
```
export CLASSPATH=$CLASSPATH:/server/todo/presto-jdbc-0.184.jar
```
其实个人感觉不太方便，所以在hue的启动脚本`build/env/bin/hue`里添加了一下环境变量的设置，算是高内聚一下hue相关
```py
if __name__ == '__main__':
    # 添加环境变量设置
    os.putenv('CLASSPATH', '/server/todo/presto-jdbc-0.184.jar:' + os.getenv('CLASSPATH', ''))
    print os.getenv('CLASSPATH')

    sys.argv[0] = re.sub(r'(-script\.pyw?|\.exe)?$', '', sys.argv[0])
    sys.exit(
        load_entry_point('desktop', 'console_scripts', 'hue')()
    )

```
##### 3.在hue.ini里配置interpreter
```
  [[interpreters]]
    # Define the name and how to connect and execute the language.
    [[[presto]]]
      name=Presto JDBC
      interface=jdbc
      options='{"user": "root", "password": "***empty***", "url": "jdbc:presto://datanode36.will.com:8099/hive/", "driver": "com.facebook.presto.jdbc.PrestoDriver"}'

```
这里有一个比较坑的地方，就是我们一般不会在presto server的接口做https，甚至不需要用户名密码验证。最开始的时候我配置成了
```
options='{"user": "root", "password": "", "url": "jdbc:presto://datanode36.will.com:8099/hive/", "driver": "com.facebook.presto.jdbc.PrestoDriver"}'
```

或者
```
options='{ "url": "jdbc:presto://datanode36.will.com:8099/hive/", "driver": "com.facebook.presto.jdbc.PrestoDriver"}'
```
就都会报不一样的错误, 看了一下presto jdbc的[相关源码](https://github.com/prestodb/presto/blob/master/presto-jdbc/src/main/java/com/facebook/presto/jdbc/PrestoDriverUri.java)：
```java
...
 // TODO: fix Tempto to allow empty passwords
String password = PASSWORD.getValue(properties).orElse("");
// 如果密码不为空，就走这里
if (!password.isEmpty() && !password.equals("***empty***")) {
    if (!useSecureConnection) {
        throw new SQLException("Authentication using username/password requires SSL to be enabled");
    }
    builder.addInterceptor(basicAuth(getUser(), password));
}

if (useSecureConnection) {
    setupSsl(
            builder,
            SSL_KEY_STORE_PATH.getValue(properties),
            SSL_KEY_STORE_PASSWORD.getValue(properties),
            SSL_TRUST_STORE_PATH.getValue(properties),
            SSL_TRUST_STORE_PASSWORD.getValue(properties));
}
...

```

如果密码不为空的话，java.sql.DriverManager会报错，没有提供密码...

还有就是url里一定要执行catalog，不然报错`Catalog must be specified when session catalog`


##### 4. 改jdbc.py源码

最后一个是相对最麻烦的，就是hue的通用的获取数据库元数据的方式，对于presto来说并不适用，需要自己修改hue的[python代码](https://github.com/cloudera/hue/blob/release-4.1.0/desktop/libs/notebook/src/notebook/connectors/jdbc.py)。


可以看到下面这个类里的各种获取元数据的sql，其实都是不适用于我们的presto 0.184的，所以需要针对性的修改。

```py
...
class Assist():

  def __init__(self, db):
    self.db = db

  def get_databases(self):
    #dbs, description = query_and_fetch(self.db, 'SELECT DatabaseName FROM DBC.Databases')
    dbs, description = query_and_fetch(self.db, 'show schemas')
    return [db[0] and db[0].strip() for db in dbs]

  def get_tables(self, database, table_names=[]):
    #tables, description = query_and_fetch(self.db, "SELECT * FROM dbc.tables WHERE tablekind = 'T' and databasename='%s'" % database)
    tables, description = query_and_fetch(self.db, "show tables from %s" % database)
    return [{"comment": "will ff", "type": "Table", "name": table[0] and table[0].strip()} for table in tables]

  def get_columns(self, database, table):
    #columns, description = query_and_fetch(self.db, "SELECT ColumnName, ColumnType, CommentString FROM DBC.Columns WHERE DatabaseName='%s' AND TableName='%s'" % (database, table))
    columns, description = query_and_fetch(self.db, "desc %s.%s" % (database, table))
    return [[col[0] and col[0].strip(), self._type_converter(col[1]), '', '', col[3], ''] for col in columns]

  def get_sample_data(self, database, table, column=None):
    column = column or '*'
    return query_and_fetch(self.db, 'SELECT %s FROM %s.%s' % (column, database, table))

  def _type_converter(self, name):
    return {
        "I": "INT_TYPE",
        "I2": "SMALLINT_TYPE",
        "CF": "STRING_TYPE",
        "CV": "CHAR_TYPE",
        "DA": "DATE_TYPE",
      }.get(name, 'STRING_TYPE')
```


当然直接修改这个通用接口类对于架构来说肯定是不好的，甚至是破坏性的。我测试了一下，这个类只对hue不能支持的(也就是mysql、oracle、postgresql、hive等)之外的自定义的jdbc起作用，所以并不会影响到其他数据库和引擎的使用，我们应该也没有需要其他自定义数据源了。所以，暂时这样，后面要TODO优化。

另外，我发现还有一个问题。presto的执行user是固定的，并不像hiveserver那样，执行时使用真正的用户名去hive提交。这样就不能保证ACL的很多特性了。这方面其实可以找到hue相关部分的源码（应该也在jdbc.py）里进行改进。
```py
class JdbcApi(Api):

  def __init__(self, user, interpreter=None):
    global API_CACHE
    Api.__init__(self, user, interpreter=interpreter)

    self.db = None
    self.options = interpreter['options']

    if self.cache_key in API_CACHE:
      self.db = API_CACHE[self.cache_key]
    elif 'password' in self.options:
      # 主要在于这个地方，看代码可以发现，实际建立JDBC连接的时候
      # 会先去获取option里的user配置项，如果没有这个配置项的话，
      # 就会使用当前的用户名去创建JDBC连接了。这样也就实现了hiveserver的
      # doAs类似的功能，即以真实执行用户的身份执行最终查询
      username = self.options.get('user') or user.username
      #username = self.user
      print('-------------------got  username --------------------------------- %s' % username)
      self.db = API_CACHE[self.cache_key] = Jdbc(self.options['driver'], self.options['url'], username, self.options['password'])

  def create_session(self, lang=None, properties=None):
    ...
    if self.db is None:
      if 'password' in properties:
        user = properties.get('user') or self.options.get('user')
        props['properties'] = {'user': user}
        self.db = API_CACHE[self.cache_key] = Jdbc(self.options['driver'], self.options['url'], user, properties.pop('password'))
    ...
    return props

```
从上面的代码分析可得，我们只要在option里不配置user信息，就可以了。。。（结果让我相对崩溃.....）


#### 升级

从3.10到4.1，压根儿不知道应该怎样升级数据库里的数据。搜了很多文档都没有找到，最后到github上找到cdh/hue的源码，看到有个commit的comment是关于upgrade的，点进去发现了一篇文档：https://github.com/cloudera/hue/tree/master/dist。里面提到以下三个步骤:

- 备份原有数据库
- ./build/env/bin/hue syncdb
- ./build/env/bin/hue migrate

其实在第二个步骤的时候，就会进行数据库比对。而第三个数据库则会把所有4.1多余3.10的migrations进行执行。因为hue是基于django的，所以利用了django的migrate的特性。

不知道为什么一直没有搜到这个hue的文档，可能还是关键字有问题吧。

#### 进一步思考

其实现在presto是可用了，但是依照规范的话，我们是可以自己依照hive、mysql的数据查询类实现presto查询类的。而且难度也基本是很低的，hue这么长时间都没有presto的兼容，不知道是不是官网别有用心[impala]。

#### 参考
- https://github.com/skame/docker-prestogres
- https://github.com/treasure-data/prestogres
- https://medium.com/@ilkkaturunen/integrating-presto-with-hue-61702b244839
- https://prestodb.io/docs/0.184/installation/jdbc.html
- https://groups.google.com/forum/#!topic/presto-users/24MMrfLKdu0
- https://github.com/cloudera/hue/blob/release-4.1.0/desktop/libs/notebook/src/notebook/connectors/jdbc.py
