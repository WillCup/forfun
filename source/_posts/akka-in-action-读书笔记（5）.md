title: akka in action 读书笔记（5）
date: 2016-08-11 16:35:33
tags: [akka]
---

这一章聊一下akka的Future。Actor提供了一个机制替代了并发object系统，Future提供了一个机制来替代并发function系统。

5.1 Futrue用例
---
Actor被用来处理message，获取state，基于收到的不同message执行不同代码。借助于监管和监控，他们成为生命力极强的对象，即便出现错误也可以继续生存。

Future是用来不用object，而使用function就能解决问题的工具。它是一个function结果的占位符，在未来的某一个时刻生效，是一个异步结果。

![异步函数结果的占位符](/imgs/akka/akka_5_placeholder_for_async.png)

Future是只读的，而且可以被读多次，不能被外界修改。

注意：Future并不是对java.util.concurrent.Futrue的封装。后者的get方法还是会引起block的。

Future对于pipeline的处理方式很友好，就是经过function1的结果传递给function2使用。

以买票的例子来说，假设我们创建了一个web页面，提供一些额外的信息，比如天气预报、交通状况、出行计划、停车等。下面是event的处理流程：
![异步函数链](/imgs/akka/akka_5_chain_async_func.png)

图中getEvent和getTraffic都是异步请求web服务的function，顺序执行。getTrafficInfo接收一个Event参数，当Future[Event]获得结果后就会触发getTrafficInfo的执行。这与在并发线程中执行getEvent方法等待返回结果是不同的。我们定义了一个一定会被执行的getTrafficInfo函数，而且不用任何等待。

下图展示了一个简单的例子：
![同步与异步的对比](/imgs/akka/akka_5_aggr_result_sync_vs_async.png)

可见异步的话只需要花费4和2中的最长时间，而同步就需要2+4。

下图展示了另一个用例，利用异步最快获取天气信息：

![异步最快获取天气信息](/imgs/akka/akka_5_weather_info_async.png)

假设X服务出现超时问题的话，就直接不用管它的结果了。

这些场景使用actor并不是不可以，只是直接使用actor的话会特别麻烦。假设使用actor实现上面那个获取天气和交通信息的功能，需要包含以下步骤：

![使用actor组合web请求](/imgs/akka/akka_5_combine_web_request_actors.png)

我们需要为天气和交通服务分别创建两个actor，这样他们才能被同时调用。调用方式被定义在TicketInfo Actor中。这里也并不能实现非阻塞。

如果应用包含以下特性，就可以考虑使用future：
- 不要block
- 调用一次某个function之后，在未来的某个时间点使用调用结果
- 组合许多一次调用的function结果
- 调用一些相互竞争的function，只使用其中的一部分结果，比如：最快的天气信息
- 当function出现错误的时候，需要返回默认值，以便后面的function能够继续调用
- pipeline对结果相互依赖的function

5.2 Futrue中无阻塞
---

是时候写一下TicketInfoService完成上面获取天气和交通信息了。

异步与同步的方法在与程序中的处理流，下面是一个同步的程序：
![同步调用代码](/imgs/akka/akka_5_sync_call_code.png)

这里既没有使用futrue，也没有使用scala的懒加载，所以每行都需要等返回值才能继续。

再看一下异步的程序：
![异步调用代码](/imgs/akka/akka_5_async_chain_call_code.png)

callEventService实际上是在另一个线程里调用的，这个线程其实是阻塞的，后面我们会解决这个问题。futrue是将一段代码进行一部执行调用的Futrue.apply的简写。

以上代码其实是一个闭包，把request从main线程传递给一个新的线程去执行某个方法。

下面把交通信息串进来：
![处理异步结果，继续异步调用](/imgs/akka/akka_5_async_chain_call_code.png)

这段代码块只有在futrueEvent可用的时候才会执行。如果我们也需要这段程序有返回值咋办

![返回Future[Route]](/imgs/akka/akka_5_async_chain_call_return_code.png)

也可以直接串起来。

