title: akka in action 读书笔记（6）
date: 2016-08-15 18:51:25
tags: [akka]
---

这一章我们介绍：
- scale out
- 分布式的卖票app
- 远程使用
- 测试分布式的actor系统

这一章中，我们会拿出第二章的例子来进行scale out，也就是在更多的机器上部署同一个app。【回忆一下scale up，在更多的jvm上部署同一个app】

这一章只介绍scale out，下一章介绍集群使用。13章会深入scale out的一些细节。

6.1 scale out
---
不要指望一下子就能够把任何app扩展到几千台机器，分布式计算是众所周知的难点。akka只是让分布式计算简单一些，让并发编程简单一些，我们这里还是使用 GoTicks.com 项目，把它弄成分布式的。

许多网络技术都使用RPC，这种方式尽量掩饰本地调用与远程调用的不同，主要思想是既然本地调用很简单，那么就让编程人员像调用本地方法一样调用远程方法。这种通信方式对于点对点的server连接是有效的，但是对于大型网络并不是一个好的方案。面向message的中间件可以解决这个问题，但使app理解吸收message系统到app层的代价有些大。akka另辟蹊径，远程协作者相对透明，不用修改akka代码。

进入探讨之前，先看一下网络拓扑的例子和一些通用术语。要是你对这些已经很了解了就直接跳到6.2.

#### 6.1.1 通用网络拓扑

![通用网络拓扑](/imgs/akka/akka_6_common_network_topo.png)

- Node: 在网络中运行着的一个app，是网络拓扑的一个连接点。一个server上可以有多个node。
- Role： 每个node在分布式系统中都有自己指定的Role。它代表着这个node可以执行一些列的特定任务。
- transport protocol ： node使用指定的传输协议来互相沟通，比如TCP/IP
- membership：当许多node在同一个分布式系统中的时候，他们公用一个group membership。这个membership可以是静态或动态的。静态是指node的数量和角色都是固定不变的。动态的是允许node自由加入或离开网络，也可能在不同时刻担任不同的角色。静态的最简单，也最脆弱。动态的则可以随着node的增多动态调整node之间的关系，可以应对一些错误问题。当有node加入或者离开网络的时候，需要有一种discovery机制。

#### 6.1.2 为何要分布式编程模型
扩展多n多node的第一步是县弄出一个本地的app支持分布逻辑？我们可以只定义抽象的处理逻辑，然某个牛叉的工具去处理这些转化细节么？答案是no。只指出分布式和本地环境的区别是不够的。根据论文 A Note on Distributed Computing所说，在本地编程和分布式编程，有4个重要的区别：延迟、内存访问、部分失败、并发。下面是一个简单的总结：
- 延迟：在协作者之间出现网络意味着每个message都需要更长的时间，可能由于各种网络原因造成延迟
- 部分失败：当系统的部分功能并不总是可见、有时消失有时又出现，我们很难确定整个系统都能正常工作。
- 内存访问： 在本地系统中获取一个内存对象的引用基本不会失败，但是对于分布式系统就失败概率就变大了。
- 并发： no one 'owner' of everything

由于以上因素，我们不能在分布式环境中使用本地编程模型。akka的切入点恰恰相反，只需要一个分布式编程模型，同时适用于本地系统和分布式系统。前面提到的论文也提到这个观点，但是有些关心会不会把编程难度提高到分布式编程的水平。

时代变了。前面章节我们介绍到了并发编程的易用性。我们已经熟用了异步迭代，部分失败处理，并发之间不共享任何状态，简化了多核编程，我们已经准备好弄一个分布式环境了。

6.2 scale out with remoting
---
我们先选区一个最简单的网络拓扑模型：使用两个node组成client-server 静态网络拓扑。两个node一个是前端，一个是后端。REST接口运行在前端node上，BoxOffice和所有的TicketSeller运行在后端node上。node之间互相有一个静态的网络地址引用。如下图：
![从单个节点到client->server的变化](/imgs/akka/akka_6_sigle_to_cli_serv.png)

我们需要使用到akka-remote module。

