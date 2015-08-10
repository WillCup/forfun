title: akka in action 读书笔记（2）
date: 2015-08-10 02:45:52
tags: [akka]
---




### clone

> git clone https://github.com/RayRoestenburg/akka-in-action.git

先切换到ch02分支上【这个项目的分支有点儿乱 】，找到目录里的chapter2目录，引入idea
![1 引入IDE](/imgs/akka/2.1 引入IDE.png)

引入过程中IDE应该会自动下载sbt依赖，如果没有的话，到项目根目录下执行：
> sbt assembly

此项目使用的依赖中除了Spray其他都是scala栈中的成员。当然后面可能会引入更多的其他依赖，比如根据不同的环境进行不同参数的调整。

使用下面命令测试是否构建成功：

> java -jar target/scala-2.10/goticks-server.jar

```
Slf4jEventHandler started
akka://com-goticks-Main/user/io-bridge started
akka://com-goticks-Main/user/http-server started on /0.0.0.0:5000
```

### sbt
sbt的配置文件就是项目根目录下的build.sbt文件。

```
import AssemblyKeys._
import com.typesafe.sbt.SbtStartScript

// 有两个jar需要用到上面两个import。一个类似maven里的shade，将项目打成一个大jar；另一个则是用来部署项目到Heroku上。

name := "goticks"

version := "0.1-SNAPSHOT"

organization := "com.goticks"

scalaVersion := "2.11.2"

resolvers ++= Seq("Typesafe Repository" at "http://repo.typesafe.com/typesafe/releases/",
                  "Spray Repository"    at "http://repo.spray.io")
// 指定repositories

libraryDependencies ++= {
  val akkaVersion       = "2.3.9"
  val sprayVersion      = "1.3.2"
  Seq(
    "com.typesafe.akka" %% "akka-actor"      % akkaVersion,
    "io.spray"          %% "spray-can"       % sprayVersion,
    "io.spray"          %% "spray-routing"   % sprayVersion,
    "io.spray"          %% "spray-json"      % "1.3.1",
    "com.typesafe.akka" %% "akka-slf4j"      % akkaVersion,
    "ch.qos.logback"    %  "logback-classic" % "1.1.2",
    "com.typesafe.akka" %% "akka-testkit"    % akkaVersion   % "test",
    "org.scalatest"     %% "scalatest"       % "2.2.0"       % "test"
  )
}

// Assembly settings
mainClass in Global := Some("com.goticks.Main")

jarName in assembly := "goticks-server.jar"

assemblySettings

// StartScript settings
seq(SbtStartScript.startScriptForClassesSettings: _*)

```

### REST server

到根目录下启动server
> sbt run

![2 REST规范](/imgs/akka/2.2 REST规范.png)
存取[为方便测试，请事先安装Linux工具httpie]:

```
root@ubuntu:~/gitLearning/akka-in-action/chapter2# http PUT localhost:5000/events event=RHCP nrOfTickets:=10
HTTP/1.1 200 OK
Content-Length: 2
Content-Type: text/plain; charset=UTF-8
Date: Mon, 10 Aug 2015 06:28:08 GMT
Server: GoTicks.com REST API

OK

root@ubuntu:~/gitLearning/akka-in-action/chapter2# http PUT localhost:5000/events event=DjMadlib nrOfTickets:=15
HTTP/1.1 200 OK
Content-Length: 2
Content-Type: text/plain; charset=UTF-8
Date: Mon, 10 Aug 2015 06:28:45 GMT
Server: GoTicks.com REST API

OK

root@ubuntu:~/gitLearning/akka-in-action/chapter2# http GET localhost:5000/events
HTTP/1.1 200 OK
Content-Length: 92
Content-Type: application/json; charset=UTF-8
Date: Mon, 10 Aug 2015 06:28:52 GMT
Server: GoTicks.com REST API

[
    {
        "event": "DjMadlib", 
        "nrOfTickets": 15
    }, 
    {
        "event": "RHCP", 
        "nrOfTickets": 10
    }
]

root@ubuntu:~/gitLearning/akka-in-action/chapter2# http GET localhost:5000/ticket/RHCP
HTTP/1.1 200 OK
Content-Length: 32
Content-Type: application/json; charset=UTF-8
Date: Mon, 10 Aug 2015 06:29:32 GMT
Server: GoTicks.com REST API

{
    "event": "RHCP", 
    "nr": 1
}

root@ubuntu:~/gitLearning/akka-in-action/chapter2# http GET localhost:5000/events
HTTP/1.1 200 OK
Content-Length: 91
Content-Type: application/json; charset=UTF-8
Date: Mon, 10 Aug 2015 06:29:46 GMT
Server: GoTicks.com REST API

[
    {
        "event": "DjMadlib", 
        "nrOfTickets": 15
    }, 
    {
        "event": "RHCP", 
        "nrOfTickets": 9
    }
]

```
以上已经包含了CRUD操作，当然这并不全面。我们没有考虑如果一个Event被卖出去了，那么它的状态应该马上变为不可卖，后面我们会再详细谈论此类问题。