![直接串起来](/imgs/akka/akka_5_async_chain_all_code.png)

重构一下getEvent和getRoute方法：
![重构之后](/imgs/akka/akka_5_refactor.png)

其实要真正的异步的话，callEventService和callTrafficService应该也是异步返回Futrue才对，这个可以借助spray-client，在后面的rest相关章节我们会介绍。

需要注意一个细节，就是钥匙用futrue的话需要引入对ExecutionContext的隐式转换，不然编译不过去。
```scala
import scala.concurrent.Implicits.global 
```
ExecutionContext是在某个线程池执行任务的抽象，可以类比java.util.concurrent.Executor。

这里介绍了成功调用的案例，下面说一下怎样从失败中恢复。

5.3 失败
---

假如在异步处理的过程中出现了Exception咋办？启动一个scala REPL，执行以下代码
```
 scala> :paste
 // Entering paste mode (ctrl-D to finish)

 import scala.concurrent._
 import ExecutionContext.Implicits.global

 val futureFail = future { throw new Exception("error!")}
 futureFail.foreach(value=> println(value))

 // Exiting paste mode, now interpreting.

 futureFail: scala.concurrent.Future[Nothing] =
 scala.concurrent.impl.Promise$DefaultPromise@193cd8e1

 scala> 
```

可以看到并没有打印我们期望的结果，命令行也没有打印方法栈。由于前面没有执行成功，所以foreach代码没有执行。

我们可以使用OnComplete接口
```scala
 scala> :paste
 // Entering paste mode (ctrl-D to finish)

 import scala.util._
 import scala.util.control.NonFatal
 import scala.concurrent._
 import ExecutionContext.Implicits.global

 val futureFail = future { throw new Exception("error!")}
 futureFail.onComplete {
 case Success(value) => println(value)
 case Failure(NonFatal(e)) => println(e)
 }

 // Exiting paste mode, now interpreting.

 java.lang.Exception: error! 

```

OnComplete回调函数总是会被调用，而且返回值是Unit。

```scala
 futureFail.onFailure {
 case NonFatal(e) => println(e)
 }
```

TicketInfo服务同样也应该保持一些信息，以便出现异常后记录抛出的异常信息。下图是TicketInfo里保存的事件信息：
![TicketInfo保存的事件信息](/imgs/akka/akka_5_accumulate_info_tickit_info.png)

getEvent和getTraffic改用TicketInfo作为返回值与接收值，这样TicketInfo可以一路积攒信息，TicketInfo是一个case class。

注意：在使用futrue的过程中一定要全程使用不可变数据。

下图说明了GetTraffic调用失败的时候会发生什么：
![忽略失败的服务的response](/imgs/akka/akka_5_ignore_failed_resp.png)

我们可以定义recover方法，当出现异常的时候就返回这个方法的返回值。下面代码说明了怎样使用：
```scala
 val futureStep1 = getEvent(ticketNr)//返回Future[TicketInfo]
 val futureStep2 = futureStep1.flatMap { ticketInfo => // 使用flatMap我们可以直接返回Future[TicketInfo]而不是一个TicketInfo
   getTraffic(ticketInfo).recover { //返回Future[TicketInfo]
     case e:TrafficServiceException => ticketInfo //使用一个包含初始TicketInfo的Future来恢复
   }
 }
```

getTraffic啥的都是创建TicketInfo的copy。

还有一个recoverWith用来返回 Future[TicketInfo]而不是TicketInfo。注意上面的recover方法是同步调用的。

上面代码中还有一个问题，如果getEvent就失败了咋办？flatMap根本就不会执行。
```scala
 val futureStep1 = getEvent(ticketNr)
 val futureStep2 = futureStep1.flatMap { ticketInfo =>
     getTraffic(ticketInfo).recover {
       case e:TrafficServiceException => ticketInfo
     }
 }.recover {
   case NonFatal(e) => TicketInfo(ticketNr)
 }
```
如果getEvent失败的话，就直接返回包含ticketNr的空TicketInfo。

5.4 整合Futrue
---