在本地模式中，当有新的Event的时候，BoxOffice Actor就创建TicketSeller actor。在client-server拓扑中也是这样。只是我们可以看到使用akka-remote之后就可以远程创建与发布actor了。前端使用它知道的地址在后端找负责创建TicketSeller的BoxOffice actor。注意还有一个变化就是前端在后端远程部署了一个BoxOffice actor。

#### 6.2.1 动手写程序
在源码的chapter6文件夹里有一个对chapter2的升级版。

首先我们修改以下sbt文件，添加akka-remote和akka-mutlnode-testkit依赖：
```
"com.typesafe.akka"       %%  "akka-remote"                    % akkaVersion,
"com.typesafe.akka"       %%  "akka-multi-node-testkit"        % akkaVersion   % "test",
```
调用sbt update来pull这些依赖。

#### 6.2.2 远程REPL

akka提供两种方式来获取远程node上的actor引用。一种是通过path，另一种是创建actor，获取它的引用，然后远程发布。先介绍第一种。

REPL控制台是一个很好的交互工具。进入到chapter6目录下，执行sbt console进入REPL，我们需要开启两个session，也就是两个REPL。为了使远程生效，我们需要在src/main/resources下提供一个配置文件application.conf，REPL启动的时候会自动加载它。

在REPL中执行:paste后，贴入以下代码：
```scala
val conf="""
akka {
  actor {
    provider = "akka.remote.RemoteActorRefProvider" // 选择 Remote ActorRef Provider来启动远程
  }
 
  // 远程配置
  remote { 
    enabled-transports = ["akka.remote.netty.tcp"] // 启用TCP
    netty.tcp {
      hostname = "0.0.0.0"
      port = 2552
    }
  }
}
"""
```

我们会将这段配置信息加载进ActorSystem。需要注意的是它定义了一个ActorRefProvider来启动akka-remote module。顾名思义，他就是负责远程Actor的ActorRef。

下面代码首先引入必须的配置和actor包，然后把配置加载进了ActorSystem：
```scala
scala> import com.typesafe.config._
 import com.typesafe.config._

 scala> import akka.actor._
 import akka.actor._

// 解析字符串配置为对象config
 scala> val config = ConfigFactory.parseString(conf)
 config: com.typesafe.config.Config = ....

// 使用config对象创建ActorSystem
 scala> val backend = ActorSystem("backend", config)
 [Remoting] Starting remoting
 .....
 [Remoting] Remoting now listens on addresses:
 [akka.tcp://backend@0.0.0.0:2551]
 backend: akka.actor.ActorSystem = akka://backend
```

随着敲完最后一行，我们启动了一个启动了远程的ActorSystem。通过启动远程的config对象，我们启动了后端ActorSystem。如果忘了没有使用这个config，默认akka的application.conf里是不启用远程功能的。下面我们弄一个直接输出输入数据的小的actor：
```scala
 scala> :paste
 // Entering paste mode (ctrl-D to finish)

 class Simple extends Actor {

 def receive = {
 case m => println(s"received $m!")
 }
 }
 // Exiting paste mode, now interpreting.
 // 使用后端ActorSystem创建这个简单的Actor并命名为simple
 scala> backend.actorOf(Props[Simple], "simple") 
```

这个简单的actor已经运行在后端ActorSystem上了，并且有了名字simple，这样其他的node就可以找到它了。下面我们另起一个终端，开始编写前端：
```scala
val conf = """
akka {

  actor {
    provider = "akka.remote.RemoteActorRefProvider"
  }

  remote {
    enabled-transports = ["akka.remote.netty.tcp"]
    netty.tcp {
      hostname = "0.0.0.0"
      port = 2551 // 注意端口不同
    }
  }
}
"""

 import com.typesafe.config._

 import akka.actor._

 val config = ConfigFactory.parseString(conf)
 val frontend= ActorSystem("frontend", config)
 [Remoting] Starting remoting
 .....
 [Remoting] Remoting now listens on addresses:
 [akka.tcp://backend@0.0.0.0:2552]
 frontend: akka.actor.ActorSystem = akka://frontend
```

