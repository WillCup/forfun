
title: ambari源码解析-告警流程
date: 2017-10-31 19:23:25
tags: [youdaonote]
---

从ambari的包找起，找到一个alert的package，下面有一个类StaleAlertRunnable，看一下注释：。那么谁调用它呢？AmbariServerAlertService，与它在同一个目录的还有一个名字很显眼的类AlertNoticeDispatchService。

它是用来扫描表`alert_notice`中状态为pending的记录，并调用各种dispatcher执行告警分发的。那么谁把notice放进这个表里的呢？没有追踪到。

不过我们发现它继承自guava的`AbstractScheduledService`类，那么就搜一下这个类的使用，发现在ControllerModule里出现，而ControllerModule是出现在AmbariServer.main之中的。


```java
Injector injector = Guice.createInjector(new ControllerModule());

```

使用工具guice执行了ControllerModule中的依赖注入工作。我们知道Guice.createInjector接收`AbstractModule`接口的类，一般我们需要实现`AbstractModule`接口的`configure`方法。然后在`configure`方法中将依赖的接口与其具体实现进行绑定，也就是执行`bind(xxx.class).to(xxx.class)`等工作。下面我们就看一下ControllerModule()做了哪些依赖注入声明的工作。

```
protected void configure() {
    // 各种工厂接口与其实现类的绑定
    installFactories();


    // 绑定session相关manager到具体实例
    final SessionIdManager sessionIdManager = new HashSessionIdManager();
    final SessionManager sessionManager = new HashSessionManager();
    sessionManager.getSessionCookieConfig().setPath("/");
    sessionManager.setSessionIdManager(sessionIdManager);
    bind(SessionManager.class).toInstance(sessionManager);
    bind(SessionIdManager.class).toInstance(sessionIdManager);

    // kerberos相关
    bind(KerberosOperationHandlerFactory.class);
    bind(KerberosDescriptorFactory.class);
    bind(KerberosServiceDescriptorFactory.class);
    bind(KerberosHelper.class).to(KerberosHelperImpl.class);

    bind(CredentialStoreService.class).to(CredentialStoreServiceImpl.class);

    bind(Configuration.class).toInstance(configuration);
    bind(OsFamily.class).toInstance(os_family);
    bind(HostsMap.class).toInstance(hostsMap);
    bind(PasswordEncoder.class).toInstance(new StandardPasswordEncoder());
    bind(DelegatingFilterProxy.class).toInstance(new DelegatingFilterProxy() {
      {
        setTargetBeanName("springSecurityFilterChain");
      }
    });

    bind(Gson.class).annotatedWith(Names.named("prettyGson")).toInstance(prettyGson);
    
    // jpa持久化相关
    install(buildJpaPersistModule());

    bind(Gson.class).in(Scopes.SINGLETON);
    bind(SecureRandom.class).in(Scopes.SINGLETON);

    bind(Clusters.class).to(ClustersImpl.class);
    bind(AmbariCustomCommandExecutionHelper.class);
    bind(ActionDBAccessor.class).to(ActionDBAccessorImpl.class);
    bindConstant().annotatedWith(Names.named("schedulerSleeptime")).to(
      configuration.getExecutionSchedulerWait());

    // This time is added to summary timeout time of all tasks in stage
    // So it's an "additional time", given to stage to finish execution before
    // it is considered as timed out
    bindConstant().annotatedWith(Names.named("actionTimeout")).to(600000L);

    bindConstant().annotatedWith(Names.named("dbInitNeeded")).to(dbInitNeeded);
    bindConstant().annotatedWith(Names.named("statusCheckInterval")).to(5000L);

    //ExecutionCommands cache size

    bindConstant().annotatedWith(Names.named("executionCommandCacheSize")).
        to(configuration.getExecutionCommandsCacheSize());


    // Host role commands status summary max cache enable/disable
    bindConstant().annotatedWith(Names.named(HostRoleCommandDAO.HRC_STATUS_SUMMARY_CACHE_ENABLED)).
      to(configuration.getHostRoleCommandStatusSummaryCacheEnabled());

    // Host role commands status summary max cache size
    bindConstant().annotatedWith(Names.named(HostRoleCommandDAO.HRC_STATUS_SUMMARY_CACHE_SIZE)).
      to(configuration.getHostRoleCommandStatusSummaryCacheSize());
    // Host role command status summary cache expiry duration in minutes
    bindConstant().annotatedWith(Names.named(HostRoleCommandDAO.HRC_STATUS_SUMMARY_CACHE_EXPIRY_DURATION_MINUTES)).
      to(configuration.getHostRoleCommandStatusSummaryCacheExpiryDuration());




    bind(AmbariManagementController.class).to(
      AmbariManagementControllerImpl.class);
    bind(AbstractRootServiceResponseFactory.class).to(RootServiceResponseFactory.class);
    bind(ExecutionScheduler.class).to(ExecutionSchedulerImpl.class);
    bind(DBAccessor.class).to(DBAccessorImpl.class);
    bind(ViewInstanceHandlerList.class).to(AmbariHandlerList.class);
    bind(TimelineMetricCacheProvider.class);
    bind(TimelineMetricCacheEntryFactory.class);
    bind(SecurityConfigurationFactory.class).in(Scopes.SINGLETON);

    bind(PersistedState.class).to(PersistedStateImpl.class);

    requestStaticInjection(ExecutionCommandWrapper.class);
    requestStaticInjection(DatabaseChecker.class);
    requestStaticInjection(KerberosChecker.class);
    
    // 这里是ControllerModule自定义的一个方法，根据注解执行绑定工作
    bindByAnnotation(null);
    
    // 这里是我们关心的dispatcher相关绑定
    bindNotificationDispatchers();
    registerUpgradeChecks();
  }
```

