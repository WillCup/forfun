
title: ambari源码解析-REST解析
date: 2017-10-31 16:32:24
tags: [youdaonote]
---

先看下入口的地方, ambariServer.run()：
```java
ServletHolder sh = new ServletHolder(ServletContainer.class);
sh.setInitParameter("com.sun.jersey.config.property.resourceConfigClass",
  "com.sun.jersey.api.core.PackagesResourceConfig");

sh.setInitParameter("com.sun.jersey.config.property.packages",
"org.apache.ambari.server.api.rest;" +
  "org.apache.ambari.server.api.services;" +
  "org.apache.ambari.eventdb.webservice;" +
  "org.apache.ambari.server.api");

sh.setInitParameter("com.sun.jersey.api.json.POJOMappingFeature", "true");
root.addServlet(sh, "/api/v1/*");
sh.setInitOrder(2);
```
就是`ServletHolder.setInitParameter`里`com.sun.jersey.config.property.packages`对应所有的包路径下的类都被jettyserver看作rest资源，然后所有的带有jersey的REST注解的方法，就都是对应一个path的了。




然后看下ambari自己定制的东西，拿我关注的alert_targets接口来看,对应类为AlertTargetService，看POST方法
```java
  @POST
  @Produces("text/plain")
  public Response createTarget(String body, @Context HttpHeaders headers,
      @Context UriInfo ui) {
    return handleRequest(headers, body, ui, Request.Type.POST,
        createAlertTargetResource(null));
  }


  private ResourceInstance createAlertTargetResource(Long targetId) {
    Map<Resource.Type, String> mapIds = new HashMap<Resource.Type, String>();

    mapIds.put(Resource.Type.AlertTarget,
        null == targetId ? null : targetId.toString());

    return createResource(Resource.Type.AlertTarget, mapIds);
  }
```

ambari中所有的Service都继承自BaseService，上面的`createAlertTargetResource`调用的`createResource`以及`handleRequest`都是继承自BaseService的。

BaseService的`createResource`方法实现，下面的type应该是Resource.Type.AlertTarget，是一个枚举值。
```
 public ResourceInstance createResource(Resource.Type type, Map<Resource.Type, String> mapIds) {

    /**
     * 找到type对应的ResourceDefinition， 这里返回的是AlertTargetResourceDefinition
     */
    ResourceDefinition resourceDefinition = getResourceDefinition(type, mapIds);
    
    // 使用AlertTargetResourceDefinition构建QueryImpl并返回，mapIds为null
    return new QueryImpl(mapIds, resourceDefinition, ClusterControllerHelper.getClusterController());
  }
```

再看下BaseService的`handleRequest`方法, 通过这个方法可以让所有的请求处理都通过一个公用逻辑。它会创建一个request实例，然后触发它的process方法。
```java

protected Response handleRequest(HttpHeaders headers, String body,
                                   UriInfo uriInfo, Request.Type requestType,
                                   MediaType mediaType, ResourceInstance resource) {
    // 结果
    Result result = new ResultImpl(new ResultStatus(ResultStatus.STATUS.OK));
    try {
      // 解析请求内容，使用JsonRequestBodyParser执行解析，其实就是解析成一系列kv
      Set<RequestBody> requestBodySet = getBodyParser().parse(body);

      Iterator<RequestBody> iterator = requestBodySet.iterator();
      while (iterator.hasNext() && result.getStatus().getStatus().equals(ResultStatus.STATUS.OK)) {
        RequestBody requestBody = iterator.next();
        
        // 使用RequestFactory创建request，我们会调用createPostRequest方法
        Request request = getRequestFactory().createRequest(
            headers, requestBody, uriInfo, requestType, resource);
        // 执行request
        result  = request.process();
      }
    } 

   ....
   ....
   
    return builder.build();
  }

```


创建request的代码部分：

```java
private Request createPostRequest(HttpHeaders headers, RequestBody body, UriInfo uriInfo, ResourceInstance resource) {
    boolean batchCreate = !applyDirectives(Request.Type.POST, body, uriInfo, resource);;
    
    // 如果是批处理，就返回QueryPostRequest, 否则返回PostRequest，我们这里返回的是PostRequest
    // PostRequest会与一个CreateHandler一一对应，作为它的handler存在
    return (batchCreate) ?
        new QueryPostRequest(headers, body, uriInfo, resource) :
        new PostRequest(headers, body, uriInfo, resource);
  }
```

