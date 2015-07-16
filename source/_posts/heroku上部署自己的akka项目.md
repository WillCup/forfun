title: heroku上部署自己的akka项目
date: 2015-07-16 02:29:27
tags: [akka, heroku]
---

heroku这个东西貌似是可以在云端发布scala项目的，我是在akka-in-action这本书遇到。既然玩儿了，就顺便记录一下。

## 1. 注册
到heroku.com使用邮箱注册一个免费用户【注意：163邮箱被屏蔽了，我用的阿里的】

## 2. 安装工具
在本地Ubuntu上安装heroku toolbelt, 可参考https://toolbelt.heroku.com/debian
> wget -O- https://toolbelt.heroku.com/install-ubuntu.sh | sh

## 3. 本地登录heroku：
> heroku login

## 4. 在heroku上建立一个app
```
root@ubuntu:~/gitLearning/akka-in-action/chapter2# heroku create
Creating floating-woodland-3032... done, stack is cedar-14
https://floating-woodland-3032.herokuapp.com/ | https://git.heroku.com/floating-woodland-3032.git
Git remote heroku added
```
上面执行的heroku create命令会同时在本地git配置里以heroku为名添加一个git remote。

## 5. 修改参数
我们需要修改几个参数，这样heroku才知道怎样构建我们的项目。

- project/build.properties 【经验证，这个可以不用改】
> sbt.version=0.12.2
- project/plugins.sbt【经验证，这个可以不用改】
> resolvers += Classpaths.typesafeResolver
> addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.8.7")
> addSbtPlugin(
> "com.typesafe.startscript" % "xsbt-start-script-plugin" % "0.5.3"
> )

以上包含了我们目前使用的所有的Plugin【貌似上面提到两个文件只是为了说明我们需要这几个依赖，但是并不严格需要弄成上面的那种】。startscript是为我们在Heroku上运行项目创建一个启动脚本用的。我们还需要项目根目录下的**Procfile**文件，这个文件会告诉Heroku我们的app应该运行在一个Web Dyno上。
可以看一下这个文件里的内容：
> web: target/start com.goticks.Main
这是指定了Heroku应该在启动脚本上添加的主类参数。

## 6. 本地项目运行测试
> sbt clean compile stage

```
root@ubuntu:~/gitLearning/akka-in-action/chapter2# sbt clean compile stage
[info] Loading project definition from /root/gitLearning/akka-in-action/chapter2/project
[info] Set current project to goticks (in build file:/root/gitLearning/akka-in-action/chapter2/)
[success] Total time: 1 s, completed Jul 15, 2015 1:15:05 AM
[info] Updating {file:/root/gitLearning/akka-in-action/chapter2/}chapter2...
[info] Resolving jline#jline;2.12 ...
[info] Done updating.
[warn] Scala version was updated by one of library dependencies:
[warn] 	* org.scala-lang:scala-library:(2.11.2, 2.11.1, 2.11.0) -> 2.11.5
[warn] To force scalaVersion, add the following:
[warn] 	ivyScala := ivyScala.value map { _.copy(overrideScalaVersion = true) }
[warn] Run 'evicted' to see detailed eviction warnings
[info] Compiling 4 Scala sources to /root/gitLearning/akka-in-action/chapter2/target/scala-2.11/classes...
[warn] there was one feature warning; re-run with -feature for details
[warn] one warning found
[success] Total time: 41 s, completed Jul 15, 2015 1:15:47 AM
[info] Wrote start script for mainClass := Some(com.goticks.Main) to /root/gitLearning/akka-in-action/chapter2/target/start
[success] Total time: 1 s, completed Jul 15, 2015 1:15:47 AM
```
这个命令是后面再heroku上会运行的，它会编译项目并创建target/start启动脚本。然后可以使用heroku的命令本地运行Procfile文件：

## 7. 本地启动并测试
使用heroku的工具foreman启动一下项目
> foreman start

