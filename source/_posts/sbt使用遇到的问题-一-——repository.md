title: sbt使用遇到的问题(一)——repository
date: 2015-07-12 19:20:38
tags: [sbt, repository, scala]
---
你懂得，由于当前国内的网络环境，前面使用了oschina的镜像作为sbt的repository。
> [repositories]
> local
> osc: http://maven.oschina.net/content/groups/public/
> oschina-ivy: http://maven.oschina.net/content/groups/public/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext]
> typesafe: http://repo.typesafe.com/typesafe/ivy-releases/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext], bootOnly
> \#sonatype-oss-releases
> \#maven-central
> \#sonatype-oss-snapshots

配置了以上文件repositories后，把它放到~/.sbt/下就可以了。也可以自己制定配置文件的地址。具体可参考：http://www.scala-sbt.org/0.13/docs/Proxy-Repositories.html

但是最近一段时间貌似oschina不再提供maven镜像库的服务了，而最近自己又想学习akka，尝试了好多次都是下面的输出：
```
compile
[info] Updating {file:/root/gitLearning/akka-in-action/chapter-channels/}channels...
[info] Resolving org.scala-lang#scala-library;2.11.6 ...
[warn] 	module not found: org.scala-lang#scala-library;2.11.6
[warn] ==== local: tried
[warn]   /root/.ivy2/local/org.scala-lang/scala-library/2.11.6/ivys/ivy.xml
[info] Resolving com.typesafe.akka#akka-actor_2.11;2.3.10 ...
[warn] 	module not found: com.typesafe.akka#akka-actor_2.11;2.3.10
[warn] ==== local: tried
[warn]   /root/.ivy2/local/com.typesafe.akka/akka-actor_2.11/2.3.10/ivys/ivy.xml
[info] Resolving com.typesafe.akka#akka-slf4j_2.11;2.3.10 ...
[warn] 	module not found: com.typesafe.akka#akka-slf4j_2.11;2.3.10
[warn] ==== local: tried
[warn]   /root/.ivy2/local/com.typesafe.akka/akka-slf4j_2.11/2.3.10/ivys/ivy.xml
[info] Resolving com.typesafe.akka#akka-remote_2.11;2.3.10 ...
[warn] 	module not found: com.typesafe.akka#akka-remote_2.11;2.3.10
[warn] ==== local: tried
[warn]   /root/.ivy2/local/com.typesafe.akka/akka-remote_2.11/2.3.10/ivys/ivy.xml
[info] Resolving com.typesafe.akka#akka-multi-node-testkit_2.11;2.3.10 ...
[warn] 	module not found: com.typesafe.akka#akka-multi-node-testkit_2.11;2.3.10
[warn] ==== local: tried
[warn]   /root/.ivy2/local/com.typesafe.akka/akka-multi-node-testkit_2.11/2.3.10/ivys/ivy.xml
[info] Resolving com.typesafe.akka#akka-contrib_2.11;2.3.10 ...
[warn] 	module not found: com.typesafe.akka#akka-contrib_2.11;2.3.10
[warn] ==== local: tried
[warn]   /root/.ivy2/local/com.typesafe.akka/akka-contrib_2.11/2.3.10/ivys/ivy.xml
[info] Resolving com.typesafe.akka#akka-remote-tests_2.11;2.3.10 ...
[warn] 	module not found: com.typesafe.akka#akka-remote-tests_2.11;2.3.10
[warn] ==== local: tried
[warn]   /root/.ivy2/local/com.typesafe.akka/akka-remote-tests_2.11/2.3.10/ivys/ivy.xml
[info] Resolving com.typesafe.akka#akka-testkit_2.11;2.3.10 ...
[warn] 	module not found: com.typesafe.akka#akka-testkit_2.11;2.3.10
[warn] ==== local: tried
[warn]   /root/.ivy2/local/com.typesafe.akka/akka-testkit_2.11/2.3.10/ivys/ivy.xml
[info] Resolving org.scalatest#scalatest_2.11;2.2.0 ...
[warn] 	module not found: org.scalatest#scalatest_2.11;2.2.0
[warn] ==== local: tried
[warn]   /root/.ivy2/local/org.scalatest/scalatest_2.11/2.2.0/ivys/ivy.xml
[info] Resolving org.scala-lang#scala-compiler;2.11.6 ...
[warn] 	module not found: org.scala-lang#scala-compiler;2.11.6
[warn] ==== local: tried
[warn]   /root/.ivy2/local/org.scala-lang/scala-compiler/2.11.6/ivys/ivy.xml
[warn] 	::::::::::::::::::::::::::::::::::::::::::::::
[warn] 	::          UNRESOLVED DEPENDENCIES         ::
[warn] 	::::::::::::::::::::::::::::::::::::::::::::::
[warn] 	:: org.scala-lang#scala-library;2.11.6: not found
[warn] 	:: com.typesafe.akka#akka-actor_2.11;2.3.10: not found
[warn] 	:: com.typesafe.akka#akka-slf4j_2.11;2.3.10: not found
[warn] 	:: com.typesafe.akka#akka-remote_2.11;2.3.10: not found
[warn] 	:: com.typesafe.akka#akka-multi-node-testkit_2.11;2.3.10: not found
[warn] 	:: com.typesafe.akka#akka-contrib_2.11;2.3.10: not found
[warn] 	:: com.typesafe.akka#akka-remote-tests_2.11;2.3.10: not found
[warn] 	:: com.typesafe.akka#akka-testkit_2.11;2.3.10: not found
[warn] 	:: org.scalatest#scalatest_2.11;2.2.0: not found
[warn] 	:: org.scala-lang#scala-compiler;2.11.6: not found
[warn] 	::::::::::::::::::::::::::::::::::::::::::::::
[warn] 
[warn] 	Note: Unresolved dependencies path:
[warn] 		com.typesafe.akka:akka-contrib_2.11:2.3.10 (/root/gitLearning/akka-in-action/chapter-channels/build.sbt#L14)
[warn] 		  +- manning:akka-sample-multi-node-scala_2.11:0.1-SNAPSHOT
[warn] 		com.typesafe.akka:akka-remote-tests_2.11:2.3.10 (/root/gitLearning/akka-in-action/chapter-channels/build.sbt#L14)
[warn] 		  +- manning:akka-sample-multi-node-scala_2.11:0.1-SNAPSHOT
[warn] 		com.typesafe.akka:akka-remote_2.11:2.3.10 (/root/gitLearning/akka-in-action/chapter-channels/build.sbt#L14)
[warn] 		  +- manning:akka-sample-multi-node-scala_2.11:0.1-SNAPSHOT
[warn] 		com.typesafe.akka:akka-actor_2.11:2.3.10 (/root/gitLearning/akka-in-action/chapter-channels/build.sbt#L14)
[warn] 		  +- manning:akka-sample-multi-node-scala_2.11:0.1-SNAPSHOT
[warn] 		com.typesafe.akka:akka-testkit_2.11:2.3.10 (/root/gitLearning/akka-in-action/chapter-channels/build.sbt#L14)
[warn] 		  +- manning:akka-sample-multi-node-scala_2.11:0.1-SNAPSHOT
[warn] 		org.scala-lang:scala-library:2.11.6 ((sbt.Classpaths) Defaults.scala#L1203)
[warn] 		  +- manning:akka-sample-multi-node-scala_2.11:0.1-SNAPSHOT
[warn] 		com.typesafe.akka:akka-slf4j_2.11:2.3.10 (/root/gitLearning/akka-in-action/chapter-channels/build.sbt#L14)
[warn] 		  +- manning:akka-sample-multi-node-scala_2.11:0.1-SNAPSHOT
[warn] 		com.typesafe.akka:akka-multi-node-testkit_2.11:2.3.10 (/root/gitLearning/akka-in-action/chapter-channels/build.sbt#L14)
[warn] 		  +- manning:akka-sample-multi-node-scala_2.11:0.1-SNAPSHOT
[warn] 		org.scalatest:scalatest_2.11:2.2.0 (/root/gitLearning/akka-in-action/chapter-channels/build.sbt#L14)
[warn] 		  +- manning:akka-sample-multi-node-scala_2.11:0.1-SNAPSHOT
[warn] 		org.scala-lang:scala-compiler:2.11.6
[warn] 		  +- manning:akka-sample-multi-node-scala_2.11:0.1-SNAPSHOT
[trace] Stack trace suppressed: run last *:update for the full output.
[error] (*:update) sbt.ResolveException: unresolved dependency: org.scala-lang#scala-library;2.11.6: not found
[error] unresolved dependency: com.typesafe.akka#akka-actor_2.11;2.3.10: not found
[error] unresolved dependency: com.typesafe.akka#akka-slf4j_2.11;2.3.10: not found
[error] unresolved dependency: com.typesafe.akka#akka-remote_2.11;2.3.10: not found
[error] unresolved dependency: com.typesafe.akka#akka-multi-node-testkit_2.11;2.3.10: not found
[error] unresolved dependency: com.typesafe.akka#akka-contrib_2.11;2.3.10: not found
[error] unresolved dependency: com.typesafe.akka#akka-remote-tests_2.11;2.3.10: not found
[error] unresolved dependency: com.typesafe.akka#akka-testkit_2.11;2.3.10: not found
[error] unresolved dependency: org.scalatest#scalatest_2.11;2.2.0: not found
[error] unresolved dependency: org.scala-lang#scala-compiler;2.11.6: not found
[error] Total time: 0 s, completed Jul 11, 2015 5:13:57 PM
```
今儿灵机一动，想到了会不会是oschina镜像库的问题(尼玛，都忘记了以前自己跟sbt配置过oschina的镜像了)。打开配置文件~~~ 果然如此。。。这里感觉sbt稍微有点儿不好了，看日志也没看到它去哪个网址下载库的，maven库路径是没有输出的，如果没有经验的人看这些东西压根儿看不出啥来。