### Actors

你可以尝试跟着github上搞下来的源码自己一步步构建项目试试。现在我们已经知道了，actor可以有四种操作：create, send\receive, become, supervise。这里我们会接触到前两个。
先看看actor之间怎样相互作用传递信息: 创建event、改票、完成event。

![3. Actor Creation Sequence Triggered by REST Request](/imgs/akka/2.3 Actor Creation Sequence Triggered by REST Request.png)

RestInterface Actor负责处理HTTP request，其实就是一个HTTP请求的adapter。TicketSeller则负责一些tickets，所有关于票务买卖的event它都管。下图解释了一个request怎样通过actor system创建一个event：

![4 Create an Event from the received JSON request](/imgs/akka/2.4 Create an Event from the received JSON request.png)
注意：我看的版本里不是TicketMaster而是BoxOffice

![5 Buy a Ticket](/imgs/akka/2.5 Buy a Ticket.png)

以上就是这个小APP里主要的业务流了，主要就是让customer能够跟可用的票之间能够建立连接，进行交互。

下面让我们退回去看一下代码，首先是入口Main类

```
package com.goticks

import scala.concurrent.duration._

import akka.actor._
import akka.io.IO
import akka.pattern.ask
import akka.util.Timeout

import spray.can.Http
import spray.can.Http.Bound

import com.typesafe.config.ConfigFactory

object Main extends App {
  val config = ConfigFactory.load() 	// 加载配置文件
  val host = config.getString("http.host")
  val port = config.getInt("http.port")

  implicit val system = ActorSystem("goticks")

  val api = system.actorOf(Props(new RestInterface()), "httpInterface") // 创建最高层的actor
 
  implicit val executionContext = system.dispatcher
  implicit val timeout = Timeout(10 seconds)

  // 启动Spray HTTP Server
  IO(Http).ask(Http.Bind(listener = api, interface = host, port = port))
    .mapTo[Http.Event]
    .map {
      case Http.Bound(address) =>
        println(s"REST interface bound to $address")
      case Http.CommandFailed(cmd) =>
        println("REST interface could not bind to " +
          s"$host:$port, ${cmd.failureMessage}")
        system.shutdown()
    }
}

```

这个App内部沟通的消息类型都定义在TicketProtocol中：
```java
object TicketProtocol {
  import spray.json._

  case class Event(event:String, nrOfTickets:Int)

  case object GetEvents

  case class Events(events:List[Event])

  case object EventCreated

  case class TicketRequest(event:String)

  case object SoldOut

  case class Tickets(tickets:List[Ticket])

  case object BuyTicket

  case class Ticket(event:String, nr:Int)

  //----------------------------------------------
  // JSON
  //----------------------------------------------

  object Event extends DefaultJsonProtocol {
    implicit val format = jsonFormat2(Event.apply)
  }

  object TicketRequest extends DefaultJsonProtocol {
    implicit val format = jsonFormat1(TicketRequest.apply)
  }

  object Ticket extends DefaultJsonProtocol {
    implicit val format = jsonFormat2(Ticket.apply)
  }

}
```
作为一个典型的APP，我们有一个接口来解决围绕Event主体和Ticket整个生命周期的所有问题。记住，Akka会使用immutable message将这些部分链接起来，所以actor应该知道它需要的所有的信息。

#### TicketSeller

TicketSeller是由BoxOffice创建的, 主要负责处理订单的，自己维护了一些ticket。每次有买票的请求，它就从自己的ticket list里拿出第一张来：
```
class TicketSeller extends Actor {
  import TicketProtocol._

  var tickets = Vector[Ticket]()	// 票

  def receive = {

    case GetEvents => sender ! tickets.size	// 返回当前票数

    case Tickets(newTickets) => tickets = tickets ++ newTickets //添加新票

    case BuyTicket =>	// 有请求买票
      if (tickets.isEmpty) {
        sender ! SoldOut
        self ! PoisonPill	// 如果卖光了就服毒自杀，这会导致当前actor被清理
      }

      tickets.headOption.foreach { ticket =>
        tickets = tickets.tail
        sender ! ticket
      }
  }
}

```

#### BoxOffice
BoxOffice 需要为每个event都创建TicketSeller，并把买票请求代理给他们。

