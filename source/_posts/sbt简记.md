title: scalatra入门
date: 2017-11-09 16:08:49
tags: [scala, scalatra]
---

#### 生成scalatra项目

下载项目，并配置相关信息：
```
# 从github上下载项目
root@will-vm:/usr/local/will/learning# sbt new scalatra/scalatra.g8
[info] Set current project to learning (in build file:/usr/local/will/learning/)
# 用来发布项目的，一般就是反序域名[联想一下java包]
organization [com.example]: com.will
# scalatra app名字
name [My Scalatra Web App]: willup
# 项目版本，自己随便定义，比如0.0.1
version [0.1.0-SNAPSHOT]: 
# 我们的servlet类的名字
servlet_name [MyScalatraServlet]: Will
# 所有的scala文件都属于一个pacakge
package [com.example.app]: com.will.app
# 使用的scala版本
scala_version [2.12.3]: 2.12.1
# 使用的sbt版本
sbt_version [1.0.2]: 1.0.3
# 使用的scalatra版本
scalatra_version [2.5.4]: 
# 初始项目生成在willup目录了，有些像django
Template applied in ./willup
root@will-vm:/usr/local/will/learning# ll
总用量 16
drwxr-xr-x 4 root root 4096 11月  9 17:12 ./
drwxr-xr-x 8 root root 4096 11月  9 16:06 ../
drwxr-xr-x 3 root root 4096 11月  9 17:11 target/
drwxr-xr-x 4 root root 4096 11月  9 17:12 willup/
```

#### 构建

进入willup, 执行`sbt`命令，sbt就会自动帮我们自动下载Scalatra的开发环境了，这需要些时间。

