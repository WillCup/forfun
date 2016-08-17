title: akka in action 读书笔记（7）
date: 2016-08-17 11:02:12
tags: [akka]
---

- 配置
- 日志
- 单点app
- web app
- 发布

只是了解actor和ActorSysten显然是不够的，一个完成的app在发布之前应该要包含其他必备功能。

7.1 配置
---
akka使用了Typesafe的配置库。可以定义一些属性，然后直接在代码里引用。还有优雅的方式自动按照时间merge并override多个文件中的多个属性。对于配置还有一个比较重要的就是能够分test, dev, prod环境。

#### 7.1.1 牛刀小试
首先看一下Typesafe定义配置文件的实例：
![配置内容示例](/imgs/akka/akka_7_config_example.png)

我们使用ConfigurationFactory来加载配置文件；
```scala
val config = ConfigFactory.load()
```

它会按照以下顺序找默认的配置文件：
 - application.properties
 - application.json
 - application.conf. 这种说明一下是HOCON格式的，基于json，但是可读性更强

也可以同时支持以上所有的文件类型：
```
hostname="localhost" 
MyAppl {
   version = 10
   description = "My application"
   database {
     connect="jdbc:mysql://localhost/mydata"
     user="me"
   }
 }  
```

对于大多数app来说，这些配置就已经足够使用了。这种配置可读性强，易于分组管理属性，JDBC就是一个绝佳的例子。对于依赖注入的框架，需要把item写进控制他们的对象中去，这里有一个更简单的方法。

下面看一下怎样使用上面定义的属性：
```scala
val applicationVersion = config.getInt("MyAppl.version")
val databaseConnectSting = config.getString("MyAppl.database.connect") 
```
也可以分开获取连接串
```scala
val databaseCfg = configuration.getConfig("MyAppl.database")
val databaseConnectSting = databaseCfg.getString("connect") 
```

配置文件中包含的属性一般都是app名称、版本号等，一般会在程序中多次使用它们。

我们还可以使用环境变量来设置变量：
```scala
hostname=${?HOST_NAME}
```
？代表我们会在系统环境变量中获取该变量的值。因为我们并不能保证有这个环境变量，所以还是先定义一个默认值：
```
hostname="localhost"
hostname=${?HOST_NAME}
```

#### 7.1.2 使用默认值

还是看JDBC配置。开发环境我们可能配置地址为localhost，但是有人要看demo的话，肯定要改成别的地址。最简单的方法就是复制配置文件，重新命名这个文件，然后让代码读取这个文件的配置内容。问题是这样我们就在两个位置分别有了两个可以相互替代的配置文件。如果还要切换新环境的话，就成了3个了。

配置库包含一个fall-back机制，默认配置会被放进一个对象中，然后这个对象被作为fall-back机制的配置source。见下图：
![fall-back配置](/imgs/akka/akka_7_config_fallback.png)

注意：这个机制里使用到的属性必须在配置文件中存在默认值，如果找不到的话，就会报错了。

fall-back机制为我们提供默认的配置，那么我们怎样配置默认值呢？应该在jar文件的根目录下的reference.conf配置，每个库都应该包含自己的默认配置。配置库会找到所有的reference.conf文件，整个这些配置到fall-back结构中。这样，某个库所需要的所有的属性都有了默认值。

前面说到配置库是支持多种配置文件的，你可以在一个app中使用多个不同类型的配置文件。每个文件都可以作为下一个文件的fall-back。

下图展示了配置库使用多个文件构建配置tree的优先级：
![配置fall-back结构的优先级](/imgs/akka/akka_7_priority_fallback.png)

多数app只需要一种配置文件就可以了。但是如果你定了多个，那么默认值之间的覆盖关系就是上图中，上面的覆盖下面的配置。

默认读取文件是 application.{conf,json,properties}。有两种方式自定义配置文件，一种是load的时候指定
```scala
val config = ConfigFactory.load("myapp") 
```
这样它就会去找myapp.{conf,json,properties}。

另一种方式是使用系统变量。有时这种使用起来比较方便，我们可以构建bash脚本的时候给app指定(比去jar和war里面找要好一些)。
 - config.resource 此变量指定项目resource中配置文件完整名称，例如： application.conf
 - config.file  文件系统中配置文件的全名
 - config.url   

