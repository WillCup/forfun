
title: azkaban源码追踪(1)-——-上传zip文件
date: 2016-12-05 17:09:26
tags: [youdaonote]
---

#### 相关类
 - ProjectMangerServlet
 - ProjectManager
 - ValidatorManager
 - DirectoryFlowLoader
 - JdbcProjectLoader

#### 对应的流程log
```log
2016/06/14 15:39:18.537 +0800 INFO [ProjectManagerServlet] [Azkaban] Uploading file willjob.zip
2016/06/14 15:39:23.861 +0800 INFO [ProjectManager] [Azkaban] Uploading files to mytestProject
2016/06/14 15:40:16.454 +0800 WARN [XmlValidatorManager] [Azkaban] Validator directory validators does not exist or is not a directory
2016/06/14 15:40:16.455 +0800 WARN [XmlValidatorManager] [Azkaban] Azkaban properties file does not contain the key project.validators
2016/06/14 15:40:35.752 +0800 INFO [ProjectManager] [Azkaban] Validating project willjob.zip using the registered validators [Director
2016/06/14 16:11:45.952 +0800 INFO [XmlValidatorManager] [Azkaban] Validation status of validator Directory Flow is PASS
2016/06/14 16:56:51.338 +0800 INFO [ProjectManager] [Azkaban] Uploading file to db willjob.zip
2016/06/14 16:56:51.338 +0800 INFO [JdbcProjectLoader] [Azkaban] Uploading to mytestProject version:1 file:willjob.zip
2016/06/14 16:56:51.341 +0800 INFO [JdbcProjectLoader] [Azkaban] Creating message digest for upload willjob.zip
2016/06/14 16:56:51.343 +0800 INFO [JdbcProjectLoader] [Azkaban] Md5 hash created
2016/06/14 16:56:51.359 +0800 INFO [JdbcProjectLoader] [Azkaban] Read bytes for willjob.zip size:560
2016/06/14 16:56:51.359 +0800 INFO [JdbcProjectLoader] [Azkaban] Running update for willjob.zip chunk 0
2016/06/14 16:56:51.360 +0800 INFO [JdbcProjectLoader] [Azkaban] Finished update for willjob.zip chunk 0
2016/06/14 16:56:51.362 +0800 INFO [JdbcProjectLoader] [Azkaban] Commiting upload willjob.zip
2016/06/14 16:56:51.363 +0800 INFO [ProjectManager] [Azkaban] Uploading flow to db willjob.zip
2016/06/14 16:56:51.363 +0800 INFO [JdbcProjectLoader] [Azkaban] Uploading flows
2016/06/14 16:56:51.418 +0800 INFO [JdbcProjectLoader] [Azkaban] Flow upload will3 is byte size 214
2016/06/14 16:56:51.432 +0800 INFO [JdbcProjectLoader] [Azkaban] Flow upload will2 is byte size 238
2016/06/14 16:56:51.434 +0800 INFO [ProjectManager] [Azkaban] Changing project versions willjob.zip
2016/06/14 16:56:51.436 +0800 INFO [ProjectManager] [Azkaban] Uploading Job properties
2016/06/14 16:56:51.820 +0800 INFO [ProjectManager] [Azkaban] Uploading Props properties
2016/06/14 16:56:51.822 +0800 INFO [ProjectManager] [Azkaban] Uploaded project files. Cleaning up temp files.
2016/06/14 16:56:51.927 +0800 INFO [ProjectManager] [Azkaban] Cleaning up old install files older than -2

```
#### ProjectManager
白话一下ProjectManager处理上传zip文件的过程：
1. 将文件存到/tmp目录并解压
2. 使用DirectoryFlowLoader解析文件，放置到这个loader的状态变量中。这些变量会一直保存在内存里，以供后面使用
3. 遍历loader里的flowMap，把每个flow都加上最新的project_id和version
4. 上传project信息
    - 4.1 把zip文件弄成字节流上传到mysql的project_files表中
    - 4.2 把最新版本的project信息上传到project_versions表中
5. 上传flow: 把flow逐个序列化成json，再弄成byte字节流，再使用gzip压缩，上传至project_flows表中
6. 更新projects里的相关记录，并修改内存中的project信息【修改时间等、最新版本的flows】
7. 上传job的详细信息到project_properties中，job的详细信息也是json->byte[]->gzip的过程
8. 上传props的详细信息到project_properties中，流程同job完全一样【这个debug的时候数据是空的】
9. 将此次操作插入到project_event中，之后删除本地/tmp下的文件
10. 使用project_id和当前version判断依次清除先前版本的project_flows的flow、project_properties、project_files、project_versions中的记录【会有最近版本保留数量设置project.version.retention，默认是3】
11. 完成

