title: ambari-views中文翻译
date: 2016-06-20 15:26:42
tags: [ambari, 源码]
---

简述
-----

ambari views可以在ambari-web中提供插件定制化的可视化、管理、监控等特性。

当用户添加了一个第三方的插件到ambari的时候，我们可以添加一个view来支持这个插件的UI，也就是说可以发布一个应用到ambari container里来。

每个view实例使用name和version来在ambari container里唯一标识自己。

Views in Ambari 是顶级的resource，并不直接跟cluster实例产生直接关联。

例子
-----
官网的例子在ambari-views/examples.

基础
-----
### View Instances
一个view可以有多个实例，每个实例可能有各自不同的属性。

### View Context
view context让view componenets访问ambari container提供的运行时context。这个context会一直伴随这view实例，从出生到死亡。

一个view context可能会被container注入到多个不同的view componenet里去，比如resources, resource provider, servlets等。

context接口提供给view instance以下信息:
```java

      /**
       * Get the current user name.
       *
       * @return the current user name
       */
      public String getUsername();
    
      /**
       * Determine whether or not the access specified by the given permission name
       * is permitted for the given user.
       *
       * @param userName        the user name
       * @param permissionName  the permission name
       *
       * @throws SecurityException if the access specified by the given permission name
       *         is not permitted
       */
      public void hasPermission(String userName, String permissionName) throws SecurityException;
    
      /**
       * Get the view name.
       *
       * @return the view name
       */
      public String getViewName();
    
      /**
       * Get the view definition associated with this context.
       *
       * @return the view definition
       */
      public ViewDefinition getViewDefinition();
    
      /**
       * Get the view instance name.
       *
       * @return the view instance name; null if no instance is associated
       */
      public String getInstanceName();
    
      /**
       * Get the view instance definition associated with this context.
       *
       * @return the view instance definition; null if no instance is associated
       */
      public ViewInstanceDefinition getViewInstanceDefinition();
    
      /**
       * Get the property values specified to create the view instance.
       *
       * @return the view instance property values; null if no instance is associated
       */
      public Map<String, String> getProperties();
    
      /**
       * Save an instance data value for the given key.
       *
       * @param key    the key
       * @param value  the value
       *
       * @throws IllegalStateException if no instance is associated
       */
      public void putInstanceData(String key, String value);
    
      /**
       * Get the instance data value for the given key.
       *
       * @param key  the key
       *
       * @return the instance data value; null if no instance is associated
       */
      public String getInstanceData(String key);
    
      /**
       * Get the instance data values.
       *
       * @return the view instance property values; null if no instance is associated
       */
      public Map<String, String> getInstanceData();
    
      /**
       * Remove the instance data value for the given key.
       *
       * @param key  the key
       *
       * @throws IllegalStateException if no instance is associated
       */
      public void removeInstanceData(String key);
    
      /**
       * Get a property for the given key from the ambari configuration.
       *
       * @param key  the property key
       *
       * @return the property value; null indicates that the configuration contains no mapping for the key
       */
      public String getAmbariProperty(String key);
    
      /**
       * Get the view resource provider for the given resource type.
       *
       * @param type  the resource type
       *
       * @return the resource provider; null if no instance is associated
       */
      public ResourceProvider<?> getResourceProvider(String type);
    
      /**
       * Get a URL stream provider.
       *
       * @return a stream provider
       */
      public URLStreamProvider getURLStreamProvider();
    
      /**
       * Get a data store for view persistence entities.
       *
       * @return a data store; null if no instance is associated
       */
      public DataStore getDataStore();
    
      /**
       * Get all of the available view definitions.
       *
       * @return the view definitions
       */
      public Collection<ViewDefinition> getViewDefinitions();
    
      /**
       * Get all of the available view instance definitions.
       *
       * @return the view instance definitions
       */
      public Collection<ViewInstanceDefinition> getViewInstanceDefinitions();
    
      /**
       * Get a view controller associated with this context.
       *
       * @return the view controller
       */
      public ViewController getController();
    
      /**
       * Get the HTTP Impersonator.
       *
       * @return the HTTP Impersonator, which internally uses the App Cookie Manager
       */
      public HttpImpersonator getHttpImpersonator();
    
      /**
       * Get the default settings for the Impersonator.
       *
       * @return the Impersonator settings.
       */
      public ImpersonatorSetting getImpersonatorSetting();
```