config同样被加载进了前端的actorSystem。前端actorSystem已经启用了远程，而且启动起来了。下面我们想办法在前端获取Simple Actor的引用。我们首先构建一个actor path。下图是path的组成：
![actor path的组成](/imgs/akka/akka_6_remote_actor_path.png)

我们可以使用前端actorSystem的actorSelection方法通过path获取到后端ActorSystem的simple ActorRef：
```scala
scala> :paste
// Entering paste mode (ctrl-D to finish)
val path = "akka.tcp://backend@0.0.0.0:2551/user/simple"
val simple = frontend.actorSelection(path)
// Exiting paste mode, now interpreting.
path: String = akka.tcp://backend@0.0.0.0:2551/user/simple
simple: akka.actor.ActorSelection =
ActorSelection[Actor[akka.tcp://backend@0.0.0.0:2551/]/user/simple]
```
ctorSelection可以用来发送一些message给所有符合条件的actor。下面我们就在前端发送一个信息给后端：
```scala
 scala> simple ! "Hello Remote World!"
```

切换到后端的终端上，可以看到，我们收到并打印出了这条message：
```scala
 scala> received Hello Remote World!!
```

过程是message被序列化，发送给一个TCP socket，通过远程module收到，然后反序列化，再交给后端的Simple Actor处理。

#### 6.2.3 远程查找

我们并不会直接在RestInterface actor中直接创建BoxOffice actor，而是在后端node上找BoxOffice。如下图：
![远程查找BoxOffice actor](/imgs/akka/akka_6_remotelooup_boxoffice_actor.png)
前面版本的RestInterface是直接创建的BoxOffice actor：
```scala
val boxOffice = context.actorOf(Props[BoxOffice], "boxOffice")
```

这样调用的话二者就是直接的父子关系。为了让app更灵活一些，使之也能适应分布式系统，我们把这段代码放进一个trait，然后后面再mixin。
```scala
// 这个trait必须被mixin进一个actor才能使用context
trait BoxOfficeCreator { this: Actor =>
  def createBoxOffice:ActorRef = context.actorOf(Props[BoxOffice],"boxOffice")
}

// RestApi trait包含了RestInterface actor的所有逻辑
class RestInterface extends HttpServiceActor
                    with RestApi {
  def receive = runRoute(routes)
}


trait RestApi extends HttpService with ActorLogging with BoxOfficeCreator { actor: Actor =>
  val boxOffice = createBoxOffice //使用BoxOfficeCreator trait中定义的方法创建BoxOffice
	......
	.....
```

这样我们把创建BoxOffice的代码抽取到了一个trait中，然后在RestInterface把它设置为创建本地boxOffice的默认方法。

下面就定义三个入口方法：SingleNodeMain、FrontendMain、BackendMain。

```scala
// 单点
  val system = ActorSystem("singlenode", config)

  val restInterface = system.actorOf(Props[RestInterface],
                                     "restInterface")

// FrontendMain
  val config = ConfigFactory.load("frontend")
  val system = ActorSystem("frontend", config)


  // mixin RemoteBoxOfficeCreator trait 
  class FrontendRestInterface extends RestInterface
                              with RemoteBoxOfficeCreator

  // 使用mixin的RemoteBoxOfficeCreator trait的FrontendRestInterface创建rest接口
  val restInterface = system.actorOf(Props[FrontendRestInterface],
                                     "restInterface")

// BackendMain
  val config = ConfigFactory.load("backend")
  val system = ActorSystem("backend", config)

  system.actorOf(Props[BoxOffice], "boxOffice")
```

以上三个入口分别加载的是singlenode.conf, frontend.conf ， backend.conf配置文件里的配置。frontend.conf相对多出一个部分来指定怎样找boxoffice actor。RemoteBoxOfficeCreator负责加载这部分配置：
```
backend {
  host = "0.0.0.0"
  port = 2552
  protocol = "akka.tcp"
  system = "backend"
  actor = "user/boxOffice"
}
```