```
root@ubuntu:~/gitLearning/akka-in-action/chapter2# foreman start
01:20:02 web.1  | started with pid 7030
01:20:10 web.1  | REST interface bound to /0:0:0:0:0:0:0:0:5000
01:20:10 web.1  | [INFO] [07/15/2015 01:20:10.437] [goticks-akka.actor.default-dispatcher-3] [akka://goticks/user/IO-HTTP/listener-0] Bound to /0.0.0.0:5000
```

到另一个session里测试：
```
root@ubuntu:~# http GET localhost:5000/events
HTTP/1.1 200 OK
Content-Length: 2
Content-Type: application/json; charset=UTF-8
Date: Wed, 15 Jul 2015 08:20:20 GMT
Server: GoTicks.com REST API

[]

```

## 8. 部署到heroku
> git push keroku master

在提交了所有的本地变化后将本地项目push到git仓库的master分支，heroku会hook到这个git进程的执行，并且认出这是一个scala app。它会在云端下载所有依赖，编译代码，之后启动这个app。输出如下:
```
root@ubuntu:~/gitLearning/akka-in-action/chapter2# git push keroku master
fatal: 'keroku' does not appear to be a git repository
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
root@ubuntu:~/gitLearning/akka-in-action/chapter2# git remote
heroku
origin

```
.....报错了。查看remote，发现已经有了heroku这个remote了。参考https://devcenter.heroku.com/articles/git，执行下列命令：
使用 git remote -v, 查看当前项目下的remote url，确定我们的项目名称
```
root@ubuntu:~/gitLearning/akka-in-action/chapter2# git remote -v
heroku	https://git.heroku.com/floating-woodland-3032.git (fetch)
heroku	https://git.heroku.com/floating-woodland-3032.git (push)
origin	https://github.com/RayRoestenburg/akka-in-action.git (fetch)
origin	https://github.com/RayRoestenburg/akka-in-action.git (push)
```

> heroku git:remote -a floating-woodland-3032

```
root@ubuntu:~/gitLearning/akka-in-action/chapter2# heroku git:remote -a floating-woodland-3032
Installing plugin heroku-git... done
set git remote heroku to https://git.heroku.com/floating-woodland-3032.git
```
添加remote参考自: https://devcenter.heroku.com/articles/git【经后面证实，如果不**重命名**，可以不管这个，因为git remote -v 里已经有了。重命名使用-r other_remote_name】
然后再次执行push: git push heroku master
```
Counting objects: 1776, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (791/791), done.
Writing objects: 100% (1776/1776), 298.54 KiB | 0 bytes/s, done.
Total 1776 (delta 564), reused 1768 (delta 561)
remote: Compressing source files... done.
remote: Building source:
remote: 
remote: -----> Play! app detected
remote: -----> Installing OpenJDK 1.6... done
remote: -----> WARNING: Play! version not specified in dependencies.yml. Default version: 1.3.1 being used....
remote: -----> Installing Play! 1.3.1.....
remote: -----> done
remote: -----> Building Play! application...
remote:        ~        _            _ 
remote:        ~  _ __ | | __ _ _  _| |
remote:        ~ | '_ \| |/ _' | || |_|
remote:        ~ |  __/|_|\____|\__ (_)
remote:        ~ |_|            |__/   
remote:        ~
remote:        ~ play! 1.3.1, https://www.playframework.com
remote:        ~
remote:        1.3.1
remote:        Building Play! application at directory ./
remote:        Resolving dependencies: .play/play dependencies ./ --forProd --forceCopy --silent -Duser.home=/tmp/build_4bb492e51a64b714c9d6757c67b0ccf2 2>&1
remote:        ~ !! /tmp/build_4bb492e51a64b714c9d6757c67b0ccf2/conf/dependencies.yml does not exist
remote:  !     Failed to build Play! application
remote:  !     Cleared Play! framework from cache
remote: 
remote:  !     Push rejected, failed to compile Play! app
remote: 
remote: Verifying deploy....
remote: 
remote: !	Push rejected to floating-woodland-3032.
remote: 
To https://git.heroku.com/floating-woodland-3032.git
 ! [remote rejected] master -> master (pre-receive hook declined)
error: failed to push some refs to 'https://git.heroku.com/floating-woodland-3032.git'
```
主要在于这个远程的hook拒绝了我的提交。我擦，简直不能忍！！！Google~~~因为没有成功build play，原因是没有找到play的依赖配置文件。
看了几个解说的，貌似是因为**当前目录是一个git repository的子目录**。

