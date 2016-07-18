title: azkaban源码追踪（三）-上传zip的坑
date: 2016-06-16 19:16:06
tags: [azkaban, 源码]
---
前面其实已经解析了上传zip文件到azkaban的流程，但是自己都感觉写的实在是太操蛋了。

### 1. 时序图
![上传zip文件的时序图](/imgs/azkaban/azkaban-atash-sequence-diagram.png)

### 2. 坑的表现

ETL的依赖关系肯定的多父多子的，所以就上传了一个典型到azkaban上。结果被分成了两个flow


![拥有相关ETL的flow被分成了两个](/imgs/azkaban/azkaban-two-wrong-flow.png)

![第一个错误的flow](/imgs/azkaban/azkaban-wrong-flow-1.png)

![第二个错误的flow](/imgs/azkaban/azkaban-wrong-flow-2.png)

可以看到对于mymeta@my_table_c之上的etl，对于下面的node应该是公用的，像现在这样两个flow都有的话，就会导致这些node被多次执行。

### 3.源码分析
根据上面的时序图，我们可以看到问题应该是出在分析依赖或者创建flow的过程中。所以过去剖析对应源码
```java
// 就是把每个node的dependency map放进HashMap<String, Map<String, Edge>> nodeDependencies里面，一个对应多个dependencies
private void resolveDependencies() {
    // Add all the in edges and out edges. Catch bad dependencies and self
    // referrals. Also collect list of nodes who are parents.
    for (Node node : nodeMap.values()) {
      Props props = jobPropsMap.get(node.getId());

      List<String> dependencyList =
          props.getStringList(CommonJobProperties.DEPENDENCIES,
              (List<String>) null);

      if (dependencyList != null) {
        Map<String, Edge> dependencies = nodeDependencies.get(node.getId());
        if (dependencies == null) {
          dependencies = new HashMap<String, Edge>();

          for (String dependencyName : dependencyList) {
            dependencyName =
                dependencyName == null ? null : dependencyName.trim();
            if (dependencyName == null || dependencyName.isEmpty()) {
              continue;
            }

            Edge edge = new Edge(dependencyName, node.getId());
            Node dependencyNode = nodeMap.get(dependencyName);
            if (dependencyNode == null) {
              if (duplicateJobs.contains(dependencyName)) {
                edge.setError("Ambiguous Dependency. Duplicates found.");
                dependencies.put(dependencyName, edge);
                errors.add(node.getId() + " has ambiguous dependency "
                    + dependencyName);
              } else {
                edge.setError("Dependency not found.");
                dependencies.put(dependencyName, edge);
                errors.add(node.getId() + " cannot find dependency "
                    + dependencyName);
              }
            } else if (dependencyNode == node) {
              // We have a self cycle
              edge.setError("Self cycle found.");
              dependencies.put(dependencyName, edge);
              errors.add(node.getId() + " has a self cycle");
            } else {
              dependencies.put(dependencyName, edge);
            }
          }

          if (!dependencies.isEmpty()) {
            nodeDependencies.put(node.getId(), dependencies);
          }
        }
      }
    }
  }
```

跟踪源码的时候，没有发现上面代码有任何问题。再看下根据依赖构建flow的过程
```java
  private void buildFlowsFromDependencies() {
    // Find all root nodes by finding ones without dependents.
    /********************注意这里 ***********************/
    HashSet<String> nonRootNodes = new HashSet<String>();
    for (Map<String, Edge> edges : nodeDependencies.values()) {
      for (String sourceId : edges.keySet()) {
        nonRootNodes.add(sourceId);
      }
    }

    // Now create flows. Bad flows are marked invalid
    Set<String> visitedNodes = new HashSet<String>();
    for (Node base : nodeMap.values()) {
      // Root nodes can be discovered when parsing jobs
      /********************注意这里 ***********************/
      if (rootNodes.contains(base.getId())
          || !nonRootNodes.contains(base.getId())) {
        rootNodes.add(base.getId());
        Flow flow = new Flow(base.getId());
        Props jobProp = jobPropsMap.get(base.getId());

        // 遍历properties里的各种通知邮件啥的

        flow.addFailureEmails(failureEmail);
        flow.addSuccessEmails(successEmail);

        flow.addAllFlowProperties(flowPropsList);
        constructFlow(flow, base, visitedNodes);
        flow.initialize();
        flowMap.put(base.getId(), flow);
      }
    }
  }

```
上面第一个地方是找到所有非root的node，看代码意思就是所有有dependency的node。第二个注意的地方是找rootNodes里有或者不存在与非rootNode集合的node。这不是叶子节点么，下面紧接着就add进去rootNodes了。也就是说azkaban中这里的node的**root角色其实是指的叶子节点**。第三个是flow是按照叶子节点的id命名的，也就是有多少叶子节点就会有多少flow被生成。
解释通了，因为我们的project的zip文件中是有两个叶子节点的job，所以就被分成了两个flow。

### 4. 解决方案
生成最终的zip文件之前，生成一个虚拟end job，把所有的叶子节点都配置成它的dependency，这样就只有一个flow了。

![最终生成的正确的flow图](/imgs/azkaban/azkaban-right-flow.png)

![尝试执行以下正确的这个flow](/imgs/azkaban/azkaban-right-flow-execution.png)

可以看到，并列在同一个level的node是并发执行的，因为他们两个之间相互没有任何影响。这样使用并发充分利用资源，但是azkaban的并发性能和稳定性需要进一步探查。

### 5.TODO
每个任务的Runner稳定性测试与定制开发(希望不需要)
