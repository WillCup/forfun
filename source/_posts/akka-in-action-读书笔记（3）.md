title: akka in action 读书笔记（3）
date: 2016-08-01 16:55:55
tags: [akka]
---
这一篇我们一起看一下:使用akka测试驱动开发。

当TDD刚出现的时候，大家都感觉它浪费很长时间。尤其感觉让测试一些组件的协作更花费时间。Actor提供了一个又去的解决方案：

- actor可以更直接的测试，因为他们本身就代表具体行为。【几乎所有的TDD都有至少一个BDD（Behavior-Driven-Development）】
- 普通的单元测试只能测试接口，而且只能分别测试
- Actor是基于消息的。也就是只要发送message就可以很轻松的模拟各种行为了。

    ####  3.1 测试Actor
    我们使用ScalaTest单元测试框架来测试发送与接收message。
    看代码以前，我们要了解，测试Actor相比测试不同的对象还是要稍微复杂一点儿的，原因如下：
    - 时间。 发送message是一步的，所以不知道多长时间才能去assert返回值
    - 异步。 Actor是在多个线程中并行运行的。多线程测试相比单线程测试要复杂以下，需要处理一些并发原是变量来同步多个actor的结果，比如lock、latch、barrier等。这些正是我们要远离的。仅仅是一个barrier的错用都会导致一个单元测试block住，从而影响整个test suite。
    - 无状态。actor的内部状态是隐藏的，只能通过ActorRef来访问。单元测试的时候要注意这点。
    - 协作/整合。如果要测试几个actor之间的整合，必须在这些actor之间窃听它们之间传递的message是不是期望值。

	---

    虽然有以上困难，好在Akka也提供了`akka-testkit` module。这个module包含了几个可以很方便的进行actor测试的工具。这个module覆盖了下面几种不同的测试类型：
    - 单线程测试。 一个actor实例通常并不能直接使用。testkit提供了TestActorRef，它允许访问指定的actor实例。这样就能直接调用actor的方法，来测试我们定义的actor，向测试普通对象一样了。
    - 多线程单元测试。`TestKit`和`TestProbe`两个类，让我们方便的接收回复，检查message，为特定message设置到达时限。TestKit有专门的方法来assert期望值。Actor在多线程环境中使用一个普通的dispatcher来运行。
    - 多jvm测试。测试远程actor system的时候会用到。

    TestActorRef继承了LocalActorRef，设置dispatcher为CallingThreadDispatcher, CallingThreadDispatcher只用来测试。（它直接使用调用者的线程，而不是另起一个线程）。

    明眼人看的出来，多线程环境基本贴近于生产环境，所以我们接触最多。
    关于[ScalaTest](http://www.scalatest.org/)

##### 3.1.1 准备测试

因为有一些通用的逻辑，我们先定义一个Trait，使测试能够在单元测试结束后自动停止测试系统。
```scala
package aia.testdriven
//<start id="ch02-stopsystem"/>
import org.scalatest.{ Suite, BeforeAndAfterAll }
import akka.testkit.TestKit

trait StopSystemAfterAll extends BeforeAndAfterAll { 
		//继承ScalaTest的基础trait
  this: TestKit with Suite => //这个trait只能与使用TestKit的测试使用
  override protected def afterAll() {
    super.afterAll()
    system.terminate() //测试完成后关闭测试系统
  }
} 

```

我们写测试的时候就可以mixin这个trait了。TestKit暴露了一个system变量，可以用来访问测试中创建的actor以及其他信息。


#### 3.2 单向消息

记住，我们已经不再使用那种触发某个动作，等待response的方式了，所以我们的例子都是使用`tell`来发送单向消息也就顺理成章了。拜这种fire and forget方式所赐，我们不知道message什么时候到达actor，或者即便到了，我们怎么样进行测试呢？我们期望是发送一个message给一个actor，然后这个actor能够正确的处理这条message。如果actor的处理动作对于外界完全不可见，我们只能保证它处理之后没有出现任何错误，我们应该使用TestActorRef来访问actor的状态，确保它完好。下面是我们需要注意的三种actor：
 - SilentActor。这种actor的行为对于外界不可见，但是中间它会创建一些内部状态。只要处理中没有出现任何错误或异常我们就认为测试通过。检测处理中内部状态的变化就可以了。
 - SendingActor。这种actor处理完收到的message之后会发送message到其他actor。我们把这种actor当做一个黑盒，查看输入与输出的message就行了。
 - SideEffiectingActor。这种actor接收message，与一个普通对象进行一些交互。发送message给这种actor之后，需要确认以下普通对象是否被按照预期修改了。

    下面我们为以上列举的每种actor都写个test。

##### 3.2.1 SilentActor例子
```scala
package aia.testdriven

import org.scalatest.{WordSpecLike, MustMatchers}
import akka.testkit.TestKit
import akka.actor._

//This test is ignored in the BookBuild, it's added to the defaultExcludedNames
//<start id="ch02-silentactor-test01"/>
class SilentActor01Test extends TestKit(ActorSystem("testsystem")) 
  with WordSpecLike //WordSpecLike提供了一种易读的DSL来测试BDD形式的测试
  with MustMatchers //MustMatchers提供了易读的assertion
  with StopSystemAfterAll { //我们刚刚定义的，完成测试后关闭system

  "A Silent Actor" must { // 写上文本描述
    "change state when it receives a message, single threaded" in { // 每个in都描述一个指定的测试
      //Write the test, first fail
      fail("not implemented yet")
    }
    "change state when it receives a message, multi-threaded" in {
      //Write the test, first fail
      fail("not implemented yet")
    }
  }

}
//<end id="ch02-silentactor-test01"/>

//定义一个Actor
class SilentActor extends Actor {
  def receive = {
    case msg =>  // 所有接收到的message都被吞如黑洞
  }
}
//<end id="ch02-silentactor-test01-imp"/>

```
以上是一个WordSpec形式的测试，可以写多个文本描述。下面我们定义来一个Actor。

我们首先写测试发送message给这个actor，然后观察它内部状态的变化。

SilentActor和SilentActorProtocol都是为了辅助测试的，后者包含了所有前者支持的message类型，可以方便地对message进行分组管理。

```scala
package silentactor02 {

class SilentActorTest extends TestKit(ActorSystem("testsystem"))
    with WordSpecLike
    with MustMatchers
    with StopSystemAfterAll {

    "A Silent Actor" must {
      //<start id="ch02-silentactor-test02"/>
      "change internal state when it receives a message, single" in {
        import SilentActor._ //引入Protocol定义的message类型

        val silentActor = TestActorRef[SilentActor] //为单线程测试创建TestActorRef
        silentActor ! SilentMessage("whisper")
        silentActor.underlyingActor.state must (contain("whisper")) //测试刚刚放进的message在目标actor中
      }
    }
  }

  // 对应提到的Protocol
  object SilentActor { 
    case class SilentMessage(data: String) //SilentActor可以处理的一种message类型
    case class GetState(receiver: ActorRef)
  }

  class SilentActor extends Actor {
    import SilentActor._
    var internalState = Vector[String]()

    def receive = {
      case SilentMessage(data) =>
        internalState = internalState :+ data //state这里是被存储在一个vector中，每个message都会被加入这个vector
    }

    def state = internalState //放回当前的所有内部状态
  }
}
```
由于返回的状态列表是不可变的，测试并不会影响我们验证期望数据。

下面我们看一下多线程版本的。
```scala
class SilentActorTest extends TestKit(ActorSystem("testsystem"))
    with WordSpecLike
    with MustMatchers
    with StopSystemAfterAll {

    "A Silent Actor" must {
     
      "change internal state when it receives a message, multi" in {
        import SilentActor._ 

        val silentActor = system.actorOf(Props[SilentActor], "s3") //创建actor的方式不同
        silentActor ! SilentMessage("whisper1")
        silentActor ! SilentMessage("whisper2")
        silentActor ! GetState(testActor) // 添加了一种消息类型，来返回当前的状态
        expectMsg(Vector("whisper1", "whisper2")) //测试断言
      }
    }

  }

  object SilentActor {
    case class SilentMessage(data: String)
    case class GetState(receiver: ActorRef) //添加的消息类型
  }

  class SilentActor extends Actor {
    import SilentActor._
    var internalState = Vector[String]()

    def receive = {
      case SilentMessage(data) =>
        internalState = internalState :+ data
      case GetState(receiver) => receiver ! internalState //返回参数中内部状态给receiver，也就是GetState参数的ActorRef
    }
  }

```

多线程使用TestKit内置的`testsystem`这个ActorSystem来创建SilentActor。这样我们就不能使用多线程的actor去直接访问actor实例了。这里使用GetState消息封装了一个ActorRef作为参数。TestKit有一个testActor让你用来接收你期望的message。我们在receive里添加的GetState的方法返回了指定actorRef的内部状态。expectMsg方法是期望每个发送到testActor的message都被assert。

这里的internalState依然是不可变的，所以仍然是完全安全的。

##### 3.2 SendingActor例子

回想以下第一章中的买票的例子，我们需要测试的是党我们买了一个Ticket，那么可以卖的Ticket总数就要减少一个。既然TicketingAgent负责减票并将event继续传递给下一个TicketingAgent，我们只需要创建一个SendingActor，把它作为下一个接收者放到chain里去，然后观察票数变化情况就行了。

```scala
package aia.testdriven

import akka.testkit.TestKit
import akka.actor.{ Props, ActorRef, Actor, ActorSystem }
import org.scalatest.{WordSpecLike, MustMatchers}

class SendingActor01Test extends TestKit(ActorSystem("testsystem"))
  with WordSpecLike
  with MustMatchers
  with StopSystemAfterAll {

  "A Sending Actor" must {
    "send a message to an actor when it has finished" in {
      import Kiosk01Protocol._
      val props = Props(new Kiosk01(testActor)) // 下一个TicketingAgent通过构造函数传递，这里我们传的是testActor
      val sendingActor = system.actorOf(props, "kiosk1")
      val tickets = Vector(Ticket(1), Ticket(2), Ticket(3))
      val game = Game("Lakers vs Bulls", tickets) // 湖人对公牛的门票库
      sendingActor ! game

      expectMsgPF() {
        case Game(_, tickets) => //testAcotr应该接收到了一个event
          tickets.size must be(game.tickets.size - 1) // testActor应该接收到了一个ticket被减少了
      }
    }
  }
}
object Kiosk01Protocol {
  case class Ticket(seat: Int) 
  case class Game(name: String, tickets: Seq[Ticket]) //包含message的event
}

class Kiosk01(nextKiosk: ActorRef) extends Actor {
  import Kiosk01Protocol._
  def receive = {
    case game @ Game(_, tickets) =>
      nextKiosk ! game.copy(tickets = tickets.tail) //一个不可变的去掉第一个ticket的副本被发往下一个TicketingAgent
  }
}
```

下面是一些SendingActor的变种：

Actor | 描述
---|---
MutatingCopyActor | 这种actor创建一个变种的copy，然后发送副本到下一个actor，上面用到的就属于这种
ForwardingActor | 这种actor不修改收到的message，只是转发出去
TransformingActor | 这种actor会根据收到的message创建一个不同类型的message
SequencingActor | 这种actor基于收到的message创建很多的message，然后按序发送出去

前三种使用相同的方式进行测试。

FilteringActor不同于其他的是它会定位我们传递的message是为什么没有通过测试？SequencingActor跟它类似。那么我们怎样确认我们收到了正确数量的message呢？先给FilteringActor写个测试吧。我们要弄一个可以自动过滤重复message的FilteringActor。它有一个已经收到的所有的message的list，每次来新的message的时候都检查是不是已经有了相同的message。

```scala
package aia.testdriven

import akka.testkit.TestKit
import akka.actor.{ Actor, Props, ActorRef, ActorSystem }
import org.scalatest.{MustMatchers, WordSpecLike }

class FilteringActorTest extends TestKit(ActorSystem("testsystem"))
  with WordSpecLike
  with MustMatchers
  with StopSystemAfterAll {
  "A Filtering Actor" must {
    "filter out particular messages" in {
      import FilteringActor._
      val props = FilteringActor.props(testActor, 5)
      val filter = system.actorOf(props, "filter-1")
      filter ! Event(1) // 发送一串message
      filter ! Event(2)
      filter ! Event(1)
      filter ! Event(3)
      filter ! Event(1)
      filter ! Event(4)
      filter ! Event(5)
      filter ! Event(5)
      filter ! Event(6)
      val eventIds = receiveWhile() { //收集testActor收到的所有满足case语句的message
        case Event(id) if id <= 5 => id
      }
      eventIds must be(List(1, 2, 3, 4, 5)) //测试没有收入重复的message
      expectMsg(Event(6))
    }
    "filter out particular messages using expectNoMsg" in {
      import FilteringActor._
      val props = FilteringActor.props(testActor, 5)
      val filter = system.actorOf(props, "filter-2")
      filter ! Event(1)
      filter ! Event(2)
      expectMsg(Event(1))
      expectMsg(Event(2))
      filter ! Event(1)
      expectNoMsg
      filter ! Event(3)
      expectMsg(Event(3))
      filter ! Event(1)
      expectNoMsg
      filter ! Event(4)
      filter ! Event(5)
      filter ! Event(5)
      expectMsg(Event(4))
      expectMsg(Event(5))
      expectNoMsg()
    }
  }
}
object FilteringActor {
  def props(nextActor: ActorRef, bufferSize: Int) =
    Props(new FilteringActor(nextActor, bufferSize))
  case class Event(id: Long)
}

class FilteringActor(nextActor: ActorRef,
                     bufferSize: Int) extends Actor {// 定义buffer大小 
  import FilteringActor._
  var lastMessages = Vector[Event]() //上一批message的vector
  def receive = {
    case msg: Event =>
      if (!lastMessages.contains(msg)) {
        lastMessages = lastMessages :+ msg
        nextActor ! msg // 如果buffer中没有event就发送给下个actor
        if (lastMessages.size > bufferSize) {
          // 当到了最大buffer，最老的message就被忽略了
          lastMessages = lastMessages.tail //<co id="ch02-filteringactor-discard"/>
        }
      }
  }
}


```

receiveWhile方法同样也可以用来测试