当使用load不传参的时候，就会使用系统属性了，使用系统属性指定的配置文件的时候，就不会再去搜索各种文件类型了：conf, json and properties。
 
#### 7.1.3 akka配置
上面介绍了怎样配置app的属性，但是如果我们想修改akka本身的一些配置怎么办呢？akka是怎样使用配置库的呢？有可能是有多个ActorSystem，每个Actorsystem都有自己对应的配置。当创建ActorSystem的时候没有指定配置的话，ActorSystem会使用默认值自己创建。

```scala
val system = ActorSystem("mySystem") 
```
这里内部调用了`ConfigFactory.load()`。

我们也可以指定配置文件位置：
```scala
val configuration = ConfigFactory.load("mysystem")
val systemA = ActorSystem("mysystem",configuration) 
```
这个配置就已经存在于这个app了，可以通过如下方式访问；
```scala
val mySystem = ActorSystem("myAppl")
val config = mySystem.settings.config
val applicationDescription = config.getString("myAppl.name") 
```

#### 7.1.4 多系统

有时候，在单个实例上我们需要启动多个子系统，每个子系统应该有自己的配置。

假如我们使用了多个JVM，他们基于同一个jar文件。我们就是用系统变量来解决这个问题。每启动一个新的进程，就是用一个不同的配置文件。但是配置里多数都是相同的，只有少数几个配置需要修改。我们使用include来解决这个问题

下面是baseConfig.conf
```
MyAppl {
 version = 10
 description = "My application"
 }

```

我们使用一个共享变量version，还有不同的变量description。下面是子系统的配置文件subAppl.conf
```
include "baseConfig"
MyAppl {
 description = "Sub Application"
}
```
注意include的位置。

如果子系统运行在同一个jvm里呢？那样我们就不能使用系统变量来读取其他的配置文件了。那我们就把两个系统的配置文件合二为一，指定app名称，下面是combined.conf
```
MyAppl {
   version = 10
   description = "My application"
}
subApplA {
   MyAppl {
   description = "Sub application"
   }
}
```
这里我们使用了subApplA的子树，然后把它放在了configuration chain的上端。This is called lifting a configuration, because the configuration path is shortened.。见下图；
![Lifting configuration part](/imgs/akka/akka_7_lift_config_part.png)

当我们请求MyAppl.description的时候，返回"Sub application"因为它被设置在配置中的最高层级上，请求MyAppl.version获得10，因为配置的高层级上没有它的定义，所以它使用fall-back机制获取到了10.

下面代码展示了怎样同时使用lift和callback，注意fallback被chain了起来：
```scala
val configuration = ConfigFactory.load("combined")
 val subApplACfg = configuration.getConfig("subApplA")
 // 把configuration作为fall-back
 val config = subApplACfg.withFallback(configuration)
```

说的不多，但是这些配置就已经够用了。

7.2 日志
---

下一个app必备的功能就是写日志。日志库这个东西众口难调，akka实现了一个日志adapter支持多种日志框架。日志关心两个东西，定义log级别以及定义log内容。

#### 7.2.1 Akka app中记录日志
```scala
class MyActor extends Actor {
 val log = Logging(context.system, this)
 ...
 }
```

需要注意的是，创建logger的时候需要使用ActorSystem。 logging adapter使用ActorSystem的eventStream发送日志信息给eventhandler。eventStream是Akka中的订阅系统。eventhandler收到日志信息后，使用定义的日志框架记录下来。这样，每个actor都能记录日志，而只有一个eventHandler actor依赖指定的日志框架。另一个好处就是IO，IO在并发环境下会特别慢。在高性能的app里，我们不希望等待打日志的过程。使用eventHandler就不用任何等待。下面列出了默认创建event-handler需要的配置信息：
```
akka {
 # Event handlers to register at boot time
 # (Logging$DefaultLogger logs to STDOUT)
 event-handlers = ["akka.event.Logging$DefaultLogger"]
 # Options: ERROR, WARNING, INFO, DEBUG
 loglevel = "DEBUG"
 }
```
eventHandler并没有使用任何log框架，而是把log到了标准输出。下面是一个自定义的eventHandler代码：
```scala
import akka.event.Logging.InitializeLogger
import akka.event.Logging.LoggerInitialized
import akka.event.Logging.Error
import akka.event.Logging.Warning
import akka.event.Logging.Info
import akka.event.Logging.Debug
class MyEventListener extends Actor
{
   def receive = {
     case InitializeLogger(_) =>
        sender ! LoggerInitialized
     case Error(cause, logSource, logClass, message) =>
        println( "ERROR " + message)
     case Warning(logSource, logClass, message) =>
        println( "WARN " + message)
     case Info(logSource, logClass, message) =>
        println( "INFO " + message)
     case Debug(logSource, logClass, message) =>
        println( "DEBUG " + message)
   }
}
```