### UI Components
view包里可能包含一些web应用的组件，类似html或者javascript啥的。


##### WEB-INF/web.xml
web.xml 是把view发布为一个web app的主要文件。  

例如：

```xml

      <servlet>
        <servlet-name>HelloServlet</servlet-name>
        <servlet-class>org.apache.ambari.view.hello.HelloServlet</servlet-class>
      </servlet>
      <servlet-mapping>
        <servlet-name>HelloServlet</servlet-name>
        <url-pattern>/ui</url-pattern>
      </servlet-mapping>
```


##### Servlets


Servlets contained in a view archive will be deployed as part of the view and mapped as described in the web.xml.

在servlet的init方法中，可以view context当做一个servlet context使用。

例如:
```java 

      private ViewContext viewContext;

      @Override
      public void init(ServletConfig config) throws ServletException {
        super.init(config);

        ServletContext context = config.getServletContext();
        viewContext = (ViewContext) context.getAttribute(ViewContext.CONTEXT_ATTRIBUTE);
      }
```

servlet可以使用view conetxt获取ambari container提供的一个信息。例如，在doGet()方法中，我们就可以使用view context instance访问当前的用户和view instance的属性。

```java
      protected void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        response.setContentType("text/html");
        response.setStatus(HttpServletResponse.SC_OK);

        PrintWriter writer = response.getWriter();

        Map<String, String> properties = viewContext.getProperties();

        String name = properties.get("name");
        if (name == null) {
          name = viewContext.getUsername();
        }
        writer.println("<h1>Hello " + name + "!</h1>");
      }
```

### Resources
Resources一般不会暴露出来。resource都定义在view.xml里面。每个resource都需要指定名称和service class.