request的process部分，那么就看PostRequest的实现，发现它继承了BaseRequest，实际调用的是BaseRequest的process方法：
```java
Result result;
try {
  parseRenderer();
  parseQueryPredicate();
  result = getRequestHandler().handleRequest(this);
} catch (InvalidQueryException e) {
  result =  new ResultImpl(new ResultStatus(ResultStatus.STATUS.BAD_REQUEST,
      "Unable to compile query predicate: " + e.getMessage()));
} catch (IllegalArgumentException e) {
  result =  new ResultImpl(new ResultStatus(ResultStatus.STATUS.BAD_REQUEST,
      "Invalid Request: " + e.getMessage()));
}

if (! result.getStatus().isErrorState()) {
  getResultPostProcessor().process(result);
}
```

BaseManagementHandler中又调用了persist方法
```java
  @Override
  public Result handleRequest(Request request) {
    Query query = request.getResource().getQuery();
    Predicate queryPredicate = request.getQueryPredicate();

    query.setRenderer(request.getRenderer());
    if (queryPredicate != null) {
      query.setUserPredicate(queryPredicate);
    }
    return persist(request.getResource(), request.getBody());
  }

```

`persist(ResourceInstance resource, RequestBody body)`方法在CreateHandler中有实现, 代码中的PersistenceManager只有一个实现PersistenceManagerImpl。
```
RequestStatus status = getPersistenceManager().create(resource, body);
result = createResult(status);
```

PersistenceManagerImpl的`create(ResourceInstance resource, RequestBody requestBody)`相关部分。
```
// 获取resource对应的主键、外键
Map<Resource.Type, String> mapResourceIds = resource.getKeyValueMap();
Resource.Type type = resource.getResourceDefinition().getType();
Schema schema = m_controller.getSchema(type);

// 获取body中解析的各种键值
Set<NamedPropertySet> setProperties = requestBody.getNamedPropertySets();
if (setProperties.isEmpty()) {
requestBody.addPropertySet(new NamedPropertySet("", new HashMap<String, Object>()));
}

// 遍历body中的键值
for (NamedPropertySet propertySet : setProperties) {
for (Map.Entry<Resource.Type, String> entry : mapResourceIds.entrySet()) {
  Map<String, Object> mapProperties = propertySet.getProperties();
  // schema中组合了ResourceProvider，这里就是AlertTargetResourceProvider，getKeyPropertyId方法是调用AlertTargetResourceProvider获取可以唯一确认一条记录的唯一键
  // AlertTarget中定义的唯一键是"AlertTarget/id"和"AlertTarget/name"
  String property = schema.getKeyPropertyId(entry.getKey());
  // 如果不包含唯一键，就添加进入mapProperties
  if (!mapProperties.containsKey(property)) {
    mapProperties.put(property, entry.getValue());
  }
}
}
// 创建resource
return m_controller.createResources(type, createControllerRequest(requestBody));
```

最后一步就是调用了`ClusterControllerImpl`的`createResources(Type type, Request request)`方法
```java
public RequestStatus createResources(Type type, Request request){
    // 获取对应的ResrouceProvider，如果找不到就用DefaultProviderModule通过type去createResourceProvider对应的ResrouceProvider，我们这里对应的是上面提到的AlertTargetResourceProvider
    ResourceProvider provider = ensureResourceProvider(type);
    if (provider != null) {
    
      checkProperties(type, request, null);
    
      return provider.createResources(request);
    }
    return null;
}
```

最后一步AlertTargetResourceProvider, 然后会调用到AlertDispatchDAO的`create(AlertTargetEntity alertTarget)`方法，就入库了。
```
  public RequestStatus createResources(final Request request)
      throws SystemException,
      UnsupportedPropertyException, ResourceAlreadyExistsException,
      NoSuchParentResourceException {

    createResources(new Command<Void>() {
      @Override
      public Void invoke() throws AmbariException {
        createAlertTargets(request.getProperties(), request.getRequestInfoProperties());
        return null;
      }
    });

    notifyCreate(Resource.Type.AlertTarget, request);
    return getRequestStatus(null);
  }
```

细看一下，是个事务接口
```
  public void create(AlertTargetEntity alertTarget) {
    // 持久化target到数据库
    entityManagerProvider.get().persist(alertTarget);
    
    // 如果这个target是global的，就遍历所有的group，与这个target关联起来
    // 就是更新数据库中的关联关系
    if (alertTarget.isGlobal()) {
      List<AlertGroupEntity> groups = findAllGroups();
      for (AlertGroupEntity group : groups) {
        group.addAlertTarget(alertTarget);
        merge(group);
      }
    }
  }
```



感想
---

没有看到什么特别的好处，整个处理过程特别绕...如果只是为了统一处理请求的话，使用spring的filter之类的不能搞定么？为什么要弄一个handleRequest的抽象方法呢？