```
class BoxOffice extends Actor with CreateTicketSellers with ActorLogging {
  import TicketProtocol._
  import context._
  implicit val timeout = Timeout(5 seconds)

  def receive = {

    case Event(name, nrOfTickets) =>
      log.info(s"Creating new event ${name} with ${nrOfTickets} tickets.")

      if(context.child(name).isEmpty) {	// 如果这个name的TicketSeller还没有的话
        val ticketSeller = createTicketSeller(name)

        // 塞进初始数量的票
        val tickets = Tickets((1 to nrOfTickets).map(nr=> Ticket(name, nr)).toList)
        ticketSeller ! tickets
      }

      sender ! EventCreated

    case TicketRequest(name) =>
      log.info(s"Getting a ticket for the ${name} event.")

      // 将BuyTicket message传递给TicketSeller
      context.child(name) match {
        case Some(ticketSeller) => ticketSeller.forward(BuyTicket)
        case None               => sender ! SoldOut // 如果找不到对应的ticketSeller，就返回SoldOut message给sender，也就是RestInterface
      }



    // 向所有的ticketSeller收集当前票数，然后弄成一个list。ask是一个异步操作，因为我们不希望BoxOffice阻塞，而停止处理其他request。
    case GetEvents =>
      import akka.pattern.ask

      val capturedSender = sender

      // 向TicketSeller传递GetEvents message的本地方法
      def askAndMapToEvent(ticketSeller:ActorRef) =  {
      	// 不阻塞的获取每个name下的票数，futureInt会在执行完后获取到应得的value
        val futureInt = ticketSeller.ask(GetEvents).mapTo[Int]

        futureInt.map(nrOfTickets =>
          // 将future的值转化为Event
          Event(ticketSeller.actorRef.path.name, nrOfTickets))
      }

  	  // 遍历所有的ticketSeller
      val futures = context.children.map(ticketSeller =>
        askAndMapToEvent(ticketSeller))

      // 所有future都得到值后，发送信息。
      Future.sequence(futures).map { events =>
        capturedSender ! Events(events.toList)
      }

  }

}

// 注意是使用context创建的actor：谁创建的，谁就是这个新actor的supervisor
trait CreateTicketSellers { self:Actor =>
  def createTicketSeller(name:String) =  context.actorOf(Props[TicketSeller], name)
}
```

上例中我们使用ask返回的是一个Future对象，这个东西是在未来某个时间点才会有值的。获取一个future引用，定义当这个值**可用后**要做的操作，而不是阻塞等待执行完毕，也不要直接使用这个future。第七章我们会深入探索Future的特性，弄清楚这些非阻塞的异步请求的运作原理。

#### RestInterface

RestInterface使用了Spray routing DSL，后面第九章会详解：当service越来越大，复杂的路由也越来越多，我们这个例子里倒是很少，因为只是卖票而已。

```scala
package com.goticks

import akka.actor._

import spray.routing._
import spray.http.StatusCodes
import spray.httpx.SprayJsonSupport._
import spray.routing.RequestContext
import akka.util.Timeout
import scala.concurrent.duration._
import scala.language.postfixOps

class RestInterface extends HttpServiceActor
  with RestApi {

  def receive = runRoute(routes)
}

trait RestApi extends HttpService with ActorLogging { actor: Actor =>
  import context.dispatcher
  import com.goticks.TicketProtocol._

  implicit val timeout = Timeout(10 seconds)
  import akka.pattern.ask
  import akka.pattern.pipe

  // 初始化就创建一个BoxOffice
  val boxOffice = context.actorOf(Props[BoxOffice])

  def routes: Route =

    path("events") {
      put {
        entity(as[Event]) { event => requestContext =>
          val responder = createResponder(requestContext)
          boxOffice.ask(event).pipeTo(responder)
        }
      } ~
      get { requestContext =>
        val responder = createResponder(requestContext)
        boxOffice.ask(GetEvents).pipeTo(responder)
      }
    } ~
    path("ticket") {
      /**
      entity方法把request直接解析成一个TicketRequest对象，不用我们自己再去编写代码来映射到对象。
      response被pipe到一个使用pipe模型的responder。
      */
      get {
        entity(as[TicketRequest]) { ticketRequest => requestContext =>
          val responder = createResponder(requestContext)
          boxOffice.ask(ticketRequest).pipeTo(responder)
        }
      }
    } ~
    path("ticket" / Segment) { eventName => requestContext =>
      val req = TicketRequest(eventName)
      val responder = createResponder(requestContext)
      boxOffice.ask(req).pipeTo(responder)
    }
  def createResponder(requestContext:RequestContext) = {
    context.actorOf(Props(new Responder(requestContext)))
  }

}

// 围绕一个HTTP request生命周期的Responder，发送消息给BoxOffice，同时也处理来自TicketSeller和BoxOffice的response。
class Responder(requestContext:RequestContext) extends Actor with ActorLogging {
  import TicketProtocol._
  import spray.httpx.SprayJsonSupport._

  // responder接收到TicketSeller返回的以下四类消息后就算完成了HTTP request。之后自杀。
  def receive = {

    case ticket:Ticket =>
      requestContext.complete(StatusCodes.OK, ticket)
      self ! PoisonPill

    case EventCreated =>
      requestContext.complete(StatusCodes.OK)
      self ! PoisonPill

    case SoldOut =>
      requestContext.complete(StatusCodes.NotFound)
      self ! PoisonPill

    case Events(events) =>
      requestContext.complete(StatusCodes.OK, events)
      self ! PoisonPill
  }
}

```


