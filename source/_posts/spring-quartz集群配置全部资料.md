title: spring + quartz集群配置全部资料
date: 2015-12-10 15:25:25
tags: [spring, quartz]
---

今天玩儿quartz正玩儿的起劲儿，下午就被否掉了，另一个同事已经被分配了相关的工作，那么我这边的工作就重复了。
暂时记下目前了解到的一些配置，方便以后使用。

### quartz.xml配置文件
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans
        xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id = "willtest" class="com.will.quartz.task.WillTask"/>

    <bean id="abnormalDitail" class="org.springframework.scheduling.quartz.JobDetailBean">
        <property name="jobClass" value="com.will.quartz.task.WillSpringJob" />
        <property name="jobDataAsMap">
            <map>
                <entry key="timeout" value="5" />
            </map>
        </property>
    </bean>
    <bean id="anotherJob" class="org.springframework.scheduling.quartz.JobDetailBean">
        <property name="jobClass" value="com.will.quartz.task.AnotherSJob" />
        <property name="jobDataAsMap">
            <map>
                <entry key="timeout" value="5" />
            </map>
        </property>
    </bean>

    <!--  调度触发器 -->
    <bean id="abnormalTigger" class="org.springframework.scheduling.quartz.CronTriggerBean">
        <property name="jobDetail" ref="abnormalDitail"/>
        <property name="cronExpression">
            <value>*/5 * * * * ?</value>
        </property>
    </bean>
    <!--  调度触发器 -->
    <bean id="anotherJobTigger" class="org.springframework.scheduling.quartz.CronTriggerBean">
        <property name="jobDetail" ref="anotherJob"/>
        <property name="cronExpression">
            <value>*/8 * * * * ?</value>
        </property>
    </bean>

    <bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
        <property name="driverClass" value="${database.driverClassName}" />
        <property name="jdbcUrl" value="${database.quartz.url}" />
        <property name="user" value="${database.quartz.username}" />
        <property name="password" value="${database.quartz.password}" />
        <property name="initialPoolSize" value="${pool.initialPoolSize}" />
        <property name="minPoolSize" value="${pool.minPoolSize}" />
        <property name="maxPoolSize" value="${pool.maxPoolSize}" />
        <property name="maxIdleTime" value="${pool.maxIdleTime}" />
        <property name="checkoutTimeout" value="12000" />
        <property name="acquireIncrement" value="${pool.acquireIncrement}" />
    </bean>

    <bean  id="quartzExceptionSchedulerListener"  class="com.will.quartz.task.QuartzExceptionSchedulerListener"/>

<!-- 调度工厂 -->
<bean id="quartzScheduler"
      class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
       <property name="dataSource" ref="dataSource" />
       <property name="quartzProperties">
              <props>
                     <prop key="org.quartz.scheduler.instanceName">scheduler</prop>
                  <prop key="org.quartz.scheduler.driverDelegateClass">org.quartz.impl.jdbcjobstore.StdJDBCDelegate</prop>
                  <prop key="org.quartz.scheduler.instanceId">AUTO</prop>
                     <!-- 线程池配置 -->
                     <prop key="org.quartz.threadPool.class">org.quartz.simpl.SimpleThreadPool</prop>
                     <prop key="org.quartz.threadPool.threadCount">20</prop>
                     <prop key="org.quartz.threadPool.threadPriority">5</prop>
                     <!-- JobStore 配置 -->
                     <prop key="org.quartz.jobStore.class">org.quartz.impl.jdbcjobstore.JobStoreTX</prop>

                     <!-- 集群配置 -->
                     <prop key="org.quartz.jobStore.isClustered">true</prop>
                     <prop key="org.quartz.jobStore.clusterCheckinInterval">15000</prop>
                     <prop key="org.quartz.jobStore.maxMisfiresToHandleAtATime">1</prop>

                     <prop key="org.quartz.jobStore.misfireThreshold">120000</prop>

                     <prop key="org.quartz.jobStore.tablePrefix">QRTZ_</prop>
              </props>

       </property>

       <property name="schedulerName" value="CRMscheduler" />

       <!--必须的，QuartzScheduler 延时启动，应用启动完后 QuartzScheduler 再启动 -->
       <property name="startupDelay" value="30" />

       <property name="applicationContextSchedulerContextKey" value="applicationContextKey" />

       <!--可选，QuartzScheduler 启动时更新己存在的Job，这样就不用每次修改targetObject后删除qrtz_job_details表对应记录了 -->
       <property name="overwriteExistingJobs" value="true" />

       <!-- 设置自动启动 -->
       <property name="autoStartup" value="true" />


       <!-- 注册触发器 -->
       <property name="triggers">
              <list>
                     <ref bean="abnormalTigger" />
                     <ref bean="anotherJobTigger" />
              </list>
       </property>

       <!-- 注册jobDetail -->
       <property name="jobDetails">
              <list>
              </list>
       </property>


       <property name="schedulerListeners">
              <list>
                     <ref bean="quartzExceptionSchedulerListener" />
              </list>
       </property>