这只是一个简单的示例。Akka内置了两个eventHandler的实现，一个是上面的输出到STDOUT的，还有一个就是SLF4J。需要在配置文件中添加：
```
akka {
 event-handlers =
 ["akka.event.slf4j.Slf4jEventHandler"]
 # Options: ERROR,
 WARNING,
 INFO, DEBUG
 loglevel = "DEBUG"
 }
```


#### 7.2.2 使用日志功能
重新定义logger：
```scala
class MyActor extends Actor {
 val log = Logging(context.system, this)
 ...
 }
```

第二个参数就是日志的source。这里就是这个类的实例。可以mixin ActorLogging trait来简化代码。

打日志过程中还可以使用placeholder
```scala
log.debug("two parameters: {}, {}", "one","two")
```


#### 7.2.3 控制日志功能

有时我们想自由控制日志的输出内容，比如查看生产环境的某些信息。看一下这个配置
```
akka {
   # logging must be set to DEBUG to use any of the options below
   loglevel = DEBUG
   # Log the complete configuration at INFO level when the actor
   # system is started. This is useful when you are uncertain of
   # what configuration is used.
   log-config-on-start = on
   debug {
   # logging of all user-level messages that are processed by
   # Actors that use akka.event.LoggingReceive enable function of
   # LoggingReceive, which is to log any received message at
   # DEBUG level
   # 把所有启用了akka.event.LoggingReceive的收到的user-leve的debug level的日志信息都打印出来
   receive = on

   # enable DEBUG logging of all AutoReceiveMessages
   # (Kill, PoisonPill and the like)
   autoreceive = on
   # enable DEBUG logging of actor lifecycle changes
   # (restarts, deaths etc)
   lifecycle = on
   # enable DEBUG logging of all LoggingFSMs for events,
   # transitions and timers
   fsm = on
   # enable DEBUG logging of subscription (subscribe/unsubscribe)
   # changes on the eventStream
   event-stream = on
   }
   remote {
   # If this is "on", Akka will log all outbound messages at
   # DEBUG level, if off then they are not logged
   log-sent-messages = on
   # If this is "on," Akka will log all inbound messages at
   # DEBUG level, if off then they are not logged
   log-received-messages = on
   }
}
```


对应使用logger的代码是：
```scala
class MyActor extends Actor with ActorLogging {
 def receive = LoggingReceive {
 case ... => ...
 }
}
```

现在，只要我们设置属性akka.debug.receive 为on，我们actor收到的所有message就都可以被log出来了。

好了。日志就介绍到这里。


7.3 发布基于Akka的app
---
这一节介绍发布应用的两种方式，一种是独立app，另一种是基于play-min的web app。

#### 7.3.1 独立app
我们使用akka的插件MicroKernel创建独立app。先弄一个HelloWorld Actor，只接收message，恢复hello。
```scala
class HelloWorld extends Actor
   with ActorLogging {
     def receive = {
         case msg:String =>
         val hello = "Hello %s".format(msg)
         sender ! hello
         log.info("Sent response {}",hello)
     }
}
```

下面我们创建HelloWorldCaller，它负责调用上面的 HelloWorld actor。
```scala
class HelloWorldCaller(timer:Duration, actor:ActorRef)
 extends Actor with ActorLogging {
 
   case class TimerTick(msg:String)

   override def preStart() {
       super.preStart()
       // 使用akka scheduler给自己发送message
       context.system.scheduler.schedule(
         timer, 	// 首次触发之前的时长
         timer,		// 周期 
         self,		// 发往的actor，这里是自己
         new TimerTick("everybody"))	// 发出的message
   }
   def receive = {
       case msg: String => log.info("received {}",msg)
       case tick: TimerTick => actor ! tick.msg
   }
}
```