boxoffice actor的路径是从这个配置部分获取并构建的。下面就是像前面REPL一样使用Actor Selection获取远程Actor，一旦确认后端正常，我们就可以尝试发送message了。这次我们希望使用ActorRef而不是单节点那样使用，下面是RemoteBoxOfficeCreator的代码：
```scala
object RemoteBoxOfficeCreator {
  val config = ConfigFactory.load("frontend").getConfig("backend")
  val host = config.getString("host")
  val port = config.getInt("port")
  val protocol = config.getString("protocol")
  val systemName = config.getString("system")
  val actorName = config.getString("actor")
}

trait RemoteBoxOfficeCreator extends BoxOfficeCreator { this:Actor =>
  import RemoteBoxOfficeCreator._

  def createPath:String = {
    s"$protocol://$systemName@$host:$port/$actorName"
  }

  override def createBoxOffice:ActorRef = {
    val path = createPath
    context.actorOf(Props(classOf[RemoteLookup],path), "lookupBoxOffice") // 创建一个负责寻找boxoffice actor的actor，这个actor的参数是远程boxoffice的path
  }
}

```

RemoteBoxOfficeCreator创建了一个额外的actor RemoteLookup来寻找boxOffice。在前面版本的akka里你可能会使用actorFor方法来直接获取一个远程actor的ActorRef。但是这个方法目前已经过期了，因为如果actor本身挂掉的话，他返回的ActorRef的表现并不完全像本地actor一样。还有就是通过actorFor返回的ActorRef可以指向一个新建的远程actor实例，这对本地context来说绝无可能。再一个原因就是不能像本地actor那样监控它的终结message。所以把这个方法废除了。取而代之的是RemoteLookup actor：
- 后端ActorSystem可能还没有启动，或者暂时挂掉了，正在重启
-  boxOffice actor本身也可能挂掉重启中
- 理想状态下我们能先于前端启动后端

RemoteLookup负责处理以上这些场景。下路是RemoteLookup在RestInterface和BoxOffice的作用，它主要负责传递message给RestInterface：
![RestInterface actor](/imgs/akka/akka_6_remotelooup_actor.png)

RemoteLookup actor是一个只包含两种状态的状态机：identify或者active。它使用become方法在这两种状态之间进行切换。当RemoteLookup想要获取一个BoxOffice的ActorRef的时候，如果它暂时还没有，就是在identify状态。不然它就是在active状态，传递所有的message到一个有效的BoxOffice ActorRef。

如果RemoteLookup发现BoxOffice已经被终结了，他会在接收不到message的一段时间后，重新获取一个有效的ActorRef。我们使用RemoteDeath监控这种情况。这东西我们并不陌生，跟普通的actor监控是一样的：
```scala
class RemoteLookup(path:String)
  extends Actor with ActorLogging {

  // 若3秒内没有收到任何message，就发送一个ReceiveTimeout消息 
  context.setReceiveTimeout(3 seconds)
  // 马上请求actor的identify
  sendIdentifyRequest()

  def sendIdentifyRequest(): Unit = {
    // 使用path获取actor
    val selection = context.actorSelection(path)
    // 向actorSelection发送一个Identify message
    selection ! Identify(path)
  }

  def receive = identify

  def identify:Receive = {
    case ActorIdentity(`path`, Some(actor)) => // 某个actor已经被确认(identified)了，返回它的ActorRef
      context.setReceiveTimeout(Duration.Undefined) // 既然当前actor已经是active了，就不用发送ReceiveTimeout了
      log.info("switching to active state")
      context.become(active(actor))	// 修改状态
      context.watch(actor)		// 监视

    case ActorIdentity(`path`, None) =>   // actor还不可用，后端连不上，或者还没有启动
      log.error(s"Remote actor with path $path is not available.")

    case ReceiveTimeout => // 如果还没有收到message，就继续identify远程actor
      sendIdentifyRequest()

    case msg:Any =>   	// 在identify接收状态下，不发送任何message
      log.error(s"Ignoring message $msg, remote actor is not ready yet.")
  }

  def active(actor: ActorRef): Receive = {
    case Terminated(actorRef) =>  	// 如果远程的actor被终结，RemoteLookup应修改它的状态到identify接收状态
      log.info("Actor $actorRef terminated.")
      log.info("switching to identify state")
      context.become(identify)
      context.setReceiveTimeout(3 seconds)
      sendIdentifyRequest()

    case msg:Any => actor forward msg		// 当远程actor是active的时候，向下传递所有其他message
  }
}
```