</bean>
</beans>
```

### 数据库表准备
就是准备好集群需要的那些表，昨天找了半天的那个，参见：spring-quartz集群环境搭建遇到的问题。

### config.properties
就是xml里使用的那些变量
```java
base.driverClassName=com.mysql.jdbc.Driver
database.quartz.url=jdbc:mysql://192.168.22.235:3306/testdb?useUnicode=true&characterEncoding=utf8
database.quartz.username=root
database.quartz.password=123


pool.initialPoolSize=5
pool.minPoolSize=5
pool.maxPoolSize=100
pool.maxIdleTime=3600
pool.acquireIncrement=1
pool.idleConnectionTestPeriod=60
pool.preferredTestQuery=SELECT 1
pool.checkoutTimeout=60000
```

### JobDetail
也就是我们需要定时执行的任务。
```java
public class WillSpringJob extends QuartzJobBean {

    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        System.out.println("sssssssssssssssss");
    }
}
```

```java
public class AnotherSJob extends QuartzJobBean {
    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        System.out.println("fun.. I am a jerk.");
    }
}
```

### SchedulerListener
```java
public class QuartzExceptionSchedulerListener extends SchedulerListenerSupport {
    private Logger logger = LoggerFactory.getLogger(QuartzExceptionSchedulerListener.class);
    @Override
    public void schedulerError(String message, SchedulerException e) {
        super.schedulerError(message, e);
        logger.error(message, e.getUnderlyingException());
    }
}
```

### 其他
其实主要的配置都在SchedulerFactoryBean这儿，我们可以看一下这个类里到底有哪些element，也就是xml里在它下面配置的property。其实主要是他的父类SchedulerAccessor，展示一下源码截图：

![](/imgs/quartz/SchedulerAccessor.png)

### 几个小问题

- 1. quartz虽然启动的时候会delay，但是delay的时候会囤积task，到时间会一股脑地将前面delay那段时间的Task全给执行咯；
结果是：是的。不可避免
- 2. 目前一个trigger里只能写一个job，看了一下cronTrigger里就只有一个JobDetail属性，而不是一个list。对于同一个调度频率的任务一定需要弄多个trigger吗？
结果是：很少有这种需求，可以写成一个Job
- 3. SchedulerFactoryBean里的jobDetails如果不配置，只设置trigger的话,JobDetail其实也是会被写入mysql对应的表里的。trigger里的JobDetail和SchedulerFactoryBean里的有什么区别和联系。
结果是： ScheduelrAccessor里的注释是Register a list of JobDetail objects with the Scheduler that this FactoryBean creates, to be referenced by Triggers.
This is not necessary when a Trigger determines the JobDetail itself: In this case, the JobDetail will be implicitly registered in combination with the Trigger.

Done!! 

### 参考

- http://tech.meituan.com/mt-crm-quartz.html
- https://www.ibm.com/developerworks/cn/opensource/os-cn-quartz/
- http://www.quartz-scheduler.org/generated/2.2.2/html/qtz-all/
- https://github.com/xishuixixia/quartz-monitor 【quartz monitor】
- http://hot66hot.iteye.com/blog/1726143