- 下载安装一个git插件
	- git clone https://github.com/apenwarr/git-subtree.git
	- 切换到最近的分支，执行 sh install.sh

- 到git repository目录，也就是本地的git主目录下，我的机器上就是~/gitLearning/akka-in-action，执行：
> git subtree push --prefix "chapter2" heroku master

这个prefix是我们要push的项目相对git repository的路径。
```
git push using:  heroku master
Counting objects: 201, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (119/119), done.
Writing objects: 100% (201/201), 22.30 KiB | 0 bytes/s, done.
Total 201 (delta 54), reused 155 (delta 38)
remote: Compressing source files... done.
remote: Building source:
remote: 
remote: -----> Scala app detected
remote: -----> Installing OpenJDK 1.8... done
remote: -----> Running: sbt compile stage
remote:        Downloading sbt launcher for 0.13.8:
remote:          From  http://repo.typesafe.com/typesafe/ivy-releases/org.scala-sbt/sbt-launch/0.13.8/sbt-launch.jar
remote:            To  /tmp/scala_buildpack_build_dir/.sbt_home/launchers/0.13.8/sbt-launch.jar
remote:        Getting org.scala-sbt sbt 0.13.8 ...
remote:        downloading https://repo.typesafe.com/typesafe/ivy-releases/org.scala-sbt/sbt/0.13.8/jars/sbt.jar ...
remote:        	[SUCCESSFUL ] org.scala-sbt#sbt;0.13.8!sbt.jar (321ms)
remote:        downloading https://repo1.maven.org/maven2/org/scala-lang/scala-library/2.10.4/scala-library-2.10.4.jar ...
remote:        	[SUCCESSFUL ] org.scala-lang#scala-library;2.10.4!scala-library.jar (2957ms)
remote:        downloading https://repo.typesafe.com/typesafe/ivy-releases/org.scala-sbt/main/0.13.8/jars/main.jar ...
.......
......
remote:        	[SUCCESSFUL ] org.scala-lang#jline;2.10.4!jline.jar (73ms)
remote:        downloading https://repo1.maven.org/maven2/org/fusesource/jansi/jansi/1.4/jansi-1.4.jar ...
remote:        	[SUCCESSFUL ] org.fusesource.jansi#jansi;1.4!jansi.jar (49ms)
remote:        :: retrieving :: org.scala-sbt#boot-scala
remote:        	confs: [default]
remote:        	5 artifacts copied, 0 already retrieved (24459kB/42ms)
remote:        [info] Loading global plugins from /tmp/scala_buildpack_build_dir/.sbt_home/plugins
remote:        [info] Updating {file:/tmp/scala_buildpack_build_dir/.sbt_home/plugins/}global-plugins...
remote:        [info] Resolving org.scala-lang#scala-library;2.10.4 ...
.......
.......
remote:        [info] Resolving org.fusesource.jansi#jansi;1.4 ...
remote:        [info] Done updating.
remote:        [info] Compiling 1 Scala source to /tmp/scala_buildpack_build_dir/.sbt_home/plugins/target/scala-2.10/sbt-0.13/classes...
remote:        [error] [/tmp/scala_buildpack_build_dir/project/plugins.sbt]:3: ';' expected but '#' found.
remote:        [error] [/tmp/scala_buildpack_build_dir/project/plugins.sbt]:11: ';' expected but '#' found.
remote:        Project loading failed: (r)etry, (q)uit, (l)ast, or (i)gnore? 
remote:  !     ERROR: Failed to run sbt!
remote:        We're sorry this build is failing! If you can't find the issue in application
remote:        code, please submit a ticket so we can help: https://help.heroku.com
remote:        You can also try reverting to the previous version of the buildpack by running:
remote:        $ heroku buildpacks:set https://github.com/heroku/heroku-buildpack-scala#previous-version
remote:        
remote:        Thanks,
remote:        Heroku
remote: 
remote: 
remote:  !     Push rejected, failed to compile Scala app
remote: 
remote: Verifying deploy....
remote: 
remote: !	Push rejected to floating-woodland-3032.
remote: 
To https://git.heroku.com/floating-woodland-3032.git
 ! [remote rejected] 9bfe06799b9975c15c426ab17618967b7b8e65b0 -> master (pre-receive hook declined)
error: failed to push some refs to 'https://git.heroku.com/floating-woodland-3032.git'

```
可以看到已经成功进行项目的build了，但还是报错了。。。。从输出来看原因是project/plugins.sbt里的#应该被替换成;，好吧，我是sbt新手。。。修改后重新push
```
root@ubuntu:~/gitLearning/akka-in-action# git subtree push --prefix "chapter2" heroku master
git push using:  heroku master
Counting objects: 205, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (123/123), done.
Writing objects: 100% (205/205), 22.61 KiB | 0 bytes/s, done.
Total 205 (delta 56), reused 155 (delta 38)
remote: Compressing source files... done.
remote: Building source:
remote: 
remote: -----> Scala app detected
remote: -----> Installing OpenJDK 1.8... done
remote: -----> Running: sbt compile stage
remote:        Downloading sbt launcher for 0.13.8:
remote:          From  http://repo.typesafe.com/typesafe/ivy-releases/org.scala-sbt/sbt-launch/0.13.8/sbt-launch.jar
remote:            To  /tmp/scala_buildpack_build_dir/.sbt_home/launchers/0.13.8/sbt-launch.jar
remote:        Getting org.scala-sbt sbt 0.13.8 ...
remote:        downloading https://repo.typesafe.com/typesafe/ivy-releases/org.scala-sbt/sbt/0.13.8/jars/sbt.jar ...
.....
.....
remote:        [info] downloading https://repo1.maven.org/maven2/org/scala-lang/modules/scala-parser-combinators_2.11/1.0.2/scala-parser-combinators_2.11-1.0.2.jar ...
remote:        [info] 	[SUCCESSFUL ] org.scala-lang.modules#scala-parser-combinators_2.11;1.0.2!scala-parser-combinators_2.11.jar(bundle) (185ms)
remote:        [info] downloading https://repo1.maven.org/maven2/jline/jline/2.12/jline-2.12.jar ...
remote:        [info] 	[SUCCESSFUL ] jline#jline;2.12!jline.jar (99ms)
remote:        [info] Done updating.
remote:        [warn] Scala version was updated by one of library dependencies:
remote:        [warn] 	* org.scala-lang:scala-library:(2.11.2, 2.11.1, 2.11.0) -> 2.11.5
remote:        [warn] To force scalaVersion, add the following:
remote:        [warn] 	ivyScala := ivyScala.value map { _.copy(overrideScalaVersion = true) }
remote:        [warn] Run 'evicted' to see detailed eviction warnings
remote:        [info] Compiling 4 Scala sources to /tmp/scala_buildpack_build_dir/target/scala-2.11/classes...
remote:        [info] 'compiler-interface' not yet compiled for Scala 2.11.2. Compiling...
remote:        [info]   Compilation completed in 33.95 s
remote:        [warn] there was one feature warning; re-run with -feature for details
remote:        [warn] one warning found
remote:        [success] Total time: 87 s, completed Jul 15, 2015 10:04:22 AM
remote:        [info] Wrote start script for mainClass := Some(com.goticks.Main) to /tmp/scala_buildpack_build_dir/target/start
remote:        [success] Total time: 1 s, completed Jul 15, 2015 10:04:22 AM
remote: -----> Discovering process types
remote:        Procfile declares types -> web
remote: 
remote: -----> Compressing... done, 165.8MB
remote: -----> Launching... done, v5
remote:        https://floating-woodland-3032.herokuapp.com/ deployed to Heroku
remote: 
remote: Verifying deploy.... done.
To https://git.heroku.com/floating-woodland-3032.git
 * [new branch]      75151c92e9b25df8462fc0422b0345cf280f01ce -> master

```