下面是买票服务中的所有的case class。
```scala
package com.goticks

import org.joda.time.{Duration, DateTime}

case class TicketInfo(ticketNr:String,
                      userLocation:Location,
                      event:Option[Event]=None,
                      travelAdvice:Option[TravelAdvice]=None,
                      weather:Option[Weather]=None,
                      suggestions:Seq[Event]=Seq())

case class Event(name:String,location:Location, time:DateTime)

case class Artist(name:String, calendarUri:String)

case class Location(lat:Double, lon:Double)

// 简单起见，所有的route都弄成了一个简单的字符串
case class RouteByCar(route:String, timeToLeave:DateTime, origin:Location, destination:Location,
                 estimatedDuration:Duration, trafficJamTime:Duration)

case class PublicTransportAdvice(advice:String, timeToLeave:DateTime, origin:Location, destination:Location,
                                 estimatedDuration:Duration)

case class TravelAdvice(routeByCar:Option[RouteByCar]=None,
                        publicTransportAdvice: Option[PublicTransportAdvice]=None)

case class Weather(temperature:Int, precipitation:Boolean)
```

下图是TicketInfoService的处理流：
![TicketInfoService的处理流](/imgs/akka/akka_5_ticket_flow.png)

combinators在图中以菱形表示。所有的future最终都被整合进了Future[TicketInfo]。

我们以最快选取天气信息先开始讲解，下面是这块逻辑使用combinator的地方：
![天气信息流](/imgs/akka/akka_5_weather_flow.png)

两个天气服务都返回Future[Weather]，然后我们需要的是Future[TicketInfo]。下面是代码：
```scala
  def getWeather(ticketInfo:TicketInfo):Future[TicketInfo] = {

    val futureWeatherX = callWeatherXService(ticketInfo).recover(withNone) // 恢复代码封装在withNone里了

    val futureWeatherY = callWeatherYService(ticketInfo).recover(withNone)

    Future.firstCompletedOf(Seq(futureWeatherX, futureWeatherY)).map{ weatherResponse =>
      ticketInfo.copy(weather = weatherResponse)// 复制到ticketInfo里
    }
  }

```

firstCompletedOf只返回第一个返回结果的服务的response，成功失败无所谓，也就是说加入先返回的是失败，那么后面即便有成功返回也不会生效了。【可以考虑firstSucceededOf】

下面是公共交通信息和行车路由服务，它们应该并行，然后等两者都结束调用就将结果combine进TravelAdvice。下图展示了这个combinator：
![旅行路径建议流](/imgs/akka/akka_5_travel_flow.png)

getTraffic和getPublicTransport的Futrue各自返回一种类型数据：RouteByCar以及PublicTransportAdvice。这两个值先被放进一个tuple，然后把这个tuple map放进TravelAdvice里面。
```scala
case class TravelAdvice(routeByCar:Option[RouteByCar]=None,
                        publicTransportAdvice: Option[PublicTransportAdvice]=None)
```

基于这个信息，用户就可以决定是使用公共交通工具出行，还是自驾游。下面是zip combinator的代码：
```scala
  def getTravelAdvice(info:TicketInfo, event:Event):Future[TicketInfo] = {

    val futureRoute = callTrafficService(info.userLocation, event.location, event.time).recover(withNone)

    val futurePublicTransport = callPublicTransportService(info.userLocation, event.location, event.time).recover(withNone)

	// 先把 Future[RouteByCar] 和 Future[PublicTransportAdvice] zip成tuple放入Future[(RouteByCar, PublicTransportAdvice)]
	// 然后再将这两者map放入Future[TicketInfo]
    futureRoute.zip(futurePublicTransport).map { case(routeByCar, publicTransportAdvice) =>
      val travelAdvice = TravelAdvice(routeByCar, publicTransportAdvice)
      info.copy(travelAdvice = Some(travelAdvice))
    }
  }
```

也可以使用for替换map，有时可读性更强一些.
```scala
    for((routeByCar, publicTransportAdvice) <- futureRoute.zip(futurePublicTransport);
         travelAdvice = TravelAdvice(routeByCar, publicTransportAdvice)
    ) yield info.copy(travelAdvice = Some(travelAdvice))
```

