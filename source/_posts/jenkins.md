
title: jenkins
date: 2016-12-05 17:09:07
tags: [youdaonote]
---

 
 安装插件
 ---
 `问题`：每次在线安装插件的时候，Jenkins默认都是使用www.google.com去验证网络是否通畅，如果不通畅下面指定的插件还是会去下载，但是根本就不会安装。(比较让人费解的是，都下了还不安，要么就别下了也)
 
 `期望`：使用www.baidu.com来验证网络，或者跳过验证。
 
 `方案`: 修改/var/lib/jenkins/updates/default.json里的connectionCheckUrl，这个文件特别大， 但是这个connectionCheckUrl一般都在最开始的地方。
 
启用用户验证
---
系统管理-> Configure Global Security

maven相关
---
安装maven插件

#### maven仓库
主要是为了能够让每个maven项目build的时候都使用一个公共的maven地址，而又不想去默认的~/.m2/repository(我们这边/的硬盘不是很大).

安装一个插件：config file provider plugin
在系统管理->Managed files里添加一个maven的settings文件，可以把自己线上的maven配置信息弄过来。

#### maven构建

maven项目配置的时候，记得以下几点
 - 在goals和options里指定maven构建目标，反正我是二逼的忘掉了。这一条只为二逼的自己
 - root pom指定相对根目录的pom.xml的位置。
 - 修改config file provider plugin的settings.xml指定的localRepository对于Jenkins用户的可读可写可执行的权限，懒货如我，直接整个repo都777了。


#### 发布到tomcat
`期望`：
build成功后直接使用shell将war包发布到tomcat，并重启之，等待一段时间后，测试一系列接口，确认后台正常服务。

`问题`：
jenkins的job进程创建的、包括调用脚本创建的进程都会随着Jenkins job进程的退出而退出。