发送Identify message到ActorSelection来获取boxOffice的ActorRef。后端ActorSystem的远程module对应返回一个包含远程actor的ActorRef的 ActorIdentity message。

这样就完成了从一个单节点应用改到前后端node。除了能够远程通信，前端和后端还可以分开启动，前端会去找boxoffice，当boxoffice可用的时候就与之通信，不可用就采取其他措施。

最后就是运行FrontendMain和BackendMain。启动两个终端，分别使用sbt run指定入口类运行：
```
 [info] ...
 [info] ... (sbt messages)
 [info] ...
 Multiple main classes detected, select one to run:
 [1] com.goticks.SingleNodeMain
 [2] com.goticks.FrontendMain
 [3] com.goticks.BackendMain
 Enter number:
```

测试一下如果kill掉后端，再重启。

远程event的生命周期默认是打在日志里的，可以帮主我们更快定位问题。我们还可以使用actorSystem的eventStream方法定于远程生命周期event，后面第十章会提到。考虑到连接管理，再考虑到我们可以像本地actor一样监视远程actor，我们就没必要再针对event做什么额外操作了。

重新看一下修改的地方：
- 抽出 BoxOfficeCreator trait。远程版本中向后端请求查找Boxoffice。
- RemoteBoxOfficeCreator在RestInterface和BoxOffice之间添加了一个RemoteLookup actor。它负责传递他收到的所有的消息给boxoffice。它还identifyboxoffice的actorRef，并且远程监视boxoffice的actor。


#### 6.2.4 远程发布
前面说到akka提供了两种方式获取远程actor的ActorRef。下面我们介绍第二种，远程发布。

远程发布可以通过编程或配置实现。我们先弄一下配置方式的远程发布。它的好处就是不需要重新构建app就可以修改集群。标准的 BoxOfficeCreator trait 创建boxOffice，并把后者当做自己的一个子actor：
```scala
val boxOffice = context.actorOf(Props[BoxOffice], "boxOffice")
```

local path是/restInterface/boxOffice，注意这里没有/user。当我们使用基于配置的远程发布的时候，我们只需要告诉前端actorSystem，创建/restInterface/boxOffice的actor的时候，应该是远程创建。通过如下配置实现：
```
  actor {
    provider = "akka.remote.RemoteActorRefProvider"

    deployment {
      /restInterface/boxOffice {  // 使用这个path的actor都应该被远程发布
        remote = "akka.tcp://backend@0.0.0.0:2552"   // 远程发布的目标地址，应该就是后端监听的端口
      }

    }
  }
```


基于编程的远程发布相比起来简直就没有存在的必要。大多数情况下我们都使用基于配置的远程发布。但是有时，假如你要通过CNAMES引用不同的node，写代码就方便些。当时用akka-cluster的时候，完全动态远程发布就很有意义了，因为它可以支持动态membership。下面是一个基于编程的远程发布实例：
```scala
val uri = "akka.tcp://backend@0.0.0.0:2552"
 val backendAddress = AddressFromURIString(uri) // 从uri创建一个后端地址
 // 使用远程发布范围创建一个Props
 val props = Props[BoxOffice].withDeploy(
   Deploy(scope = RemoteScope(backendAddress))
 )
 context.actorOf(props, "boxOffice")
```

上面代码也是在后端远程创建与发布了一个boxOffice。