下一个部分我们看一下推荐类似活动给用户。我们使用了两个web服务，一个是类似的艺术服务，返回用户感兴趣的类似艺术的信息。使用这个信息来调用一个日历服务，以便计划下一站的去向。
```scala
  // 返回对每种艺术活动的计划列表 Future[Seq[Events]]
  def getSuggestions(event:Event):Future[Seq[Event]] = {

    val futureArtists = callSimilarArtistsService(event).recover(withEmptySeq) // 返回类似艺术活动的列表：Future[Seq[Events]]

    for(artists <- futureArtists.recover(withEmptySeq); // 'artists' evaluates at some point to a Seq[Artist]
        events <- getPlannedEvents(event, artists).recover(withEmptySeq) // 'events' evaluates at some point to a Seq[Events], a planned event for every called artist.
    ) yield events
  }
```

getPlannedEvents使用Future.sequence方法把Seq[Future[Event]]构建成一个Future[Seq[Event]]作为返回值。也就是说把一串futrue放进一个futrue，这个futrue包含一串对象。下面是代码：
```scala
 def getPlannedEvents(event:Event, artists:Seq[Artist]) = {
    val events = artists.map(artist=> callArtistCalendarService(artist, event.location)) // events是一个 Seq[Future[Event]]
    Future.sequence(events) // 把 Seq[Future[Event]]转为Future[Seq[Event]]。当异步调用callArtistCalendarService都完成后，就可以对futrue取值了
  }
```

与sequence方法相似的还有一个traverse方法。下面是例子：
```scala
  def getPlannedEventsWithTraverse(event:Event, artists:Seq[Artist]) = {
    Future.traverse(artists) { artist=>  // traverse方法接受一个方法块返回一个futrue对象。
      callArtistCalendarService(artist, event.location)
    }
  }
```

使用sequence的话，我们必须事先创建一个Seq[Future[Event]]，才能转化出Future[Seq[Event]]。traverse就不用什么中间步骤了。

终于到了TicketInfoService数据流的最后一步了。包含Weather的TicketInfo要和包含TravelAdvice的TicketInfo对象整合起来。我们使用fold方法来自完成：
```scala
      val ticketInfos = Seq(infoWithTravelAdvice, infoWithWeather)

      val infoWithTravelAndWeather = Future.fold(ticketInfos)(info) { (acc, elem) =>
        val (travelAdvice, weather) = (elem.travelAdvice, elem.weather)

        acc.copy(travelAdvice = travelAdvice.orElse(acc.travelAdvice),
                  weather = weather.orElse(acc.weather))
      }
```
fold方法需要一个初始值、一个方法块。集合中的每个元素都要经过这个方法块对初始值产生影响。

整体代码如下：
```scala
  def getTicketInfo(ticketNr:String, location:Location):Future[TicketInfo] = {
    val emptyTicketInfo = TicketInfo(ticketNr, location)
    val eventInfo = getEvent(ticketNr, location).recover(withPrevious(emptyTicketInfo))

    eventInfo.flatMap { info =>

      val infoWithWeather = getWeather(info)

      val infoWithTravelAdvice = info.event.map { event =>
        getTravelAdvice(info, event)
      }.getOrElse(eventInfo)


      val suggestedEvents = info.event.map { event =>
        getSuggestions(event)
      }.getOrElse(Future.successful(Seq()))

      val ticketInfos = Seq(infoWithTravelAdvice, infoWithWeather)

      val infoWithTravelAndWeather = Future.fold(ticketInfos)(info) { (acc, elem) =>
        val (travelAdvice, weather) = (elem.travelAdvice, elem.weather)

        acc.copy(travelAdvice = travelAdvice.orElse(acc.travelAdvice),
                  weather = weather.orElse(acc.weather))
      }


      for(info <- infoWithTravelAndWeather;
        suggestions <- suggestedEvents
      ) yield info.copy(suggestions = suggestions)
    }
  }

 // 在TicketInfoService处理流中的错误恢复方法
  type Recovery[T] = PartialFunction[Throwable,T]

  // recover with None
  def withNone[T]:Recovery[Option[T]] = { case NonFatal(e) => None }

  // recover with empty sequence
  def withEmptySeq[T]:Recovery[Seq[T]] = { case NonFatal(e) => Seq() }

  // recover with the ticketInfo that was built in the previous step
  def withPrevious(previous:TicketInfo):Recovery[TicketInfo] = {
    case NonFatal(e) => previous
  }
```
这段代码概括了TicketInfoService使用Futrue的逻辑。没有使用任何阻塞调用。combinator方法使这些异步掉用的结果很方便的整合起来。



