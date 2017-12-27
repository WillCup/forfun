
title: ambari源码解析-资源状态监听
date: 2017-10-31 17:33:00
tags: [youdaonote]
---


AmbariServie.run()入口处找到
```
...
LOG.info("********* Reconciling Alert Definitions **********");
ambariMetaInfo.reconcileAlertDefinitions(clusters);
...

```

调用了AmbariMetaInfo.reconcileAlertDefinitions(Clusters clusters)方法，这个方法主要负责比较stack中定义的alert和数据库中的alert有没有什么变化。它会首先确认一下alert定义所在的service是不是已经被安装了，没有安装的就没有必要关心了。

这个方法还会监测agent定义的alert，这些应该在agent host上运行。

```java
public void reconcileAlertDefinitions(Clusters clusters)
      throws AmbariException {

    Map<String, Cluster> clusterMap = clusters.getClusters();
    if (null == clusterMap || clusterMap.size() == 0) {
      return;
    }

    // 遍历cluster
    for (Cluster cluster : clusterMap.values()) {
      long clusterId = cluster.getClusterId();
      StackId stackId = cluster.getDesiredStackVersion();
      StackInfo stackInfo = getStack(stackId.getStackName(),
          stackId.getStackVersion());

      // 创建service/conponent和名字的映射，方便查找
      Collection<ServiceInfo> stackServices = stackInfo.getServices();
      Map<String, ServiceInfo> stackServiceMap = new HashMap<String, ServiceInfo>();
      Map<String, ComponentInfo> stackComponentMap = new HashMap<String, ComponentInfo>();
      for (ServiceInfo stackService : stackServices) {
        stackServiceMap.put(stackService.getName(), stackService);

        List<ComponentInfo> components = stackService.getComponents();
        for (ComponentInfo component : components) {
          stackComponentMap.put(component.getName(), component);
        }
      }

      Map<String, Service> clusterServiceMap = cluster.getServices();
      Set<String> clusterServiceNames = clusterServiceMap.keySet();

      // 对于当前cluster上已经安装的所有service，获取其metainfo和alert定义， 添加到stackDefinitions
      List<AlertDefinition> stackDefinitions = new ArrayList<AlertDefinition>(50);
      for (String clusterServiceName : clusterServiceNames) {
        ServiceInfo stackService = stackServiceMap.get(clusterServiceName);
        if (null == stackService) {
          continue;
        }

        Set<AlertDefinition> serviceDefinitions = getAlertDefinitions(stackService);
        stackDefinitions.addAll(serviceDefinitions);
      }

      // 抽取数据库中所有的alert定义，把list转化为以alert定义名字为key的map
      List<AlertDefinitionEntity> persist = new ArrayList<AlertDefinitionEntity>();
      List<AlertDefinitionEntity> entities = alertDefinitionDao.findAll(clusterId);

      Map<String, AlertDefinitionEntity> mappedEntities = new HashMap<String, AlertDefinitionEntity>(100);
      for (AlertDefinitionEntity entity : entities) {
        mappedEntities.put(entity.getDefinitionName(), entity);
      }

      // 对每个stack definition，查看是否存在，如果存在，就详细比较一下变化
      for( AlertDefinition stackDefinition : stackDefinitions ){
        AlertDefinitionEntity entity = mappedEntities.get(stackDefinition.getName());

        // 如果是null，就代表是新的
        if (null == entity) {
          entity = alertDefinitionFactory.coerce(clusterId, stackDefinition);
          persist.add(entity);
          continue;
        }

        // 修改过的stack中的定义不会被覆盖，需要使用REST API来修改
        AlertDefinition databaseDefinition = alertDefinitionFactory.coerce(entity);
        
        // 如果stackfinition与数据库中不相等....暂时没有处理任何东西，只是打了个log
        if (!stackDefinition.deeplyEquals(databaseDefinition)) {
          // this is the code that would normally merge the stack definition
          // into the database; this is not the behavior we want today

          // entity = alertDefinitionFactory.merge(stackDefinition, entity);
          // persist.add(entity);

          LOG.debug(
              "The alert named {} has been modified from the stack definition and will not be merged",
              stackDefinition.getName());
        }
      }

      // 把ambari agent主机上的alert定义添加到持久化列表里
      List<AlertDefinition> agentDefinitions = ambariServiceAlertDefinitions.getAgentDefinitions();
      for (AlertDefinition agentDefinition : agentDefinitions) {
        AlertDefinitionEntity entity = mappedEntities.get(agentDefinition.getName());

        // no entity means this is new; create a new entity
        if (null == entity) {
          entity = alertDefinitionFactory.coerce(clusterId, agentDefinition);
          persist.add(entity);
        }
      }

      // 把ambari server 的alert定义添加到持久化列表里
      List<AlertDefinition> serverDefinitions = ambariServiceAlertDefinitions.getServerDefinitions();
      for (AlertDefinition serverDefinition : serverDefinitions) {
        AlertDefinitionEntity entity = mappedEntities.get(serverDefinition.getName());

        // no entity means this is new; create a new entity
        if (null == entity) {
          entity = alertDefinitionFactory.coerce(clusterId, serverDefinition);
          persist.add(entity);
        }
      }

      // 持久化新的，或者需要更新的alert定义
      for (AlertDefinitionEntity entity : persist) {
        if (LOG.isDebugEnabled()) {
          LOG.info("Merging Alert Definition {} into the database",
              entity.getDefinitionName());
        }
        alertDefinitionDao.createOrUpdate(entity);
      }

      // all definition resolved; publish their registration
      // 所有的定义都处理完成后，发布它们的注册event也就是AlertDefinitionRegistrationEvent到eventPublisher
      for (AlertDefinitionEntity def : alertDefinitionDao.findAll(cluster.getClusterId())) {
        AlertDefinition realDef = alertDefinitionFactory.coerce(def);

        AlertDefinitionRegistrationEvent event = new AlertDefinitionRegistrationEvent(
            cluster.getClusterId(), realDef);

        eventPublisher.publish(event);
      }

      // for every definition, determine if the service and the component are
      // still valid; if they are not, disable them - this covers the case
      // with STORM/REST_API where that component was removed from the
      // stack but still exists in the database - we disable the alert to
      // preserve historical references
      List<AlertDefinitionEntity> definitions = alertDefinitionDao.findAllEnabled(clusterId);
      List<AlertDefinitionEntity> definitionsToDisable = new ArrayList<AlertDefinitionEntity>();

      for (AlertDefinitionEntity definition : definitions) {
        String serviceName = definition.getServiceName();
        String componentName = definition.getComponentName();

        // the AMBARI service is special, skip it here
        if (Services.AMBARI.name().equals(serviceName)) {
          continue;
        }

        if (!stackServiceMap.containsKey(serviceName)) {
          LOG.info(
              "The {} service has been marked as deleted for stack {}, disabling alert {}",
              serviceName, stackId, definition.getDefinitionName());

          definitionsToDisable.add(definition);
        } else if (null != componentName
            && !stackComponentMap.containsKey(componentName)) {
          LOG.info(
              "The {} component {} has been marked as deleted for stack {}, disabling alert {}",
              serviceName, componentName, stackId,
              definition.getDefinitionName());

          definitionsToDisable.add(definition);
        }
      }

      // disable definitions and fire the event
      // 将alert定义diable掉，并发送这个event到eventPublisher
      for (AlertDefinitionEntity definition : definitionsToDisable) {
        definition.setEnabled(false);
        alertDefinitionDao.merge(definition);

        AlertDefinitionDisabledEvent event = new AlertDefinitionDisabledEvent(
            clusterId, definition.getDefinitionId());

        eventPublisher.publish(event);
      }
    }
  }
```


好了，上面看到所有的alert定义都处理完后，会发送一个AlertDefinitionRegistrationEvent事件到AmbariEventPublisher。那么我们看下有谁监听了AmbariEventPublisher的AlertDefinitionRegistrationEvent事件就可以了吧。然后就定位到了AlertLifecycleListener。



尼玛，这好像不是具体执行alert的地方。。。。而是管理ambari中各种资源的变化情况的地方。具体可以看下AmbariEvent的实现类列表~~~基本都是维护ambari中各种概念的状态变化的。