```
root@will-vm:/usr/local/will/learning/willup# sbt
[info] Loading settings from plugins.sbt ...
[info] Loading project definition from /usr/local/will/learning/willup/project
[info] Updating {file:/usr/local/will/learning/willup/project/}willup-build...
[info] downloading https://repo1.maven.org/maven2/com/amazonaws/jmespath-java/1.11.105/jmespath-java-1.11.105.jar ...
[info] downloading https://repo.scala-sbt.org/scalasbt/sbt-plugin-releases/com.typesafe.sbt/sbt-twirl/scala_2.12/sbt_1.0/1.3.12/jars/sbt-twirl.jar ...
[info] 	[SUCCESSFUL ] com.amazonaws#jmespath-java;1.11.105!jmespath-java.jar (824ms)
[info] downloading https://repo1.maven.org/maven2/org/apache/httpcomponents/httpclient/4.5.2/httpclient-4.5.2.jar ...
[info] 	[SUCCESSFUL ] com.typesafe.sbt#sbt-twirl;1.3.12!sbt-twirl.jar (3335ms)
[info] downloading https://repo1.maven.org/maven2/org/scalatra/sbt/sbt-scalatra_2.12_1.0/1.0.1/sbt-scalatra-1.0.1.jar ...
[info] 	[SUCCESSFUL ] org.apache.httpcomponents#httpclient;4.5.2!httpclient.jar (3126ms)
[info] downloading https://repo1.maven.org/maven2/software/amazon/ion/ion-java/1.0.2/ion-java-1.0.2.jar ...
[info] 	[SUCCESSFUL ] org.scalatra.sbt#sbt-scalatra;1.0.1!sbt-scalatra.jar (1806ms)
[info] downloading https://repo1.maven.org/maven2/com/typesafe/play/twirl-compiler_2.12/1.3.12/twirl-compiler_2.12-1.3.12.jar ...
[info] 	[SUCCESSFUL ] com.typesafe.play#twirl-compiler_2.12;1.3.12!twirl-compiler_2.12.jar (883ms)
[info] downloading https://repo1.maven.org/maven2/com/typesafe/play/twirl-api_2.12/1.3.12/twirl-api_2.12-1.3.12.jar ...
[info] 	[SUCCESSFUL ] software.amazon.ion#ion-java;1.0.2!ion-java.jar(bundle) (1941ms)
[info] downloading https://repo1.maven.org/maven2/com/fasterxml/jackson/core/jackson-databind/2.6.6/jackson-databind-2.6.6.jar ...
[info] 	[SUCCESSFUL ] com.typesafe.play#twirl-api_2.12;1.3.12!twirl-api_2.12.jar (791ms)
[info] downloading https://repo1.maven.org/maven2/com/typesafe/play/twirl-parser_2.12/1.3.12/twirl-parser_2.12-1.3.12.jar ...
[info] 	[SUCCESSFUL ] com.typesafe.play#twirl-parser_2.12;1.3.12!twirl-parser_2.12.jar (866ms)
[info] downloading https://repo.scala-sbt.org/scalasbt/sbt-plugin-releases/com.earldouglas/xsbt-web-plugin/scala_2.12/sbt_1.0/4.0.1/jars/xsbt-web-plugin.jar ...
[info] 	[SUCCESSFUL ] com.fasterxml.jackson.core#jackson-databind;2.6.6!jackson-databind.jar(bundle) (2843ms)
[info] downloading https://repo1.maven.org/maven2/com/fasterxml/jackson/dataformat/jackson-dataformat-cbor/2.6.6/jackson-dataformat-cbor-2.6.6.jar ...
[info] 	[SUCCESSFUL ] com.fasterxml.jackson.dataformat#jackson-dataformat-cbor;2.6.6!jackson-dataformat-cbor.jar(bundle) (789ms)
[info] downloading https://repo1.maven.org/maven2/joda-time/joda-time/2.8.1/joda-time-2.8.1.jar ...
[info] 	[SUCCESSFUL ] com.earldouglas#xsbt-web-plugin;4.0.1!xsbt-web-plugin.jar (2691ms)
[info] downloading https://repo1.maven.org/maven2/com/amazonaws/aws-java-sdk-elasticbeanstalk/1.11.105/aws-java-sdk-elasticbeanstalk-1.11.105.jar ...
[info] 	[SUCCESSFUL ] joda-time#joda-time;2.8.1!joda-time.jar (1708ms)
[info] downloading https://repo1.maven.org/maven2/org/apache/httpcomponents/httpcore/4.4.4/httpcore-4.4.4.jar ...
[info] 	[SUCCESSFUL ] com.amazonaws#aws-java-sdk-elasticbeanstalk;1.11.105!aws-java-sdk-elasticbeanstalk.jar (1578ms)
[info] downloading https://repo1.maven.org/maven2/com/amazonaws/aws-java-sdk-s3/1.11.105/aws-java-sdk-s3-1.11.105.jar ...
[info] 	[SUCCESSFUL ] org.apache.httpcomponents#httpcore;4.4.4!httpcore.jar (1054ms)
[info] downloading https://repo1.maven.org/maven2/com/fasterxml/jackson/core/jackson-annotations/2.6.0/jackson-annotations-2.6.0.jar ...
[info] 	[SUCCESSFUL ] com.fasterxml.jackson.core#jackson-annotations;2.6.0!jackson-annotations.jar(bundle) (831ms)
[info] downloading https://repo1.maven.org/maven2/com/fasterxml/jackson/core/jackson-core/2.6.6/jackson-core-2.6.6.jar ...
[info] 	[SUCCESSFUL ] com.amazonaws#aws-java-sdk-s3;1.11.105!aws-java-sdk-s3.jar (1627ms)
[info] downloading https://repo1.maven.org/maven2/com/amazonaws/aws-java-sdk-core/1.11.105/aws-java-sdk-core-1.11.105.jar ...
[info] 	[SUCCESSFUL ] com.fasterxml.jackson.core#jackson-core;2.6.6!jackson-core.jar(bundle) (1238ms)
[info] downloading https://repo1.maven.org/maven2/com/amazonaws/aws-java-sdk-kms/1.11.105/aws-java-sdk-kms-1.11.105.jar ...
[info] 	[SUCCESSFUL ] com.amazonaws#aws-java-sdk-core;1.11.105!aws-java-sdk-core.jar (1825ms)
[info] 	[SUCCESSFUL ] com.amazonaws#aws-java-sdk-kms;1.11.105!aws-java-sdk-kms.jar (1140ms)
[info] Done updating.
[info] Loading settings from build.sbt ...
[info] Set current project to willup (in build file:/usr/local/will/learning/willup/)
[info] sbt server started at 127.0.0.1:4841

```


#### Hello world
到此为止Scalatra已经安装完了，咱们着手弄个小app吧。