嘻嘻~~ 大功告成！！！现在我们的项目已经发布到https://floating-woodland-3032.herokuapp.com。

## 9. 测试
我们可以使用httpie或者浏览器来测试一下
```
root@ubuntu:~# http PUT https://floating-woodland-3032.herokuapp.com/events event=RHCP nrOfTickets:=10
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 2
Content-Type: text/plain; charset=UTF-8
Date: Wed, 15 Jul 2015 10:09:06 GMT
Server: GoTicks.com REST API
Via: 1.1 vegur

OK

root@ubuntu:~# http GET https://floating-woodland-3032.herokuapp.com/events event=RHCP nrOfTickets:=10
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 44
Content-Type: application/json; charset=UTF-8
Date: Wed, 15 Jul 2015 10:09:21 GMT
Server: GoTicks.com REST API
Via: 1.1 vegur

[
    {
        "event": "RHCP", 
        "nrOfTickets": 10
    }
]

```

## Plus : 二次提交测试
因为还不是很熟悉heroku，自己很纳闷，如果我们远程的APP在运行状态，能否直接提交新的commit，它会自己重启吗？
稍微修改一下项目后，重新push上去报错
```
root@ubuntu:~/gitLearning/akka-in-action# git subtree push --prefix "chapter2" heroku master
git push using:  heroku master
To https://git.heroku.com/floating-woodland-3032.git
 ! [rejected]        855562ff41e936b6769cdf808d0f798f1dbf371b -> master (non-fast-forward)
error: failed to push some refs to 'https://git.heroku.com/floating-woodland-3032.git'
hint: Updates were rejected because a pushed branch tip is behind its remote
hint: counterpart. Check out this branch and integrate the remote changes
hint: (e.g. 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```
原来是因为我刚才修改之前reset了一次，造成了commit滞后。那就先pull下来，解决冲突之后，commit再push。结果还是报同样的错误。。。。无奈，git reflog后再reset到线上的commit，重新修改后commit + push。
```
root@ubuntu:~/gitLearning/akka-in-action# git subtree push --prefix "chapter2" heroku master
git push using:  heroku master
Everything up-to-date
```