注意，远程发布并不是akka把实际的BoxOffice的class文件传送给远程server实现，而是远程server本身就有这个actor。如果后端远程ActorSystem挂掉并重启，这个ActorRef并不会自动关联到新的远程actor实例。这个实例是通过远程发布的，所以后端ActorSystem中不能是已经启动了的。基于这两点改变，下面是新的Main方法:
```scala
// 前端不再创建boxOffice
object BackendRemoteDeployMain extends App {

  val config = ConfigFactory.load("backend")
  val system = ActorSystem("backend", config)

}

// 后端不再mixin一些trait，使用默认的BoxOfficeCreator
object FrontendRemoteDeployWatchMain extends App {
  val config = ConfigFactory.load("frontend-remote-deploy")

  val host = config.getString("http.host")
  val port = config.getInt("http.port")

  val system = ActorSystem("frontend", config)

  class RestInterfaceWatch extends RestInterface
                           with ConfiguredRemoteBoxOfficeDeployment

  val restInterface = system.actorOf(Props[RestInterfaceWatch],
  "restInterface")

  Http(system).manager ! Bind(listener = restInterface,
  interface = host,
  port =port)

}


```


分别在两个终端下运行这两个main方法，尝试使用httpie创建event，会看到下面的message在前端ActorSystem打印出来：
```
// very long message, formatted in a couple of lines to fit.
INFO [RestInterface]: Received new event Event(RHCP,10), sending to
Actor[akka.tcp://backend@0.0.0.0:2552/remote/akka.tcp/
 frontend@0.0.0.0:2551/user/restInterface/boxOffice#-1230704641]
```

这是说前端向后端发送了一个message来远程发布了boxOffice。

目前来说一切都顺利。可是有个问题：如果前端尝试远程发布的时候后端没有启动会咋样，因为还是会获得一个ActorRef。即便后端ActorSystem一会儿启动了，这个ActorRef还是不可用。要解决这个问题，我们还是要对这个ActorRef添加监视。我们需要像前面RemoteLookup actor那样放一个actor在RestInterface和boxOffice之间，这次是RemoteBoxOfficeForwarder。

我们需要稍微修改以下配置文件：
```
  actor {
    provider = "akka.remote.RemoteActorRefProvider"

    deployment {
      /restInterface/forwarder/boxOffice {
        remote = "akka.tcp://backend@0.0.0.0:2552"
      }

    }
  }
```

下面是ConfiguredRemoteBoxOfficeDeployment和RemoteBoxOfficeForwarder的代码：
```scala
trait ConfiguredRemoteBoxOfficeDeployment extends BoxOfficeCreator { this:Actor =>

  // 创建一个forwarder来负责监视和发布远程的BoxOffice
  override def createBoxOffice = {
    context.actorOf(Props[RemoteBoxOfficeForwarder],
                           "forwarder")
  }
}



class RemoteBoxOfficeForwarder extends Actor with ActorLogging {
  context.setReceiveTimeout(3 seconds)

  // 远程发布并监视BoxOffice
  deployAndWatch()

  def deployAndWatch(): Unit = {
    val actor = context.actorOf(Props[BoxOffice], "boxOffice")
    context.watch(actor)
    log.info("switching to maybe active state")
    context.become(maybeActive(actor))
    context.setReceiveTimeout(Duration.Undefined)
  }

  def receive = deploying

  def deploying:Receive = {

    case ReceiveTimeout =>
      deployAndWatch()

    case msg:Any =>
      log.error(s"Ignoring message $msg, remote actor is not ready yet.")
  }

  def maybeActive(actor:ActorRef): Receive = {
    case Terminated(actorRef) => 	// 远程发布的actor被终结，那就重新远程发布一个
      log.info("Actor $actorRef terminated.")
      log.info("switching to deploying state")
      context.become(deploying)
      context.setReceiveTimeout(3 seconds)
      deployAndWatch()

    case msg:Any => actor forward msg
  }
}
```


RemoteBoxOfficeForwarder和上文的RemoteLookup特别像，都有一个状态机，不过它是 'deploying' 或  'maybe active'。没有actor selection，我们不能确认远程actor是否已经真的被发布了。 为RemoteBoxOfficeForwarder添加远程actorSelection的任务就交给读者你了，就是当前'maybe'的状态机。


前端的main类mixin了ConfiguredRemoteBoxOfficeDeployment。看一下FrontendRemoteDeployWatchMain：
```scala
  class RestInterfaceWatch extends RestInterface
                           with ConfiguredRemoteBoxOfficeDeployment

  val restInterface = system.actorOf(Props[RestInterfaceWatch],
  "restInterface")
```

