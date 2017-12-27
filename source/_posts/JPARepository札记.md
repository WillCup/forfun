
title: JPARepository札记
date: 2016-12-05 17:08:54
tags: [youdaonote]
---

join时报错
---
报错
```
QuerySyntaxException: expecting IDENT, found '*' near line 1
```
原始sql
```sql
hql:select a.* from Archive a,Employee e,Department d where a.contractEndDate = :date and a.status = 'A' and a.empId = e.id and e.status = 'A' and e.type = 'A'
```
改成
```sql
select a from Archive a,Employee e,Department d where a.contractEndDate = :date and a.status = 'A' and a.empId = e.id and e.status = 'A' and e.type = 'A'
```

select 后面为对象 a .而非 a.*

参考：
http://blog.sina.com.cn/s/blog_4bf2e5550100n2gh.html

SQL面向Entity
---
指的是在JPARepository的子类中自定义的函数
```java
public interface TblBloodRepository extends JpaRepository<TblBlood, Integer>{
    TblBlood findByTblName(String tblName);

    @Query("select a from "
            + "(select blood from TblBlood blood where valid = 1) a"
            + " left outer join "
            + "(select distinct parentTbl from TblBlood where valid = 1) b"
            + " on a.tblName = b.parentTbl"
            + " where b.parentTbl is null")
    List<TblBlood> selectAllLeaf();

    @Modifying
    @Query("update TblBlood set valid = 0 where tblName=:tblName")
    void makePreviousInvalid(@Param("tblName")String tblName);

    @Query("select b from"
            + " TblBlood a join TblBlood b"
            + " on a.parentTbl = b.tblName and b.valid = 1"
            + " where a.valid = 1 and a.tblName = :tblName")
    List<TblBlood> selectParentByTblName(@Param("tblName")String tblName);

    @Query("from TblBlood where valid = 1 and parentTbl=:tblName")
    public List<TblBlood> selectByParentTblName(@Param("tblName")String tblName);

    @Query("from TblBlood where valid = 1 and tblName=:tblName")
    List<TblBlood> selectByTblName(@Param("tblName")String tblName);
}

```
TblBlood就是对象的类名，而不是真正的表名。

自join
---
```java
@Query("select b from"
            + " TblBlood a join TblBlood b"
            + " on a.parentTbl = b.tblName and b.valid = 1"
            + " where a.valid = 1 and a.tblName = ?1")
    List<TblBlood> selectParentByTblName(String tblName);
```
报错
```
  Path expected for join!
	at org.hibernate.hql.internal.ast.HqlSqlWalker.createFromJoinElement(HqlSqlWalker.java:378)
	at org.hibernate.hql.internal.antlr.HqlSqlBaseWalker.joinElement(HqlSqlBaseWalker.java:3858)
	at org.hibernate.hql.internal.antlr.HqlSqlBaseWalker.fromElement(HqlSqlBaseWalker.java:3644)
	at org.hibernate.hql.internal.antlr.HqlSqlBaseWalker.fromElementList(HqlSqlBaseWalker.java:3522)
 
 org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'tblBloodRepository': Invocation of init method failed; nested exception is java.lang.IllegalArgumentException: Validation failed for query for method public abstract java.util.List com.will.hivesolver.repositories.TblBloodRepository.selectParentByTblName(java.lang.String)!
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1574)
	...
Caused by: java.lang.IllegalArgumentException: Validation failed for query for method public abstract java.util.List com.will.hivesolver.repositories.TblBloodRepository.selectParentByTblName(java.lang.String)!
	at org.springframework.data.jpa.repository.query.SimpleJpaQuery.validateQuery(SimpleJpaQuery.java:84)
	.....
Caused by: java.lang.IllegalStateException: No data type for node: org.hibernate.hql.internal.ast.tree.IdentNode 
 \-[IDENT] IdentNode: 'b' {originalText=b}

	at org.hibernate.hql.internal.ast.tree.SelectClause.initializeExplicitSelectClause(SelectClause.java:174)
	at org.hibernate.hql.internal.ast.HqlSqlWalker.useSelectClause(HqlSqlWalker.java:923)
	...

```

不能更新
---
```
严重: Servlet.service() for servlet [api] in context with path [/metamap] threw exception [Request processing failed; nested exception is org.springframework.dao.InvalidDataAccessApiUsageException: Executing an update/delete query; nested exception is javax.persistence.TransactionRequiredException: Executing an update/delete query] with root cause
javax.persistence.TransactionRequiredException: Executing an update/delete query
```
Spring Data JPA，事务导致的异常

需要在springMVC的xml里添加,这里只扫描Controller所在的位置里的Controller
```xml
<context:component-scan base-package="com.will.hivesolver.controller" use-default-filters="false">
        <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
        <context:include-filter type="annotation" expression="org.springframework.web.bind.annotation.ControllerAdvice"/>
    </context:component-scan>
```
在applicationContext.xml中添加，这里必须不能扫描上面扫描过的Controller：
```xml
<context:component-scan base-package="com.will.hivesolver">
		<context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
		<context:exclude-filter type="annotation" expression="org.springframework.web.bind.annotation.ControllerAdvice"/>
	</context:component-scan>
```
参考：https://github.com/springside/springside4/wiki/Spring-MVC


 No property jobId found for type Execution!惨案
 ---

Could not determine type for  at table


紧接着Could not determine type for: java.util.Set


参考：
- http://stackoverflow.com/questions/15845030/no-property-u-found-for-type-com-models-entities-orderentity
- http://stackoverflow.com/questions/15849278/org-hibernate-mappingexception-could-not-determine-type-for-java-util-set-at




failed to lazily initialize a collection of role: , could not initialize proxy - no Session
---
著名的延迟加载问题。
原因是One加载完之后，session就已经关闭了，此时再去请求many，就么有session了。
最终采用的方案是：在web.xml里添加spring的OpenSessionInViewFilter。

```xml
<filter>  
       <filter-name>OpenSessionInViewFilter</filter-name>  
       <filter-class>  
           org.springframework.orm.hibernate3.support.OpenSessionInViewFilter
       </filter-class>
       <init-param>
            <param-name>sessionFactoryBeanName</param-name>
            <param-value>sessionfactory</param-value>
        </init-param> 
    </filter>  
    <filter-mapping>  
       <filter-name>OpenSessionInViewFilter</filter-name>  
       <url-pattern>/*</url-pattern> 
</filter-mapping>
```

No persistence units parsed from {classpath*:META-INF/persistence.xml

参考：
- http://blog.csdn.net/elfenliedef/article/details/6011892
- http://lazycat0102.blog.51cto.com/8938702/1584779
- http://www.cnblogs.com/luxh/archive/2012/05/24/2516282.html