heroku这个东西是完全免费的吗？怎样关闭这个东西？它引起了我的好奇心~~~简单看了一下help，有点儿类似docker的感觉。

## PlusPlus: 项目关闭

```
root@ubuntu:~/gitLearning/akka-in-action# heroku ps:stop web.1
Stopping web.1 dyno... done
root@ubuntu:~/gitLearning/akka-in-action# heroku ps
=== web (Free): `target/start com.goticks.Main`
web.1: idle 2015/07/15 03:46:31 (~ 8s ago)

```
关闭了自己又起来了~~~~~靠~！！！！
退出登录，同样失败。
```
root@ubuntu:~/gitLearning/akka-in-action# heroku auth:logout
Local credentials cleared.

```
解决方案：
把web节点扩展到0个。。。。。竟然是这种方式，我真是醉了。。。这时再次访问就是503了。heroku ps没有任何进程。
```
root@ubuntu:~/gitLearning/akka-in-action# heroku ps:scale web=0
Scaling dynos... done, now running web at 0:Free.

```

---
对于git子目录提交的解决，参考自：

- https://gitlab.com/gitlab-com/support-forum/issues/40
- http://stackoverflow.com/questions/14286266/deploy-play-app-on-heroku-from-a-subdirectory-of-git-repo
- http://stackoverflow.com/questions/17148784/play-1-2-5-application-deployment-on-heroku-oops-conf-routes-or-conf-applicati