每个resource endpoint应该也可以被ambari api访问. [API](#api).

##### Service Class

service类使用JAX-RS注解来定义resource的endpoint。注意需要注入ViewContext.

```java
      @Inject
      ViewContext context;

      /**
       * @see org.apache.ambari.view.filebrowser.DownloadService
       * @return service
       */
      @Path("/download")
      public DownloadService download() {
        return new DownloadService(context);
      }  
      
      /**
       * @see org.apache.ambari.view.filebrowser.FileOperationService
       * @return service
       */
      @Path("/fileops")
      public FileOperationService fileOps() {
        return new FileOperationService(context);
      }    
```


#### Managed Resources
managed resource是由ambari api框架管理的resource。除了name和service class之外，managed resource定义还需要plural name，resource class和provider class。 See [view.xml](#viewxml).
 
使用managed resource的好处是任何通过rest api访问的查询都可以使用ambari api框架的一些既有的特性，包括partial response, query predicates, pagination, sub-resource queries等。See [API](#API).

##### Service Class

service使用JAX-RS注解定义获取resource的action。

下面的例子展示了service class怎样将request代理给api框架处理。注意要注入ViewResourceHandler，我们使用它在api框架中传递control。
```java
      @Inject
      ViewResourceHandler resourceHandler;

      …
      @GET
      @Path("{scriptId}")
      @Produces(MediaType.APPLICATION_JSON)
      public Response getScript(@Context HttpHeaders headers, @Context UriInfo ui,
                                @PathParam("scriptId") String scriptId) {
        return resourceHandler.handleRequest(headers, ui, scriptId);
      }
      
```


##### Resource Class

resource class是一个包含属性和resource的JavaBean。

```java

    public class PigScript {
      private String id;
      private String title;
      private String pigScript;
      private String pythonScript;
      private String templetonArguments;
      private Date dateCreated;
      private String owner;
      … 
      public String getId() {
        return id;
      }

      public void setId(String id) {
        this.id = id;
      }
      …  
    }
  
```

##### Resource Provider Class

Resource provider class实现ResourceProvider接口. 给ambari api框架提供访问resource的方式。

```java
      public class ScriptResourceProvider implements ResourceProvider<PigScript> {
        @Inject
        ViewContext viewContext;


        @Override
        public FileResource getResource(String resourceId, Set<String> propertyIds) throws
            SystemException, NoSuchResourceException, UnsupportedPropertyException {

          Map<String, String> properties = viewContext.getProperties();

          String userScriptsPath = properties.get("dataworker.userScriptsPath");

          try {
            return getResource(resourceId, userScriptsPath, propertyIds);
          } catch (IOException e) {
            throw new SystemException("Can't get resource " + resourceId + ".", e);
          }
        }
        ... 
      }
```


### 权限
administrator可以在任何view实例上设置VIEW.USE权限。  See [API](#api).


### 定制权限
编辑 view.xml
```xml 
  
      <view>
        <name>RESTRICTED</name>
        <label>The Restricted View</label>
        <version>1.0.0</version>
        <resource>
          <name>restricted</name>
          <service-class>org.apache.ambari.view.restricted.RestrictedResource</service-class>
        </resource>
        <resource>
          <name>unrestricted</name>
          <service-class>org.apache.ambari.view.restricted.UnrestrictedResource</service-class>
        </resource>
        <permission>
          <name>RESTRICTED</name>
          <description>
            Access permission for a restricted view resource.
          </description>
        </permission>
        <instance>
          <name>INSTANCE_1</name>
        </instance>
      </view>
```

这个配置文件定义了一个叫做RESTRICTED的view，它有一个实例. 还定义了一个叫做RESTRICTED的权限，两个分别叫做 'unrestricted'和 'restricted'的resource.
administrator可以将RESTRICTED权限分配给任意的RESTRICTED view的实例。

view的实现代码使用了view context来验证我们定义的权限。
 
```java            
      @Inject
      ViewContext context;
      
      @GET
      @Produces({"text/html"})
      public Response getRestricted() throws IOException{
    
        String userName = context.getUsername();
    
        try {
          context.hasPermission(userName, "RESTRICTED");
        } catch (org.apache.ambari.view.SecurityException e) {
          return Response.status(401).build();
        }
    
        return Response.ok("<b>You have accessed a restricted resource.</b>").type("text/html").build();
      } 
```

### Impersonation

Views can utilize the viewContext to facilitate calls that require impersonating a user. For example, a service may expose an endpoint that accepts parameters like "doAs=johndoe" to perform some action on behalf of that user.
The HttpImpersonator Interface provides a contract for how to perform an HTTP GET request on a URL that supports some type of "doAs" parameter, and the username.
```java
      HttpImpersonator impersonator = viewContext.getHttpImpersonator();
      ImpersonatorSetting impersonatorSetting = viewContext.getImpersonatorSetting();
      String result = impersonator.requestURL(urlToRead, "GET", impersonatorSetting);
```
The ImpersonatorSetting class contains the variables that are added to the URL params. Its default constructor sets "doAs" as the default query parameter name, and the currently logged on user as its value; both of these can be changed with the overloaded constructors.

### 持久化

应用的数据并不像view实例的属性一样，他们或许包含任意字符串，view需要把这些数据持久化，而且还提供更新以及查询。


view context包含从应用数据地图里put或者set数据的方法。
```java
      /**
       * Save an instance data value for the given key.
       *
       * @param key    the key
       * @param value  the value
       */
      public void putInstanceData(String key, String value);
      
      /**
       * Get the instance data value for the given key.
       *
       * @param key  the key
       *
       * @return the instance data value
       */
      public String getInstanceData(String key);
      
      /**
       * Get the instance data values.
       *
       * @return the view instance property values
       */
      public Map<String, String> getInstanceData();
      
      /**
       * Remove the instance data value for the given key.
       *
       * @param key  the key
       */
```
##### DataStore
data store 允许view instance存储javabean到ambari数据库里。

view组件先通过view context获取一个datastore的实例。
```java
      /**
       * Get a data store for view persistence entities.
       *
       * @return a data store
       */
      public DataStore getDataStore();

The DataStore object exposes a simple interface for persisting view entities.  The basic CRUD operations are supported …  

      /**
       * Save the given entity to persistent storage.  The entity must be declared as an
       * <entity> in the <persistence> element of the view.xml.
       *
       * @param entity  the entity to be persisted.
       *
       * @throws PersistenceException thrown if the given entity can not be persisted
       */
      public void store(Object entity) throws PersistenceException;
    
      /**
       * Remove the given entity from persistent storage.
       *
       * @param entity  the entity to be removed.
       *
       * @throws PersistenceException thrown if the given entity can not be removed
       */
      public void remove(Object entity) throws PersistenceException;
    
      /**
       * Find the entity of the given class type that is uniquely identified by the
       * given primary key.
       *
       * @param clazz       the entity class
       * @param primaryKey  the primary key
       * @param <T>         the entity type
       *
       * @return the entity; null if the entity can't be found
       *
       * @throws PersistenceException thrown if an error occurs trying to find the entity
       */
      public <T> T find(Class<T> clazz, Object primaryKey) throws PersistenceException;
    
      /**
       * Find all the entities for the given where clause.  Specifying null for the where
       * clause should return all entities of the given class type.
       *
       * @param clazz        the entity class
       * @param whereClause  the where clause; may be null
       * @param <T>          the entity type
       *
       * @return all of the entities for the given where clause; empty collection if no
       *         entities can be found
       *
       * @throws PersistenceException
       */
      public <T> Collection<T> findAll(Class<T> clazz, String whereClause) throws PersistenceException;
```
每个entity被持久化之前，都应该已经在view.xml里定义过了。 See [view.xml](#viewxml).   

### Events



##### View生命周期Events
实现View接口之后，每个view都可以监控自己的生命周期。
```java
    public interface View {
    
      ...
      
      public void onDeploy(ViewDefinition definition);
    
      public void onCreate(ViewInstanceDefinition definition);
    
      public void onDestroy(ViewInstanceDefinition definition);
    }
```

View实现类应该在view.xml文件里声明注册
```xml     
    <view>
      <name>FILES</name>
      <label>Files</label>
      <version>0.1.0</version>
      ...
      <view-class>org.apache.ambari.view.filebrowser.ViewImpl</view-class>
    </view>

```

View的实现类在注册过之后，它的对应方法会在任何生命周期event发生的时候被触发。
 
Event | 描述 | 参数
---|---|---
Deploy    |The view has been successfully deployed. |The deployed view definition.
Create    |An instance of the view has been created. |The new view instance definition.
Destroy   |An instance of the view has been destroyed. |The destroyed view instance definition.



##### 自定义event

饿哦们也可以定义一些自定义事件。

##### Listeners

view需要写一个listener接口的实现，然后注册这个listener到view框架。
```java
    public interface Listener {
    
      ...
      
      public void notify(Event event);
    }

```
listener的notify方法会在view发射一个event的时候被触发。
```java
    public class JobsListener implements Listener {
      @Override
      public void notify(Event event) {
        if (event.getId().equals("NEW_JOB")) {
          ViewInstanceDefinition instance = event.getViewInstanceSubject();         
          ...
        }      
      }
    }
```

##### ViewController

ViewController用来注册listener，还有就是发射event。
```java
    public interface ViewController {

      ...
      
      public void fireEvent(String eventId, Map<String, String> eventProperties);
    
      public void registerListener(Listener listener, String viewName);
    
      public void unregisterListener(Listener listener, String viewName);    
    }
```

view组件可以访问通过注入的view context来时用view controller。
```java
    @Inject
    ViewContext context;

    ...
    ViewController controller = context.getController();
```    


view component可以注册监听其他view发射的event。
```java
    @Inject
    ViewContext context;

    ...
    Listener listener = new JobsListener();

    ...
    ViewController controller = context.getController();
    controller.registerListener(listener, "JOBS");
```

view component也可以发射自定义的view通知event.

```java
    @Inject
    ViewContext context;

    ...
    Map<String, String> jobEventProperties = new HashMap<String, String>();

    jobEventProperties.put("JobId", jobId);
    jobEventProperties.put("JobOwner", jobOwner);
    
    ...
    ViewController controller = context.getController();
    controller.fireEvent("NEW_JOB", jobEventProperties);
```


打包
-----

所有的view都应该打包，包含配置文件和不同的组件，比如resource和resource provider啥的，还有一些 html, JavaSript and servlets.

### 依赖
The view may include dependency classes and jars directly into the archive.  Classes should be included under **WEB-INF/classes**.  Jars should be included under **WEB-INF/lib**.

### view.xml

元素 | 说明
---|---
name    |The unique view name (required).
label   |The user facing name.
version |The view version (required).
description |The description of the view.
icon64 |The 64x64 icon to display for this view. If this property is not set, the 32x32 sized icon will be used.
icon |The 32x32 icon to display for this view. Suggested size is 32x32 and will be displayed as 8x8 and 16x16 as necessary. If this property is not set, a default view framework icon is used.
system |Indicates whether or not this is a system view.  Default is false. 
view-class |The View class to receive framework events. 
masker-class |The Masker class for masking view parameters. 
parameter|Defines a configuration parameter that is used to when creating a view instance. 
resource |Describe a resources exposed by the view.
permission   |Defines a custom permission for the view. 
persistence   |Describe a view entities for persistence. 
instance |Define an instances of a view.

##### 参数
元素 | 描述
---|---
name    |The parameter name.
description    |The parameter description.
required    |Determines whether or not the parameter is required.
masked    |Indicated this parameter value is to be "masked" in the Ambari Web UI (i.e. not shown in the clear). Omitting this element default to not-masked. Otherwise, if true, the parameter value will be "masked" in the Web UI.

##### resource
元素 | 描述
---|---
name    |The resource name (i.e file).
plural-name    |The parameter description (i.e. files). *
id-property    |The id field of the managed resource class. *
resource-class |The class of the JavaBean that contains the attributes of a managed resource. *
provider-class |The class of the managed resource provider. *
service-class  |The class of the JAX-RS annotated service resource.
sub-resource-name  |The sub-resource name.

```xml

    <resource>
        <name>files</name>
        <service-class>org.apache.ambari.view.filebrowser.FileBrowserService</service-class>
    </resource>

\* only required for a managed resource.  For example ...

    <resource>
      <name>script</name>
      <plural-name>scripts</plural-name>
      <id-property>id</id-property>
      <resource-class>org.apache.ambari.view.pig.resources.scripts.models.PigScript</resource-class>
      <provider-class>org.apache.ambari.view.pig.resources.scripts.ScriptResourceProvider</provider-class>
      <service-class>org.apache.ambari.view.pig.resources.scripts.ScriptService</service-class>
    </resource>
```

##### 权限
Element | Description
---|---
name | The permission name.
description | The permission description.

##### 持久化
Element | Description
---|---
entity | An entity that may be persisted through the DataStore.

##### entity
Element | Description
---|---
class | The class ot the JavaBean that contains the attributes of an entity.
id-property | The id field of the entity.

例子
```java

    <persistence>
      <entity>
        <class>org.apache.ambari.view.employee.EmployeeEntity</class>
        <id-property>id</id-property>
      </entity>
      <entity>
        <class>org.apache.ambari.view.employee.AddressEntity</class>
        <id-property>id</id-property>
      </entity>
      <entity>
        <class>org.apache.ambari.view.employee.ProjectEntity</class>
        <id-property>id</id-property>
      </entity>
    </persistence> 
```
##### instance
Element | Description
---|---
name | The unique instance name (required).
label | The display label of the view instance. If not set, the view definition label is used.
description | The description of the view instance. If not set, the view definition description is used.
icon64 | Overrides the view icon64 for this specific view instance.
icon | Overrides the view icon for this specific view instance.
visible | If true, for the view instance to show up in the users view instance list.  The default value is true.
property | Specifies configuration parameters values for the view instance.

##### property
Element | Description
---|---
key | The property key (must match view parameter name).
value | The property value.



完整的 view.xml例子
```xml

    <view>
        <name>FILES</name>
        <label>Files</label>
        <version>0.1.0</version>

        <parameter>
            <name>dataworker.defaultFs</name>
            <description>FileSystem URI</description>
            <required>true</required>
        </parameter>
        <parameter>
            <name>dataworker.username</name>
            <description>The username (defaults to ViewContext username)</description>
            <required>false</required>
        </parameter>
    
        <resource>
            <name>files</name>
            <service-class>org.apache.ambari.view.filebrowser.FileBrowserService</service-class>
        </resource>
        
        <instance>
            <name>FILES_1</name>
            <property>
                <key>dataworker.defaultFs</key>
                <value>hdfs://<server>:8020</value>
            </property>
        </instance>
    </view>
```

这个例子中配置了一个叫FILES的view，也是有一个实例FILES_1。可以看到这个view有一个required的参数**'dataworker.defaultFs'**以及一个optional参数**'dataworker.username'**.  还有一个访问资源的入口**'files'**. 


发布
-----
要发布一个view，只需要将view的包放到ambari-server的views文件夹下，默认是在

	/var/lib/ambari-server/resources/views


### 解压包
view被发布之后，会自动在views文件夹下解压包。默认位置是:

    /var/lib/ambari-server/resources/views/work/:viewName{:viewVersion}
    
例如，上面例子中的files view解压后目录就是：

    /var/lib/ambari-server/resources/views/work/FILES{0.1.0}

用户可以直接修改解压后的文件，在ambari下次启动的时候就可以生效了。

使用
-----
### API

##### Get Views
获取所有已经发布的view。注意view是顶级resource，不属于任何其他resource。

    GET http://<server>:8080/api/v1/views/

    {
      "href" : "http://<server>:8080/api/v1/views/",
      "items" : [
        {
          "href" : "http://<server>:8080/api/v1/views/FILES",
          "ViewInfo" : {
            "view_name" : "FILES"
          }
        },
        {
          "href" : "http://<server>:8080/api/v1/views/HELLO_WORLD",
          "ViewInfo" : {
            "view_name" : "HELLO_WORLD"
          }
        }
      ]
    }
 
##### Get View
根据名称获取指定view，返回所有版本.

    GET http://<server>:8080/api/v1/views/FILES/
    
    {
      "href" : "http://<server>:8080/api/v1/views/FILES/",
      "ViewInfo" : {
        "view_name" : "FILES"
      },
      "versions" : [
        {
          "href" : "http://<server>:8080/api/v1/views/FILES/versions/0.1.0",
          "ViewVersionInfo" : {
            "version" : "0.1.0",
            "view_name" : "FILES"
          }
        }
      ]
    }


##### Get Versions
获取一个view 的所有版本.

    GET http://<server>:8080/api/v1/views/FILES/versions/
    
    {
      "href" : "http://<server>:8080/api/v1/views/FILES/versions/",
      "items" : [
        {
          "href" : "http://<server>:8080/api/v1/views/FILES/versions/0.1.0",
          "ViewVersionInfo" : {
            "version" : "0.1.0",
            "view_name" : "FILES"
          }
        }
      ]
    }

##### Get Version
获取某view的指定版本以及此版本下的所有实例.

    GET http://<server>:8080/api/v1/views/FILES/versions/0.1.0/
    
    {
      "href" : "http://<server>:8080/api/v1/views/FILES/versions/0.1.0/",
      "ViewVersionInfo" : {
        "archive" : "/var/lib/ambari-server/resources/views/work/FILES{0.1.0}",
        "label" : "Files",
        "parameters" : [
          {
            "name" : "dataworker.defaultFs",
            "description" : "FileSystem URI",
            "required" : true
          },
          {
            "name" : "dataworker.username",
            "description" : "The username (defaults to ViewContext username)",
            "required" : false
          }
        ],
        "version" : "0.1.0",
        "view_name" : "FILES"
      },
      "instances" : [
        {
          "href" : "http://<server>:8080/api/v1/views/FILES/versions/0.1.0/instances/FILES_1",
          "ViewInstanceInfo" : {
            "instance_name" : "FILES_1",
            "version" : "0.1.0",
            "view_name" : "FILES"
          }
        }
      ]
    }

##### Get Instances
获取一个view 的所有实例.

    GET http://<server>:8080/api/v1/views/FILES/versions/0.1.0/instances/
    
    {
      "href" : "http://<server>:8080/api/v1/views/FILES/versions/0.1.0/instances/",
      "items" : [
        {
          "href" : "http://<server>:8080/api/v1/views/FILES/versions/0.1.0/instances/FILES_1",
          "ViewInstanceInfo" : {
            "instance_name" : "FILES_1",
            "version" : "0.1.0",
            "view_name" : "FILES"
          }
        }
      ]
    }


##### 获取实例
根据实例名称获取实例，此实例的所有resource也会被返回。

    GET http://<server>:8080/api/v1/views/FILES/versions/0.1.0/instances/FILES_1
    
    {
      "href" : "http://<server>:8080/api/v1/views/FILES/versions/0.1.0/instances/FILES_1",
      "ViewInstanceInfo" : {
        "context_path" : "/views/FILES/0.1.0/FILES_1",
        "instance_name" : "FILES_1",
        "version" : "0.1.0",
        "view_name" : "FILES",
        "instance_data" : { },
        "properties" : {
          "dataworker.defaultFs" : "hdfs://<server>:8020"
        }
      },
      "resources" : [
        {
          "href" : "http://<server>:8080/api/v1/views/FILES/versions/0.1.0/instances/FILES_1/resources/files",
          "instance_name" : "FILES_1",
          "name" : "files",
          "version" : "0.1.0",
          "view_name" : "FILES"
        }
      ]
    }
    
##### 创建实例

    POST http://<server>:8080/api/v1/views/FILES/versions/0.1.0/instances/FILES_2
    
##### 更新实例

    PUT http://<server>:8080/api/v1/views/FILES/versions/0.1.0/instances/FILES_2
    [{
      "ViewInstanceInfo" : {
          "properties" : {
            "dataworker.defaultFs" : "hdfs://MyServer:8020"
          }
        }
    }]

##### 删除实例

    DELETE http://<server>:8080/api/v1/views/FILES/versions/0.1.0/instances/FILES_2
    

##### Resources
A view resource may be accessed through the REST API.  The href for each view resource can be found in a request for the parent view instance.  The resource endpoints and behavior depends on the implementation of the JAX-RS annotated service class specified for the resource element in the view.xml. 

    GET http://<server>:8080/api/v1/views/FILES/versions/0.1.0/instances/FILES_1/resources/files/fileops/listdir?path=%2F
    
    [{"path":"/app-logs","replication":0,"isDirectory":true,"len":0,"owner":"yarn","group":"hadoop","permission":"-rwxrwxrwx","accessTime":0,"modificationTime":1400006792122,"blockSize":0},{"path":"/mapred","replication":0,"isDirectory":true,"len":0,"owner":"mapred","group":"hdfs","permission":"-rwxr-xr-x","accessTime":0,"modificationTime":1400006653817,"blockSize":0},{"path":"/mr-history","replication":0,"isDirectory":true,"len":0,"owner":"hdfs","group":"hdfs","permission":"-rwxr-xr-x","accessTime":0,"modificationTime":1400006653822,"blockSize":0},{"path":"/tmp","replication":0,"isDirectory":true,"len":0,"owner":"hdfs","group":"hdfs","permission":"-rwxrwxrwx","accessTime":0,"modificationTime":1400006720415,"blockSize":0},{"path":"/user","replication":0,"isDirectory":true,"len":0,"owner":"hdfs","group":"hdfs","permission":"-rwxr-xr-x","accessTime":0,"modificationTime":1400006610050,"blockSize":0}]

##### Managed Resources
Any managed resources for a view may also be accessed through the REST API.  Note that instances of a managed resource appear as sub-resources of an instance under the plural name specified for the resource in the view.xml.  In this example, a list of managed resources named **'scripts'** is included in the response for the view instance … 
 
    GET http://<server>:8080/api/v1/views/PIG/versions/0.1.0/instances/INSTANCE_1
    
    {
      "href" : "http://<server>:8080/api/v1/views/PIG/versions/0.1.0/instances/INSTANCE_1",
      "ViewInstanceInfo" : {
        "context_path" : "/views/PIG/0.1.0/INSTANCE_1",
        "instance_name" : "INSTANCE_1",
        "version" : "0.1.0",
        "view_name" : "PIG",
        "instance_data" : { },
        "properties" : {
          … 
        }
      },
      "resources" : [ ],
      "scripts" : [
        {
          "href" : "http://<server>:8080/api/v1/views/PIG/versions/0.1.0/instances/INSTANCE_1/scripts/script1",
          "id" : "script1",
          "instance_name" : "INSTANCE_1",
          "version" : "0.1.0",
          "view_name" : "PIG"
        },
        {
          "href" : "http://<server>:8080/api/v1/views/FILES/versions/0.1.0/instances/INSTANCE_1/scripts/script2",
          "id" : "script2",
          "instance_name" : "INSTANCE_1",
          "version" : "0.1.0",
          "view_name" : "PIG"
        },
        …
      ]
    }
    
 
获取单个managed resource …
    
    
    GET http://<server>:8080/api/v1/views/PIG/versions/0.1.0/instances/INSTANCE_1/scripts/script1
    
    
    {
      "href" : "http://<server>:8080/api/v1/views/PIG/versions/0.1.0/instances/INSTANCE_1/scripts/script1",
      "id" : "script1",      
      "pigScript" : "… ",
      "pythonScript" : "… ",
      "templetonArguments" : "… ",
      "dateCreated" : … ,
      "owner" : "… ",
      "instance_name" : "INSTANCE_1",
      "version" : "0.1.0",
      "view_name" : "PIG"
    }
    
由于resource是被ambari api框架管理的，也可以使用partial response和query predicate。
 
    GET http://<server>:8080/api/v1/views/PIG/versions/0.1.0/instances/INSTANCE_1/scripts?fields=pythonScript&owner=jsmith
    

##### 设置view权限

To grant privileges to access the restricted resource we can create a privilege sub-resource for the view instance.  The following API will grant RESTICTED permission to the user 'bob' for the view instance 'INSTANCE_1' of the 'RESTRICTED' view. 

    POST http://<server>/api/v1/views/RESTRICTED/versions/1.0.0/instances/INSTANCE_1
    
    [
      {
        "PrivilegeInfo" : {
          "permission_name" : "RESTRICTED",
          "principal_name" : "bob",
          "principal_type" : "USER"
        }
      }
    ]

##### 查询view权限

We can see a privilege sub resource for a view instance.

    GET  http://<server>/api/v1/views/RESTRICTED/versions/1.0.0/instances/INSTANCE_1/privileges/5


    {
      "href" : "http://<server>/api/v1/views/RESTRICTED/versions/1.0.0/instances/INSTANCE_1/privileges/5",
      "PrivilegeInfo" : {
        "instance_name" : "INSTANCE_1",
        "permission_name" : "RESTRICTED",
        "principal_name" : "bob",
        "principal_type" : "USER",
        "privilege_id" : 5,
        "version" : "1.0.0",
        "view_name" : "RESTRICTED"
      }
    }
 
### View UI
The context root of a view instance is …
    
    /views/:viewName/:viewVersion/:viewInstance/

For example, the context root can be found in the response for a view instance … 
    
![image](instance_response.png)

So, to access the UI of the above Files view the user would browse to …

    http://<server>:8080/views/FILES/0.1.0/FILES_1/  



![image](ui.png)