我们发现serviceManager就在这个类中出现。它的说明是为需要注入的带有某种注解的接口提供初始化工作。典型应用一：是用来处理一些没有入口的单例，换种表达方式，就是那些没有被任何接口依赖注入，但是仍然被整体的guice框架需要的类。典型应用二：某些类需要注入静态static成员的时候。传入的参数beanDefinitions为空的时候，会默认扫描带有注解`EagerSingleton`,`StaticallyInject`,`AmbariService`的类。这三个注解都是ambari自定义的。我们看到上面调用这个方法的时候传入参数是null。
```java
protected Set<BeanDefinition> bindByAnnotation(Set<BeanDefinition> beanDefinitions) {
    List<Class<? extends Annotation>> classes = Arrays.asList(
        EagerSingleton.class, StaticallyInject.class, AmbariService.class);

    if (null == beanDefinitions || beanDefinitions.size() == 0) {
      ClassPathScanningCandidateComponentProvider scanner =
          new ClassPathScanningCandidateComponentProvider(false);

      // match only singletons that are eager listeners
      // 为scanner加入目标注解过滤器
      for (Class<? extends Annotation> cls : classes) {
        scanner.addIncludeFilter(new AnnotationTypeFilter(cls));
      }
      // 找出所有的带有注解的类
      beanDefinitions = scanner.findCandidateComponents(AMBARI_PACKAGE);
    }

    if (null == beanDefinitions || beanDefinitions.size() == 0) {
      LOG.warn("No instances of {} found to register", classes);
      return beanDefinitions;
    }

    Set<com.google.common.util.concurrent.Service> services =
        new HashSet<com.google.common.util.concurrent.Service>();

    for (BeanDefinition beanDefinition : beanDefinitions) {
      String className = beanDefinition.getBeanClassName();
      Class<?> clazz = ClassUtils.resolveClassName(className,
          ClassUtils.getDefaultClassLoader());

      if (null != clazz.getAnnotation(EagerSingleton.class)) {
        bind(clazz).asEagerSingleton();
        LOG.debug("Binding singleton {} eagerly", clazz);
      }

      if (null != clazz.getAnnotation(StaticallyInject.class)) {
        requestStaticInjection(clazz);
        LOG.debug("Statically injecting {} ", clazz);
      }

      // Ambari services are registered with Guava
      // 使用Guava的AmbariService注解类
      if (null != clazz.getAnnotation(AmbariService.class)) {
        // 看看是不是一个Guava service
        if (!AbstractScheduledService.class.isAssignableFrom(clazz)) {
          String message = MessageFormat.format(
              "Unable to register service {0} because it is not an AbstractScheduledService",
              clazz);

          LOG.warn(message);
          throw new RuntimeException(message);
        }

        // 实例化，并作为单例注入到guice中，然后再添加到services中
        AbstractScheduledService service = null;
        try {
          service = (AbstractScheduledService) clazz.newInstance();
          bind((Class<AbstractScheduledService>) clazz).toInstance(service);
          services.add(service);
          LOG.debug("Registering service {} ", clazz);
        } catch (Exception exception) {
          LOG.error("Unable to register {} as a service", clazz, exception);
          throw new RuntimeException(exception);
        }
      }
    }
    // 使用上面扫描完获取到的所有services，实例化serviceManager，并单例化注入guice
    ServiceManager manager = new ServiceManager(services);
    bind(ServiceManager.class).toInstance(manager);

    return beanDefinitions;
  }
```