#### 文件解析
DirectoryFlowLoader加载文件目录，解析所有的文件，进行解析，将信息存放在自己的状态变量中：
```java
  private Props props;                          // 环境、配置信息
  private HashSet<String> rootNodes;            // 根节点，也就是实际执行的时候的起始节点
  private HashMap<String, Flow> flowMap;        // 所有的flow，key是flowId
  private HashMap<String, Node> nodeMap;        // 所有的node，key是JobId或者embed的flowId
  // node之间的dependency，key是job名称，value是一系列的相关依赖job，一般是children，然后Edge指定从谁到谁
  private HashMap<String, Map<String, Edge>> nodeDependencies;  
  private HashMap<String, Props> jobPropsMap;   // 包含job的详细信息

  // flow之间的依赖关系【embedeed flow】
  private HashMap<String, Set<String>> flowDependencies;

  private ArrayList<FlowProps> flowPropsList;
  private ArrayList<Props> propsList;
  private Set<String> errors;
  private Set<String> duplicateJobs;
```

```java
// 临时解压目录 baseDirectory : temp/46769225
// 项目信息 project: testPro
public void loadProjectFlow(Project project, File baseDirectory) {
    propsList = new ArrayList<Props>();
    flowPropsList = new ArrayList<FlowProps>();
    jobPropsMap = new HashMap<String, Props>();
    nodeMap = new HashMap<String, Node>();
    flowMap = new HashMap<String, Flow>();
    errors = new HashSet<String>();
    duplicateJobs = new HashSet<String>();
    nodeDependencies = new HashMap<String, Map<String, Edge>>();
    rootNodes = new HashSet<String>();
    flowDependencies = new HashMap<String, Set<String>>();

    // 加载所有的properties、job文件，创建node对象
    loadProjectFromDir(baseDirectory.getPath(), baseDirectory, null);

    jobPropertiesCheck(project);

    // Create edges and find missing dependencies
    resolveDependencies();

    // 创建flows. 在此之后flowMap里就已经构建好了flow以及job之间的依赖了，node之间通过level来界定先后关系
    buildFlowsFromDependencies();

    // 处理embedded flows，就是嵌入的flow
    resolveEmbeddedFlows();

  }

```
##### loadProjectFromDir方法详细
```java
  private void loadProjectFromDir(String base, File dir, Props parent) {
    // 找出所有的properties
    File[] propertyFiles = dir.listFiles(new SuffixFilter(PROPERTY_SUFFIX));
    Arrays.sort(propertyFiles);

    for (File file : propertyFiles) {
        String relative = getRelativeFilePath(base, file.getPath());
        parent = new Props(parent, file);
        parent.setSource(relative);
        FlowProps flowProps = new FlowProps(parent);
        flowPropsList.add(flowProps);
        logger.info("Adding " + relative);
        propsList.add(parent);
    }

    // 加载job文件
    File[] jobFiles = dir.listFiles(new SuffixFilter(JOB_SUFFIX));
    for (File file : jobFiles) {
      String jobName = getNameWithoutExtension(file);
        if (!duplicateJobs.contains(jobName)) {
          if (jobPropsMap.containsKey(jobName)) {
            // 是否有重复
            ... 
          } else {
            Props prop = new Props(parent, file);
            String relative = getRelativeFilePath(base, file.getPath());
            prop.setSource(relative);

            Node node = new Node(jobName);
            String type = prop.getString("type", null);
            if (type == null) {
              errors.add("Job doesn't have type set '" + jobName + "'.");
            }
            node.setType(type); // job类型
            node.setJobSource(relative);    // job相对路径
            if (parent != null) {
              node.setPropsSource(parent.getSource());
            }

            // Force root node
            if (prop.getBoolean(CommonJobProperties.ROOT_NODE, false)) {
              rootNodes.add(jobName);
            }

            // 把properties和node放进loader中的队列中
            jobPropsMap.put(jobName, prop);
            nodeMap.put(jobName, node);
          }
        }
    }

    // 检查是否有子目录
    File[] subDirs = dir.listFiles(DIR_FILTER);
    for (File file : subDirs) {
      loadProjectFromDir(base, file, parent);
    }
  }
```

#### 常用的blob处理代码
```java
 String propertyJSON = PropsUtils.toJSONString(props, true);
    byte[] data = propertyJSON.getBytes("UTF-8");
    if (defaultEncodingType == EncodingType.GZIP) {
      data = GZIPUtils.gzipBytes(data);
    }
```


#### 其他

![Node结构](C:\Users\will\Pictures\azkaban上传project时的node结构.png)

level表示当前node所在的层次。


![Edge结构](C:\Users\will\Pictures\Edge数据结构.png)