重新启动前后端的程序，测试一把。


我们把lookup和远程发布两种方式都可以使程序特别有韧性。即便在只有两个node的时候，我们最好也能在最开始就弄的特别有韧性。

#### 6.2.5 Multi-JVM testing

sbt multi-jvm插件可以用来进行跨JVM的测试，这对于分布式系统是很重要的。需要在sbt配置文件中添加：
```
addSbtPlugin("com.typesafe.sbt" % "sbt-multi-jvm" % "0.3.5")
```

我们需要额外添加另一个sbt构件文件来使用它。 Multi-JVM插件只支持scala DSL版本的sbt下过目文件，是；uoyi我们需要在chapter6/project文件夹下添加GoTicksBuild.scala文件。Sbt会自动merge build.sbt和GoTicksBuild.scala。
```scala
import sbt._
import Keys._
import com.typesafe.sbt.SbtMultiJvm
import com.typesafe.sbt.SbtMultiJvm.MultiJvmKeys.{ MultiJvm }

object GoTicksBuild extends Build {

  lazy val buildSettings = Defaults.defaultSettings ++ multiJvmSettings ++ Seq(
    crossPaths   := false
  )

  lazy val goticks = Project(
    id = "goticks",
    base = file("."),
    settings = buildSettings ++ Project.defaultSettings
  ) configs(MultiJvm)

  lazy val multiJvmSettings = SbtMultiJvm.multiJvmSettings ++ Seq(
    // make sure that MultiJvm test are compiled by the default test compilation
    compile in MultiJvm <<= (compile in MultiJvm) triggeredBy (compile in Test),
    // disable parallel tests
    parallelExecution in Test := false,
    // make sure that MultiJvm tests are executed by the default test target
    executeTests in Test <<=
      ((executeTests in Test), (executeTests in MultiJvm)) map {
        case ((_, testResults), (_, multiJvmResults))  =>
          val results = testResults ++ multiJvmResults
          (Tests.overall(results.values), results)
      }
  )
}
```

如果你对sbt不熟，不用担心这个文件的细节。上面基本就是配置了multi-jvm插件，确认multi-jvm测试使用普通单元测试执行。要是想看看sbt的话，可以参考SBT in Action。

多jvm的测试默认是应该写在 src/multi-jvm/scala 文件夹下。既然我们的项目已经正确配置了，那么我们可以开始写代码测试goticks的前端和后端了。

首先定义MultiNodeConfig描述测试的node的角色。
```scala
object ClientServerConfig extends MultiNodeConfig {
  val frontend = role("frontend")
  val backend = role("backend")
}

```
定义了两个role。role用来确认参与测试的node运行的代码，以便于针对性的测试。在写测试之前我们需要些一些基础设施代码：

```scala
trait STMultiNodeSpec extends MultiNodeSpecCallbacks
with WordSpec with MustMatchers with BeforeAndAfterAll {

  override def beforeAll() = multiNodeSpecBeforeAll()

  override def afterAll() = multiNodeSpecAfterAll()
}
 
```

其中继承MultiNodeSpecCallbacks以获取回调函数，然后下面重写了几个回调函数。

下面我们创建MultiNodeSpec，并让他mixin刚刚定义的STMultiNodeSpec。两个版本的ClientServerSpec会运行在不同的两个jvm上。下面是ClientServerSpec：
```scala
class ClientServerSpecMultiJvmFrontend extends ClientServerSpec
class ClientServerSpecMultiJvmBackend extends ClientServerSpec

class ClientServerSpec extends MultiNodeSpec(ClientServerConfig)
with STMultiNodeSpec with ImplicitSender {

  // 参加到这次测试的node个数
  def initialParticipants = roles.size
```