这个actor使用内置的schuduler周期性的生成message。每次收到TimerTick，我们都给构造方法中传进来的actor发送一个这个tick的message。当接收到一个String的message的时候，就用日志记录下来。

下面使用akka内核来差un构建系统，这个内核包含了一个启动接口，我们就是要实现这个接口。
```scala
import akka.actor.{ Props, ActorSystem }
import akka.kernel.Bootable
import scala.concurrent.duration._
// 继承Bootable trait，启动app的时候就会被调用
class BootHello extends Bootable {
     
     val system = ActorSystem("hellokernel")

     // 当app启动的时候，会调用这个接口方法
     def startup = {
         val actor = system.actorOf(Props[HelloWorld])
         val config = system.settings.config
         val timer = config.getInt("helloWorld.timer")
         system.actorOf(Props(new HelloWorldCaller(timer millis, actor)))
     }
     
     def shutdown = {
         system.shutdown()
     }
}
```

现在我们已构建好了系统，还需要一些资源让我们的app运行起来。我们先写一下配置文件reference.conf
```
helloWorld {
 timer=5000
 }
```

我们的默认timer值是5000毫秒。注意reference.conf文件需要放在jar文件的根目录下。下面我们在application.conf里配置logger相关
```
akka {
 event-handlers =
 ["akka.event.slf4j.Slf4jEventHandler"]
 # Options: ERROR,
 WARNING,
 INFO, DEBUG
 loglevel = "DEBUG"
 }
```

万事俱备。下面使用 akka-sbt-plugin 来创建依赖配置文件, 编辑plugins.sbt
```
resolvers += "Typesafe Repository"
  at "http://repo.akka.io/releases/" 

  addSbtPlugin("com.typesafe.akka" % "akka-sbt-plugin" % "2.0.1")
```

为不熟悉sbt的同学解释一下。第一行添加了repository的地址。然后是一个空行，这个注意一下，它代表前一行完事儿了。下一行定义了我们需要的插件。


然后创建build文件， HelloKernelBuild.scala
```scala
import sbt._
 import Keys._
 import akka.sbt.AkkaKernelPlugin
 import akka.sbt.AkkaKernelPlugin.{ Dist,
     outputDirectory,
     distJvmOptions
 }
 object HelloKernelBuild extends Build {
     lazy val HelloKernel = Project(
         id = "hello-kernel-book",
         base = file("."),
         settings = defaultSettings
         	++ AkkaKernelPlugin.distSettings // 添加akka插件的功能
         	++ Seq(
			// 构建需要的依赖
             		libraryDependencies ++= Dependencies.helloKernel,
             		distJvmOptions in Dist := "-Xms256M -Xmx1024M",
			// dist输出目录
             		outputDirectory in Dist := file("target/helloDist")
         	)
     )
     lazy val buildSettings = Defaults.defaultSettings
     ++ Seq(
         organization := "com.manning",
         version := "0.1-SNAPSHOT",
         scalaVersion := "2.9.1",
         crossPaths := false,
         organizationName := "Mannings",
         organizationHomepage := Some(url("http://www.mannings.com"))
     )
     lazy val defaultSettings = buildSettings ++ Seq(
         resolvers += "Typesafe Repo" at  "http://repo.typesafe.com/typesafe/releases/",
         // compile options
         scalacOptions ++= Seq("-encoding", "UTF-8", "-deprecation",  "-unchecked"),
         javacOptions ++= Seq("-Xlint:unchecked", "-Xlint:deprecation")
     )
}  

 // Dependencies
 object Dependencies {
   import Dependency._
   val helloKernel = Seq(akkaActor, akkaKernel, akkaSlf4j, lf4jApi, slf4jLog4j, Test.junit, Test.scalatest, Test.akkaTestKit)
 }

object Dependency {
     // Versions
     object V {
       val Scalatest = "1.6.1"
       val Slf4j = "1.6.4"
       val Akka = "2.0"
     }
     // Compile
     val commonsCodec = "commons-codec" % "commons-codec"% "1.4"
     val commonsIo = "commons-io"  % "commons-io" % "2.0.1"
     val commonsNet = "commons-net"  % "commons-net" % "3.1"
     val slf4jApi = "org.slf4j" % "slf4j-api" % V.Slf4j
     val slf4jLog4j = "org.slf4j"  % "slf4j-log4j12"% V.Slf4j
     val akkaActor = "com.typesafe.akka" % "akka-actor" % V.Akka
     val akkaKernel = "com.typesafe.akka" % "akka-kernel" % V.Akka
     val akkaSlf4j = "com.typesafe.akka"  % "akka-slf4j" % V.Akka
     val scalatest = "org.scalatest" %% "scalatest" % V.Scalatest
     object Test {
         val junit = "junit" % "junit" %  "4.5" % "test"
         val scalatest = "org.scalatest" %% "scalatest" % V.Scalatest % "test"
         val akkaTestKit ="com.typesafe.akka"  % "akka-testkit" % V.Akka % "test"
     }
 }
```