ServiceManager作为AmbariServer的依赖项被注入，然后在run方法中被调用其startAsync方法，我们看到其实是遍历调用了services中的service的startAsync方法：
```
  public ServiceManager startAsync() {
    for (Service service : services) {
      State state = service.state();
      checkState(state == NEW, "Service %s is %s, cannot start it.", service, state);
    }
    for (Service service : services) {
      try {
        service.startAsync();
      } catch (IllegalStateException e) {
      }
    }
    return this;
  }
```


参考：https://github.com/google/guice/wiki/GettingStarted


看了这么多，但是实际alert来源呢？一般都是agent在本地执行检查，符合告警规则的才发送给server端。那么是怎样发送到server的呢？首先想到的是rest api里的AlertNoticeService，可是这个类只有get方法，没有POST接口。那么下一个嫌疑就是HeartBeat信息了。看下HeartBeat的定义
```
private long responseId = -1;
private long timestamp;
private String hostname;
List<CommandReport> reports = new ArrayList<CommandReport>();
List<ComponentStatus> componentStatus = new ArrayList<ComponentStatus>();
private List<DiskInfo> mounts = new ArrayList<DiskInfo>();
HostStatus nodeStatus;
private AgentEnv agentEnv = null;
private List<Alert> alerts = null;
private RecoveryReport recoveryReport;
```

果然，那么我们就也能找到HeartBeat相关的resource了：AgentResource。HeartBeatHandler处理过程中会把heartbeat放进HeartbeatProcessor的队列里，做异步处理，告警部分的持久化就在这个异步处理的代码里了。
```java
@Path("heartbeat/{hostName}")
  @POST
  @Consumes(MediaType.APPLICATION_JSON)
  @Produces({MediaType.APPLICATION_JSON})
  public HeartBeatResponse heartbeat(HeartBeat message)
      throws WebApplicationException {
    if (LOG.isDebugEnabled()) {
      LOG.debug("Received Heartbeat message " + message);
    }
    HeartBeatResponse heartBeatResponse;
    try {
    // hh是一个单例的HeartBeatHandler
      heartBeatResponse = hh.handleHeartBeat(message);
      if (LOG.isDebugEnabled()) {
        LOG.debug("Sending heartbeat response with response id " + heartBeatResponse.getResponseId());
        LOG.debug("Response details " + heartBeatResponse);
      }
    } catch (Exception e) {
      LOG.warn("Error in HeartBeat", e);
      throw new WebApplicationException(500);
    }
    return heartBeatResponse;
  }
```

中间经过guava的eventbus的多次事件中转后，在AlertStateChangedListener中被完全持久化进入数据库中。

```
public void onAlertEvent(AlertStateChangeEvent event) {
    ...
    ...
    for (AlertGroupEntity group : groups) {
      Set<AlertTargetEntity> targets = group.getAlertTargets();
      if (null == targets || targets.size() == 0) {
        continue;
      }

      for (AlertTargetEntity target : targets) {
        if (!isAlertTargetInterested(target, history)) {
          continue;
        }

        AlertNoticeEntity notice = new AlertNoticeEntity();
        notice.setUuid(UUID.randomUUID().toString());
        notice.setAlertTarget(target);
        notice.setAlertHistory(event.getNewHistoricalEntry());
        notice.setNotifyState(NotificationState.PENDING);
        notices.add(notice);
      }
    }
    m_alertsDispatchDao.createNotices(notices);
  }
```

启动HeartBeatHandler的地方出现在AmbariServer.run的代码中
```java
....
AgentResource.statHeartBeatHandler();
....
```

至于agent那边具体实现，我们暂时不太关心，因为暂时不需要自定义alert。


综上，对于告警渠道的添加，我们需要自己实现guava的`AbstractScheduledService`抽象类接口，依照既有的SNNP或者EMAIL等写Dispatcher，然后把它们放置到`ServiceManager.AMBARI_PACKAGE`的位置。然后，最好是通过ambari提供的REST API添加alert_target，因为要处理跟alert_group相关的事情，自己直接操作数据库比较复杂一些，基本上要复制一遍代码里的所有操作。


大功告成！