参考：[ProcessTreeKiller](https://wiki.jenkins-ci.org/display/JENKINS/ProcessTreeKiller)

`方案`：
下载Deploy to container Plugin，在指定project下配置构建完成后操作，配置好tomcat的相关信息之后，尝试build。

发现报错:
```
Deploying /var/lib/jenkins/jobs/metamap/workspace/metamap_java/target/metamap.war to container Tomcat 7.x Remote
ERROR: Build step failed with exception
org.codehaus.cargo.container.ContainerException: Failed to redeploy [/var/lib/jenkins/jobs/metamap/workspace/metamap_java/target/metamap.war]
	at org.codehaus.cargo.container.tomcat.internal.AbstractTomcatManagerDeployer.redeploy(AbstractTomcatManagerDeployer.java:189)
	at hudson.plugins.deploy.CargoContainerAdapter.deploy(CargoContainerAdapter.java:73)
	at hudson.plugins.deploy.CargoContainerAdapter$1.invoke(CargoContainerAdapter.java:116)
	at hudson.plugins.deploy.CargoContainerAdapter$1.invoke(CargoContainerAdapter.java:103)
	at hudson.FilePath.act(FilePath.java:990)
	at hudson.FilePath.act(FilePath.java:968)
	at hudson.plugins.deploy.CargoContainerAdapter.redeploy(CargoContainerAdapter.java:103)
	at hudson.plugins.deploy.DeployPublisher.perform(DeployPublisher.java:61)
	at hudson.tasks.BuildStepMonitor$3.perform(BuildStepMonitor.java:45)
	at hudson.model.AbstractBuild$AbstractBuildExecution.perform(AbstractBuild.java:782)
	at hudson.model.AbstractBuild$AbstractBuildExecution.performAllBuildSteps(AbstractBuild.java:723)
	at hudson.maven.MavenModuleSetBuild$MavenModuleSetBuildExecution.post2(MavenModuleSetBuild.java:1037)
	at hudson.model.AbstractBuild$AbstractBuildExecution.post(AbstractBuild.java:668)
	at hudson.model.Run.execute(Run.java:1763)
	at hudson.maven.MavenModuleSetBuild.run(MavenModuleSetBuild.java:529)
	at hudson.model.ResourceController.execute(ResourceController.java:98)
	at hudson.model.Executor.run(Executor.java:410)
Caused by: org.codehaus.cargo.container.tomcat.internal.TomcatManagerException: The username you provided is not allowed to use the text-based Tomcat Manager (error 403)
	at org.codehaus.cargo.container.tomcat.internal.TomcatManager.invoke(TomcatManager.java:555)
	at org.codehaus.cargo.container.tomcat.internal.TomcatManager.list(TomcatManager.java:686)
	at org.codehaus.cargo.container.tomcat.internal.TomcatManager.getStatus(TomcatManager.java:699)
	at org.codehaus.cargo.container.tomcat.internal.AbstractTomcatManagerDeployer.redeploy(AbstractTomcatManagerDeployer.java:174)
	... 16 more
Caused by: java.io.IOException: Server returned HTTP response code: 403 for URL: http://10.0.1.62:8080//manager/text/list
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream0(HttpURLConnection.java:1839)
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1440)
	at org.codehaus.cargo.container.tomcat.internal.TomcatManager.invoke(TomcatManager.java:544)
	... 19 more
org.codehaus.cargo.container.tomcat.internal.TomcatManagerException: The username you provided is not allowed to use the text-based Tomcat Manager (error 403)
	at org.codehaus.cargo.container.tomcat.internal.TomcatManager.invoke(TomcatManager.java:555)
	at org.codehaus.cargo.container.tomcat.internal.TomcatManager.list(TomcatManager.java:686)
	at org.codehaus.cargo.container.tomcat.internal.TomcatManager.getStatus(TomcatManager.java:699)
	at org.codehaus.cargo.container.tomcat.internal.AbstractTomcatManagerDeployer.redeploy(AbstractTomcatManagerDeployer.java:174)
	at hudson.plugins.deploy.CargoContainerAdapter.deploy(CargoContainerAdapter.java:73)
	at hudson.plugins.deploy.CargoContainerAdapter$1.invoke(CargoContainerAdapter.java:116)
	at hudson.plugins.deploy.CargoContainerAdapter$1.invoke(CargoContainerAdapter.java:103)
	at hudson.FilePath.act(FilePath.java:990)
	at hudson.FilePath.act(FilePath.java:968)
	at hudson.plugins.deploy.CargoContainerAdapter.redeploy(CargoContainerAdapter.java:103)
	at hudson.plugins.deploy.DeployPublisher.perform(DeployPublisher.java:61)
	at hudson.tasks.BuildStepMonitor$3.perform(BuildStepMonitor.java:45)
	at hudson.model.AbstractBuild$AbstractBuildExecution.perform(AbstractBuild.java:782)
	at hudson.model.AbstractBuild$AbstractBuildExecution.performAllBuildSteps(AbstractBuild.java:723)
	at hudson.maven.MavenModuleSetBuild$MavenModuleSetBuildExecution.post2(MavenModuleSetBuild.java:1037)
	at hudson.model.AbstractBuild$AbstractBuildExecution.post(AbstractBuild.java:668)
	at hudson.model.Run.execute(Run.java:1763)
	at hudson.maven.MavenModuleSetBuild.run(MavenModuleSetBuild.java:529)
	at hudson.model.ResourceController.execute(ResourceController.java:98)
	at hudson.model.Executor.run(Executor.java:410)
Caused by: java.io.IOException: Server returned HTTP response code: 403 for URL: http://10.0.1.62:8080//manager/text/list
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream0(HttpURLConnection.java:1839)
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1440)
	at org.codehaus.cargo.container.tomcat.internal.TomcatManager.invoke(TomcatManager.java:544)
	... 19 more
Build step 'Deploy war/ear to a container' marked build as failure
```

修改对应tomcat的conf/tomcat-users.xml，添加tomcat-script角色，并赋予给当前的tomcat管理员用户。

```
build pom done.....
Deploying /var/lib/jenkins/jobs/metamap/workspace/metamap_java/target/metamap.war to container Tomcat 7.x Remote
  Redeploying [/var/lib/jenkins/jobs/metamap/workspace/metamap_java/target/metamap.war]
  Undeploying [/var/lib/jenkins/jobs/metamap/workspace/metamap_java/target/metamap.war]
  Deploying [/var/lib/jenkins/jobs/metamap/workspace/metamap_java/target/metamap.war]
Finished: SUCCESS
```


使用root用户
---
因为会部署很多任务放在jenkins上，而各个任务都有自己的权限目录，使用jenkins用户执行任务会有些不方便，总是遇到权限问题。

结合我们既有环境考虑，使用root用户执行jenkins任务并不会造成额外的危险。（操作jenksin 的人很有限，而且各自有账户）

具体步骤:
```
vim /etc/sysconfig/jenkins
$JENKINS_USER="root"

chown -R root:root /var/lib/jenkins
chown -R root:root /var/cache/jenkins
chown -R root:root /var/log/jenkins

service jenkins restart
```
先在jenkins配置文件里指定jenkins系统执行任务的用户，之后修改jenkins既有目录的权限，重启服务。




问题
---

python脚本不及时输出，最后一股子出来。
在执行python命令前，先添加环境变量：
```
export PYTHONUNBUFFERED=1
```
- https://stackoverflow.com/questions/11631951/jenkins-console-output-not-in-realtime