ClientServerSpec使用了STMultiNodeSpec和ImplicitSender。后者设置testActor为所有message的默认sender，这样不用每次都设置了。
```scala
  // import后我们才可以访问到后端role
  import ClientServerConfig._

  // 测试中使用TestRemoteBoxOfficeCreator，而不是RemoteBoxOfficeCreator
  trait TestRemoteBoxOfficeCreator extends RemoteBoxOfficeCreator { this:Actor =>

    // 重写方法，返回测试的path
    override def createPath: String = {
      val actorPath = node(backend) / "user" /"boxOffice"		//node方法返回后端role node的地址。这个表达式是创建了一个ActorPath
      actorPath.toString
    }
  }
```
前端和后端role node默认是在随机端口上运行。我们使用TestRemoteBoxOfficeCreator替代RemoteBoxOfficeCreator，因为后者是使用的frontend.conf文件中的配置。我们在测试中需要找的是不一样的环境。

看一下测试代码：
```scala
"A Client Server configured app" must {

    "wait for all nodes to enter a barrier" in {
      enterBarrier("startup") // 启动所有的node
    }

    "be able to create an event and sell a ticket" in {

      // 运行在前端jvm的代码
      runOn(frontend) {
 	// 等待后端发布
        enterBarrier("deployed")
        // 创建一个RestInterfaceMock
	val restInterface = system.actorOf(Props(new RestInterfaceMock with TestRemoteBoxOfficeCreator))

        val path = node(backend) / "user" / "boxOffice"
        
	// 获取远程actor的actorSelection
	val actorSelection = system.actorSelection(path)

 	// 向actorselection发送Identify消息
        actorSelection.tell(Identify(path), testActor)

	// 等待boxoffice汇报它是否可用。RemoteLookup类会负责获取boxoffice的ActorRef
        val actorRef = expectMsgPF() {
          case ActorIdentity(`path`, ref) => ref
        }

        restInterface ! Event("RHCP", 1)

        expectMsg(EventCreated)

        restInterface ! TicketRequest("RHCP")

        expectMsg(Ticket("RHCP", 1))
      }

      // 运行在后端JVM上的代码
      runOn(backend) {
	// 创建boxOffice，并命名为boxOffice，这样RemoteLookup就可以找到它了
        system.actorOf(Props[BoxOffice], "boxOffice")
	// 发送后端已经部署的信号
        enterBarrier("deployed")
      }

      // 测试结束
      enterBarrier("finished")
    }
  }
```

这段测试可以分为4个部分。第一个是enterBarrier("startup")等待所有node启动。实际的测试会面就会指定前端和后端各自运行什么代码。前端会等后端启动成功的信号，然后执行测试。

后段代码只启动boxoffice，提供给前端使用。我们使用了RestInterfaceMock，TestRemoteBoxOfficeCreator。我们还是使用了RemoteLookup来等远程的actor被identify。 在发送message到远程boxoffice之前，我们先期望获得一个ActorIdentify message。

在项目目录下执行multi-jvm:test进行测试。

下图说明了测试的实际流程。
![Multi-JVM Test Flow](/imgs/akka/akka_6_multi_jvm_test_flow.png)

单节点运行和client server 模式的最大区别就是actorRef的获取与监视。在RestInterface和boxOffice 之间添加一个Remote Lookup，获取更大的灵活性，也能够更好地处理actor挂掉的问题。我们怎样等待远程actorRef可用呢？actorSelection和Identify message机制可以搞定。

6.3 总结
---
记得本章开始的时候我们要分布式的理由么？

除了我们要做一些改变之外，还有一些不变的事情：
 - 我们使用ActorRef，不用关心它是远程actor还是本地actor
 - 监控actor的API和local模式一样
 - 除了协作者被网络分隔之外，我们通过中间actor，让RestInterface和BoxOffice互相通信。

我们学到的新东西：
 - REPL提供了一个简单友好的方式来探索分布式拓扑
 - multi-node-testkit使得分布式系统的测试特别简单，不管这些分布式系统是基于akka-remote还是 akka-cluster

我们还没有处理在RemoteLookup和RemoteBoxOfficeForwarder的message丢失问题。下面的章节会介绍：
- 怎样在合作node之间创建爱你可信赖的proxy
- 当后端node挂掉的时候处理TicketSellers丢失的问题
- 状态怎样在集群间复制

但是。。在此之前，我们先看一下基于akka-cluster module的动态的node membership。