万事俱备了。进入项目根目录，启动sbt，执行dist命令
```
sbt
[info] Loading project
definition from J:\boek\manningAkka\
listings\kernel\project
[info]
Set current project to hello-kernel-book (in build
 file:/J:/boek/manningAkka/listings/kernel/)
> dist
```

完事儿后，查看 /target/helloDist目录，可看到4个子目录：
 - bin 启动脚本所在的地方，包含linux和window的
 - config 运行app所需的配置文件
 - deploy jar文件所在的地方
 - lib 依赖的jar文件所在的地方

使用启动脚本，启动我们的app：
```
./start BootHello
```

观察log文件。

#### 7.3.2 创建web app

相比上面独立app，这里并不多做什么。 Play-mini是基于Play的扩展。我们要做的app只是提供接口，并不提供界面。有其他的工具可以，我们只是觉得 Play-mini来做例子
```scala
object PlayMiniHello extends Application {
 lazy val system = ActorSystem("webhello")
 lazy val actor = system.actorOf(Props[HelloWorld])
}
```
只要继承Application, 就集成了好多的功能，就像前面的kernel一样。web app中需要创建路由。
```scala
object PlayMiniHello extends Application {
 def route = {
 	case GET(Path("/test")) => Action {
 		Ok("TEST @ %s\n".format(System.currentTimeMillis))
	 }
 }
}
```

在面向服务的架构里，整个app甚至就可以仅仅基于几个这样的服务映射。相比上面的kernel方式，这是唯一的区别。下面我们创建hello请求的路由，它接收一个name参数，当没有这个参数的时候就使用我们配置里的默认值。下面是从REST路径里获取参数的方式
```scala
val writeForm = Form("name" -> text(1,10))
```

这里使用了Play的From，约束长度只能在1到10之间。如果获取失败的话，就获取配置参数helloWorld.name。
```scala
case GET(Path("/hello")) => Action {
  // 把request定已成implicit类型，方便下面进行变量绑定 
  implicit request =>
       val name = try {
	  // 将request和我们的form进行绑定匹配
          writeForm.bindFromRequest.get
       } catch {
           case ex:Exception => {
             log.warning("no name specified")
	     // 获取默认的配置参数
             system.settings.config.getString("helloWorld.name")
           }
       }
   ...
}
```

现在有了name，我们可以发送信息给指定的actor。但是在创建response之前我们必须等着结果，可我们不希望阻塞。所以我们返回AsyncResult，它需要一个promise。promise很像Futrue，只是Futrue是在client端使用的。有了promise，结果就一直等在producer那儿，完成处理后producer会提供这个结果。关于Futrue和Promise的更多细节会在下一章涉及。
```scala
AsyncResult {
   // 使用ask方法发送request
   val resultFuture = actor ? name 
   // 使用futrue创建一个promise
   val promise = resultFuture.asPromise
   // 返回结果后，就创建response
   promise.map {
       case res:String => {
          Ok(res)
       }
   }
}
```

要使用 HelloWorld actor的response创建一个response，我们必须使用ask方法。ask方法返回一个Futrue，使用Future创建一个Promise。最后一步就是使用返回的结果填充进Promise。

如果超时没有收到response呢，这样map方法就不会被调用，promise里也不会被填充结果。要解决这个问题，我们继承Future，当失败的时候返回一个字符串。
```scala
val resultFuture = actor ? name recover {
     case ex:AskTimeoutException
          => "Timeout"

     case ex:Exception => {
         log.error("recover from "+ex.getMessage)
         "Exception:" + ex.getMessage
     }
 }
```