解决方式： 把原来配置的repositories文件重命名bak一下，重新到项目路径下启动sbt，再次compile：
```
compile
[info] Updating {file:/root/gitLearning/akka-in-action/chapter-channels/}channels...
[info] Resolving org.sonatype.oss#oss-parent;7 ...
[info] downloading https://repo1.maven.org/maven2/org/scala-lang/scala-library/2.11.6/scala-library-2.11.6.jar ...
[info] 	[SUCCESSFUL ] org.scala-lang#scala-library;2.11.6!scala-library.jar (142453ms)
[info] downloading https://repo1.maven.org/maven2/com/typesafe/akka/akka-actor_2.11/2.3.10/akka-actor_2.11-2.3.10.jar ...
[info] 	[SUCCESSFUL ] com.typesafe.akka#akka-actor_2.11;2.3.10!akka-actor_2.11.jar (67998ms)
[info] downloading https://repo1.maven.org/maven2/com/typesafe/akka/akka-slf4j_2.11/2.3.10/akka-slf4j_2.11-2.3.10.jar ...
[info] 	[SUCCESSFUL ] com.typesafe.akka#akka-slf4j_2.11;2.3.10!akka-slf4j_2.11.jar (1201ms)
[info] downloading https://repo1.maven.org/maven2/com/typesafe/akka/akka-remote_2.11/2.3.10/akka-remote_2.11-2.3.10.jar ...
....
[info] Done updating.
[info] Compiling 3 Scala sources to /root/gitLearning/akka-in-action/chapter-channels/target/scala-2.11/classes...
[info] 'compiler-interface' not yet compiled for Scala 2.11.6. Compiling...
[info]   Compilation completed in 20.627 s
[success] Total time: 1518 s, completed Jul 11, 2015 5:41:41 PM
```
额，好吧。成功的时候才会显示地址~
成功了再看这个有毛线作用啊~