5.5 Futrue和Actor 
---

在前面的Up and Running 章节，我们使用Spray作为REST服务，它是使用Actor来处理HTTP 请求的，ask方法的返回值也是一个Futrue对象。
```scala
// 引入ask模型，它在ActorRef上添加了ask方法
import akka.pattern.ask
// context包含了actor的dispatcher定义。
import context._
// 为ask方法必须定义一个超时
implicit val timeout = Timeout(5 seconds)
// 抓获上下文中的sender
val capturedSender = sender

// 向TicketSeller请求GetEvents的本地方法
def askEvent(ticketSeller:ActorRef): Future[Event] = {
   // ask方法返回一个Futrue.我们使用mapTo方法将返回的Futrue[Any]转为Future[Int]。
   val futureInt = ticketSeller.ask(GetEvents).mapTo[Int] 
   
   // 查一下所有的子元素，对于指定event都还剩下多少票
   futureInt.map { nrOfTickets =>
      Event(ticketSeller.actorRef.path.name, nrOfTickets)
   }
}
val futures = context.children.map {
  ticketSeller => askEvent(ticketSeller)
}
Future.sequence(futures).map {
  // 给sender回送消息。这个sender就是发送原始GetEvents请求的sender。
  events => capturedSender ! Events(events.toList)
}
 
```

这个例子比在第二章中的代码要清楚多了。每个消息的sender都不一样，这里结合闭包使用变量从context中获取当前message的sender，回送消息。

注意以下当在actor中使用futrue的时候，ActorContext会提供一个Actor的当前view。又由于actor是有状态的，所以闭包中的变量对于外部线程应该是不可变的。最容易达成这种效果的方式是使用不可变对象，然后在将这个值close over进futrue之前就获取这个不可变数据结构的引用，也就是上面例子中的capturedSender。

另一种模式是pipeTo：
```scala
 //引入pipe模型
 import akka.pattern.pipe
 path("events") {
   get { requestContext =>
     // 创建一个actor，为所有的request使用一个response完成HTTP请求
     val responder = createResponder(requestContext)
     // ask方法的Futrue返回值被pipe到responder actor
     boxOffice.ask(GetEvents).pipeTo(responder)
   }
 }


 class Responder(requestContext:RequestContext,
 	ticketMaster:ActorRef)
	 extends Actor with ActorLogging {

   def receive = {

     case Events(events) =>
         requestContext.complete(StatusCodes.OK, events)
         self ! PoisonPill

     // other messages omitted..
  }
 }
```

当我们想用一个actor来处理futrue对象，或者某些actor的ask结果还需要进一步处理的时候，就可以使用pipeTo方法了。

5.6 总结
---
这一章介绍了一个futrue。使用futrue可以创建一个异步处理流，省去不必要的阻塞和线程等待，最大化资源使用率，最小化无作为延迟。

Futrue是一个function返回值的placeholder。

combinator方法可以方便地把futrue的值整合在一起。function可以并行执行，然后通过combinator变成一个有意义的结果。

futrue包含的对象应该是不可变的。

Futrue可以与Actor结合使用，但是要注意使用时close over的状态变量。sender的引用应该在进入闭包钱就获取。Futrue是ask方法的返回值类型，也是pipeTo方法的返回值类型。