总结一下变化：
 - 从resource文件中加载默认属性
 - 抽取path中的参数
 - 处理请求超时
 - 并发地处理所有问题

用下面这个例子来总结一下这一章的所学：
```scala
// 实现 Application trait
object PlayMiniHello extends Application {

     lazy val system = ActorSystem("webhello")

     lazy val actor = system.actorOf(Props[HelloWorld])

     implicit val timeout = Timeout(1000 milliseconds)

     val log = Logging(system,PlayMiniHello.getClass)

     def route = {
         case GET(Path("/test")) => Action {
             Ok("TEST @ %sn".format(System.currentTimeMillis))
         }

	 // 使用 helloworld actor 创建一个request
         case GET(Path("/hello")) => Action {
	     // 设置request为implicit
             implicit request =>
             val name = try {
		    // 获取name参数，具体参考Play文档
                    writeForm.bindFromRequest.get
               } catch {
                     case ex:Exception => {
                         log.warning("no name specified")
                         system.settings.config.getString("helloWorld.name")
                     }
               }

	       // 由于不想阻塞，我们返回AsyncResult
               AsyncResult {
		 // 担心超时，创建一个recover回调函数
                 val resultFuture = actor ? name recover {
                     case ex:AskTimeoutException => "Timeout"
                     case ex:Exception => {
                         log.error("recover from "+ex.getMessage)
                         "Exception:" + ex.getMessage
                     }
                 }

                 val promise = resultFuture.asPromise

                 promise.map {
                     case res:String => {
                         log.info("result "+res)
                         Ok(res)
                     }
                     case ex:Exception => {
                         log.error("Exception "+ex.getMessage)
                         Ok(ex.getMessage)
                     }
                     case _ => {
                     Ok("Unexpected message")
                     }
                 }
             }
         }
    }
    val writeForm = Form("name" -> text(1,10))
}
```

此外我们还要处理一些小事情，指定启动app的类，这里是"PlayMiniHello."。
```scala
object Global extends com.typesafe.play.mini.Setup(ch04.PlayMiniHello)
```

继承了 play.api.GlobalSettings trait，这样我们就可以使用ActorSystem的onStart和onStop方法了。

编辑配置文件reference.conf
```
helloWorld {
 name=world
}
```

编辑application.conf
```
helloWorld {
    name="world!!!"
}
akka {
   event-handlers = ["akka.event.slf4j.Slf4jEventHandler"]
   # Options: ERROR, WARNING, INFO, DEBUG
   loglevel = "DEBUG"
}
```

编辑项目构建文件Build.scala
```scala
import sbt._
import Keys._
import PlayProject._
object Build extends Build {
     lazy val root = Project(id = "playminiHello",
     base = file("."), settings = Project.defaultSettings).settings(
     resolvers += "Typesafe Repo" at
     "http://repo.typesafe.com/typesafe/releases/",
     resolvers += "Typesafe Snapshot Repo" at
     "http://repo.typesafe.com/typesafe/snapshots/",
     libraryDependencies ++= Dependencies.hello,
     // 加上这一行，就可以支持sbt测试webapp了
     mainClass in (Compile, run) := Some("play.core.server.NettyServer"))
}
object Dependencies {
     import Dependency._
     val hello = Seq(akkaActor,
         akkaSlf4j,
         // slf4jLog4j,
         playmini
     )
}
object Dependency {
     // Versions
     object V {
         val Slf4j = "1.6.4"
         val Akka = "2.0"
     }
     // Compile
     val slf4jLog4j = "org.slf4j" % "slf4j-log4j12"% V.Slf4j
     val akkaActor = "com.typesafe.akka" % "akka-actor" % V.Akka
     val playmini = "com.typesafe" %% "play-mini" % "2.0-RC3"
     val akkaSlf4j = "com.typesafe.akka" % "akka-slf4j" % V.Akka
} 
```

进入sbt，运行run。app会运行在9000端口上，可以使用curl进行测试了。


7.4 总结
---

写的太绕口了。。不翻译了。。。。。akka各种牛逼。。。
