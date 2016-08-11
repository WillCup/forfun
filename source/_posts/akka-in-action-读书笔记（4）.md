title: akka in action 读书笔记（4）
date: 2016-08-09 16:45:27
tags: [akka]
---

内容提要：
- 自救系统
- Let it crash
- Actor生命周期
- Supervision
- 错误恢复策略

4.1 容错的来源
---

我们最理想的世界其实是希望系统一直可用，而且能够保证每个action都能被成功执行。只有两种方法能让这个梦想照进现实：一是使用永远不会失败的组件，二是为每次失败都提供一个完美的补救措施，这个补救措施需要能保证最终能成功完成此次action。在多数架构里，是catch住预期的异常进行处理，但是无论怎样努力都不能阻止预期之外的异常导致失败。

能够容忍错误的系统理论上不错，但是要同时兼顾到高可用、分布式的系统基本是不可能的。主要原因是任何此类系统的大部分我们都不能控制，这部分可能会导致失败。网络是最好的例子，它随时都有可能消失，你的系统再强大也么有用。我们显然不能让一个server从它的灰烬中恢复元气，而且自动修复前面遇到的问题。这就是let it crash 出现的原因，不去想办法回复它，而是提供足够多的可用server。

既然我们不能保证不发生任何失败，那么我们不得不准备一些策略：
- 程序总会有失败。系统必须能够容错，才能继续运行其他服务。可恢复的错误不应该触发或导致灾难性的失败。
- 在某些条件允许的情况下，虽然系统的某一部分发生了问题，但是并不影响其他大部分的服务，那么我们应该切断这部分坏程序以免它产生一些不正确的结果，影响系统的其他部分正常运行。
- 一些重要的组件我们需要有备份（可以考虑放在不同的server或者使用不同的resources），当正在运行的失败的时候，能够进行热切换，使系统可用性快速恢复。
- 系统的一部分出问题不应该影响整个系统垮掉，所以我们需要一个措施来隔离某些特殊的失败，且需要在这些失败发生之后有处理措施。

当然，akka并没有容错的银弹。我们还是要处理失败，但是我们可以使用一种cleaner,more application-specifc的方式来处理。下面是Akka关于容错的一些特性：

名称 | 解释
--- | ---
Fault containment or isolation | 一个fault应该是系统的一部分，不应导致整体crash
Structure | 隔离某个失败的组件意味着需要某个结构来实施隔离，系统需要一个定义好的Structure，把可以被隔离的部分放进去
冗余 | 当一个组件失败之后，应该有备份的替代它
替代 | 如果一个失败的组件可以被隔离，我们也可以在Structure中替换掉它。
重启 | 如果一个组件出现了不正确的状态，我们应该能够把它重新初始化。不正确的那个state或许就是导致错误的原因
组件生命周期 | 一个失败的组件必须被隔离，如果它不能恢复的话，就必须被终结和移除，或者重新处理化它的state。有些必要的周期：start, restart, terminate
暂停 | 当一个组件失败了，我们应该在它被修复之前把所有对它的调用都弄成suspended。这些调用应该保留下来，用来分析当前组件失败的原因
Separation of Concerns 分散关注 |  如果容错代码能够与普通执行代码分割开就最好了。在普通的流程当中，容错算是横向切入的了。把普通流程与容错恢复流程分离开会简化我们的工作

有人或许会问，用普通的异常捕捉机制就不能搞这套东西么？普通的异常是用来回退一系列操作来保证数据一致性的。但是既然提到了，我们还是用下面的部分来讨论一下。

##### 4.1.1 普通异常方式
假设有这样一个应用： 多线程，收集log，从文件中分析有兴趣的信息，然后转存为行对象，然后把这些行写入某个数据库。有一些file watcher进程持续盯着增长的文件，然后通知多个线程来处理新的文件。下图是一个概览：

![应用概览](/imgs/akka/akka_4_process_logs_app.png)

如果数据库连接断了的话，我们期望切换到另一个数据库继续写数据，而不是回滚。如果连接开始出现问题，我们或许希望马上停掉它，以免app的其他部分使用它。有时我们只要重启这个连接，摆脱这个连接内某些错误的状态就可以了。可以用伪代码看一下潜在的问题区域。先看一下使用标准的异常处理方法直接在同一个数据库上重启一个新的连接。

首先，多线程里都已经准备好了要入库的数据。下图是创建writer：


![创建DB writer](/imgs/akka/akka_4_create_writer.png)

writer的依赖是通过构造方法传入的，数据库工厂的配置，都通过创建爱你writer的thread传入。下一步，我们准备log processor。每个processor都有一个writer的引用，用来存储行数据。如下图：

![创建log processor](/imgs/akka/akka_4_create_log_processor.png)

方法的调用栈如下图：

![处理log文件的方法调用栈](/imgs/akka/akka_4_call_stack.png)

上图流程是被多线程同时调用的。下图战士了当一个DbBrokenConnectionException被跑出的时候的调用栈，这个异常暗示我们需要切换到另一个连接。

![处理log文件的调用栈](/imgs/akka/akka_4_call_stack_log.png)


除了直接向上跑出异常之外，我们期望从DbBrokenConnectionException恢复，替换掉不能使用的连接。我们面临的第一个问题是很难在不打破既有设计的前提下添加代码去恢复连接。另外，我们没有足够的信息来重建连接：我们不知道我们已经成功处理到文件的哪一行了，在处理哪一行时出现的异常。

如果让我们执行的行数和连接信息对所有对象都可见的话，就打破了我们的简单性原则，而且违背了一些基本的实践：封装、控制反转、单一职则。我们只是需要把失败的组件替换掉而已。直接在异常处理中添加恢复的代码会混淆执行log文件的功能性。即便我们找到替换连接的方式，我们还不得不着重注意替换连接的时候，其他的线程没有在使用它，不然可能导致某些行数据丢失。

还有，让多线程之间针对某个异常进行沟通本来也不是什么标准特性，你必须要自己实现它。回顾容错的必备特性，想一下这种处理方式有么有可能站住脚：
- 错误隔离。许多线程都可以同时抛出异常，所以隔离相当困难。我们必须得加些锁才行。
- Structure。必须自己创建
- 冗余。 异常不断被抛往上一层。或许会丢失掉一些重要的信息。
- 替换。 没有默认的策略支持在调用栈中替换一个对象，你必须得自己想个办法。有些依赖注入的框架支持这个特性，但是只是在特别简单的直接引用到旧的实例的时候。还有就是如果你要替换某个对象的时候，最好考虑好多线程问题。
- 重启。
- 组件生命周期。
- 暂停
- 关注分离。异常处理代码和业务代码混淆在一起，么有独立性。

简述以下主要的问题：
- 在first-class中，重新创建对象以及他们的依赖，在app中替换掉他们是不可能的 
- 直接相互沟通的对象很难隔离
- 容错代码和功能代码混淆在一起

回头看看Akka。 Actor可以有Props对象重建，是actor system的一部分，actor之间通过actorRef进行沟通。下面我们看一下actor怎样把功能代码和容错代码分离，actor的生命周期怎样保证actor的暂停和重启。


##### 4.1.2 Let it crash

前面介绍了使用普通对象和异常处理机制来构建一个容错系统有多难，下面我们展示以下Actor的风采。

akka的容错恢复代码与功能代码是分开的。功能流包含处理普通message的actor，恢复流包含监控功能流中actor的actor。这种监控其他actor的actor叫做supervisor，如下图：

![功能流和恢复流](/imgs/akka/akka_4_normal_recovery_flows.png)


actor中并不捕获异常，就直接让actor挂掉就性了。actor代码中只包含功能处理，而且没有异常处理逻辑或者恢复逻辑，不参与恢复过程，这样就清晰多了。crashed actor的mailbox在恢复流里的supervisor处理完exception之前都是暂停的。那么一个actor怎样才能变成一个supervisor呢？akka有一个父监管的机制，就是创建actor的actor自动成为被创建actor的supervisor。一个supervisor并不捕获异常，也不修复actor或者actor的状态，只负责判断怎样恢复，然后触发相应的策略。supervisor有4种策略：
 - restart。使用失败actor的Props重建一个actor。这个新的actor重启之后，就继续处理消息。因为app的其他部分都是使用ActorRef来跟actor沟通，新的actor实例可以自动获取到下一条message。
 - resume。给当前actor新生，假装啥都没发生
 - stop。终结actor。
 - escalate。supervisor不知道怎样处理这个问题，抛给它的supervisor。

下图是针对log处理系统的一个示例：

![log处理系统中的功能流与恢复流](/imgs/akka/akka_4_normal_recovery_flows_akka.png)

上图中给出的容错方案是，当DbBrokenConnectionException发生的时候，使用restart策略，替换一个重建的dbWriter actor。

多数情况下，我们不希望重复执行那条导致失败的信息。比如logprocessor遇到一个损坏的文件，这导致mailbox走火入魔，任何message都不会被处理了，因为这个损坏的文件导致我们一次又一次的失败。鉴于此，如果是akka默认对于restart策略处理的actor，不再把处理失败的message放回mailbox了，如果不想这样，后面我们会说一下怎样做。

下图展示了当supervisor选择restart策略后，一个挂掉的dbWriter actor实例怎样被一个新的实例替换掉：

![使用restart处理DbBrokenConnectionException异常](/imgs/akka/akka_4_restart_handle_exception.png)


结合容错性，说一下let it crash的好处：
- 错误隔离。supervisor可以决定是否终结actor。
- Structure。 actor system的actorRef层级使得在不影响其他actor的前提下替换actor实例
- 冗余。一个actor可以被另一个替换掉。在断掉的数据库连接例子中，新的actor实例可以连接到不同的数据库。supervisor也可以决定停掉错误的actor创建另一种类型的actor。另外还可以把message路由给一个负载均衡的actor集群中，后面第八章我们会讲到。
- 替换。actor总是可以被它的Props重建。supervisor不用了解重建actor的任何细节。
- 重启。通过restart实现
- 组件生命周期。actor是一个active component。可以被启动，停止，重启。下一节我们会讨论actor的生命周期。
- 暂停。当actor挂掉的时候，它的mailbox就暂停了，直到supervisor处理完毕。
- 关注隔离。负责功能的actor与负责容错的actor已经隔离开了。


4.2 Actor生命周期
---
actor在创建之后就自动启动了。在被停止之前，actor一直保持在Started state。停止之后的state是Terminated，这时actor就不能处理message了，而且会被GC回收。当state是Started的时候，它既可以被重启也可以设置它的state为其他值。

一个actor的生命周期中有三种类型的event发生：
 - 被创建和启动，Start
 - 被重启，Restart
 - 被停止，Stop

在Actor trait中有几个hook对应这生命周期的改变。我们可以在这些hook里添加自己的代码，用来为新建的actor指定一些自定义的state，也可以在restart之前处理前面处理失败的message，还可以清空某些资源占用。下面我们看看这三种event，并了解一下hook的重写。

#### 4.2.1 Start事件

一个普通的actor是通过actorOf方法创建并启动的。顶层actor是被ActorSystem的actorOf方法床架难点。父actor通过自身的ActorContext的actorOf方法创建子actor。

![启动一个actor](/imgs/akka/akka_4_start_actor.png)

实例创建好之后，就会被Akka启动。preStart hook是在actor启动之前被调用的。
```scala
override def preStart() {
	println("preStart")
}
```

一般这个hook可以用来设置actor的初始状态。

#### 4.2.2 Stop事件

actor通常通过调用ActorSystem和ActorContext对象的stop方法停止，或者发送一个PosionPill message给它。

![停止actor](/imgs/akka/akka_4_stop_actor.png)

postStop hook与preStart对称，在actor被终结之前，state被修改为Terminated之后调用。

```scala
override def postStop(){
	println("postStop")
}
```
一般用来释放资源，或者保存当前actor处理的一些信息到actor外界，以便新的actor实例可以使用。stopped actor和它的ActorRef会断开连接。ActorRef会被重定向到actor system的deadLetters ActorRef，这个ActorRef专门用来接收发送给挂了的actor的message。


#### 4.2.3 Restart事件

![重启actor](/imgs/akka/akka_4_restart_actor.png)

restart发生后，触发挂掉的actor的preRestart方法。在这个hook中挂掉的actor实例可以储存它当前的一些状态信息。

```scala
 override def preRestart(reason: Throwable, // 被当前actor跑出的异常
                          message: Option[Any]) { //导致错误的message
    println("preRestart")
    super.preRestart(reason, message) //警告：这是调用super的实现
  }
```

重写这个hook的时候需要额外注意。默认的preRestart实现是先停止所有的子actor，然后再调用postStop hook。如果忘记调用super.preRestart方法，刚才提到的默认逻辑就不会执行了。记住actor是由Props创建的。Props对象其实是调用的actor的构造方法。actor可以在它的构造方法中创建子actor。如果挂掉的actor的子actor没有被停掉，那么后面随着父actor的多次restart，会有很多的子actor，就泛滥了。


注意restart和stop停掉actor 的方式是不一样的。在restart中挂掉的actor实例不会产生Terminated message。新的actor实例在restar期间就连接到了原来的ActorRef上。而stoped事件中stopped actor要将message重定向。两者相同的是在挂掉的actor被切离actor system之后都会调用postStop。

保存state的方式：可以给自己的mailbox发一条message，也可以存在类似DB的介质中。


调用完preStart hook之后，一个actor实例就被创建了。在postRestart hook被调用后。。。
```scala
  override def postRestart(reason: Throwable) { //当前actor抛出的异常
    println("postRestart")
    super.postRestart(reason) //警告：调用super实现

  }
```


这里也有个警告，就是调用父级实现，它会触发preStart方法的执行。如果你确认不需要调用preStart方法，就不用调用这个父级实现了。


#### 4.2.4 把生命周期整合起来

把上面所有的event结合起来就是actor的整体生命周期了，下图只展示了一个restart：
![actor的整体生命周期](/imgs/akka/akka_4_full_life.png)

对应整体生命周期的hook代码如下：

```scala
package aia.faulttolerance

import akka.actor._

class LifeCycleHooks extends Actor with ActorLogging {
  println("Constructor")
  //<start id="ch3-life-start"/>
  override def preStart() {
    println("preStart") //<co id="ch3-life-start-1" />
  }
  //<end id="ch3-life-start"/>

  //<start id="ch3-life-stop"/>
  override def postStop() {
    println("postStop") //<co id="ch3-life-stop-1" />
  }
  //<end id="ch3-life-stop"/>

  //<start id="ch3-life-pre-restart"/>
  override def preRestart(reason: Throwable, //<co id="ch3-life-pre-restart-1a" />
                          message: Option[Any]) { //<co id="ch3-life-pre-restart-1b" />
    println("preRestart")
    super.preRestart(reason, message) //<co id="ch3-life-pre-restart-2" />
  }
  //<end id="ch3-life-pre-restart"/>

  //<start id="ch3-life-post-restart"/>
  override def postRestart(reason: Throwable) { //<co id="ch3-life-post-restart-1" />
    println("postRestart")
    super.postRestart(reason) //<co id="ch3-life-post-restart-2" />

  }
  //<end id="ch3-life-post-restart"/>

  def receive = {
    case "restart" =>
      throw new IllegalStateException("force restart")
    case msg: AnyRef =>
      println("Receive")
      sender() ! msg
  }
}

```

下面测试一下：
```scala
package aia.faulttolerance

import org.scalatest.{WordSpecLike, BeforeAndAfterAll}
import akka.testkit.TestKit
import akka.actor._

class LifeCycleHooksTest extends TestKit(ActorSystem("LifCycleTest")) with WordSpecLike with BeforeAndAfterAll {

  override def afterAll() {
    system.terminate()
  }

  "The Child" must {
    "log lifecycle hooks" in {
      //<start id="ch3-life-test"/>
      val testActorRef = system.actorOf( //<co id="ch3-life-test-start" />
        Props[LifeCycleHooks], "LifeCycleHooks")
      testActorRef ! "restart" //<co id="ch3-life-test-restart" />
      testActorRef.tell("msg", testActor)
      expectMsg("msg")
      system.stop(testActorRef) //<co id="ch3-life-test-stop" />
      Thread.sleep(1000)
      //<end id="ch3-life-test"/>

    }
  }
}
```


#### 4.2.5 生命周期监控

ActorContext的watch方法负责监控actor的死亡，unwatch方法反注册某个actor。一旦一个actor调用了某个ActorRef的watch方法，他就被这个ActorRef监控了。当被监控的actor被终结后，监控的actor会收到一个Terminated message，这个message只包含挂掉的actor的ActorRef。


```scala
package aia.faulttolerance

import akka.actor._
import akka.actor.Terminated

object DbStrategy2 {
  class DbWatcher(dbWriter: ActorRef) extends Actor with ActorLogging {
    context.watch(dbWriter) //watch dbWriter的生命周期
    def receive = {
      case Terminated(actorRef) => //被终结的actor的actorRef被放在Terminated message中传递
        log.warning("Actor {} terminated", actorRef) //监控者打印出dbWriter被终结的消息
    }
  }
  //<end id="ch03-termination"/>
}
```

与supervisor只能是父actor对子actor不同，actor之间的监控是没有任何规则的。只要某个actor可以访问到被监控actor的ActorRef，那么它就可以调用context.watch(actorRef)进行监控。监控(monitor)与监管(supervisor)联合起来使用会有特别强大的效果，下一节我们会介绍。

4.3 监管
---

这一节我们继续以log处理app为例，深入看一下监管的细节处理。这一节我们关注一下`用户空间`也就是actor path为/user下的监管层级。这个actor path是所有app的actor的家。/user之外的path我们会在actor system关闭过程中涉及到。首先我们先看一下为某个app定义supervisor的层级的几种方式，对比一下优缺点。然后看一下怎样在supervisor中自定义监管策略。

#### 4.3.1 监管层级

actor在被创建的时候就注定了由它的创建者actor监管。父actor只有终结它的子actor才能接触这种监管关系。所在在app启动的时候就需要慎重考虑监管策略，尤其是不想整批整批的替换子actor的时候。

actor所处的层级越高，危险性越大。极端情况，顶层actor出现错误的话，整个系统的所有actor都需要重启。

下图展示了supervisor和actor之间的沟通，以及一个新的文件产生到存储入数据库的信息流：

![supervisor分离与信息流](/imgs/akka/akka_4_message_flow_supervisor.png)

图中可以看到fileWatcherSupervisor同时创建了fileWatcher和logProcessorsSupervisor。fileWatcher需要一个logProcessor的关联对象，而这个logProcessor是由logProcessorsSupervisor创建的，所以我们不能直接给fileWatcher一个ActorRef。我们不得不添加一些信息，发送给logProcessorsSupervisor，向它请求logProcessor的ActorRef，然后把ActorRef转给fileWatcher。

这种方法的好处是actor之间可以之间沟通，supervisor只负责监管与创建actor实例。缺点是我们之恩那个使用restart策略，否则message就会被发送给deadLetters，丢失掉。父监管模式使得监管从信息流中解耦有些难。logProcessor只能被logProcessorsSupervisor创建，这样就很难传递它的ActorRef给fileWatcher。

下一幅图是另一种实现方式。supervisor不只是创建和监管actor，它们还负责传递消息流给它的子actor。对于被监管的actor来说没有改变什么，因为真正的发送者是透明的。比如logProcessor会认为他的message是从fileWatcher获取的，但是实际上logProcessorSupervisor参与了这个message的转发。

![supervisor转发消息流](/imgs/akka/akka_4_supervisor_forward_message.png)

这种方式的好处是间接提供了层级。supervisor可以在其他actor不知情的前提下终结或者扩张它的子actor。相比前面的方式，这种方式不会出现信息流断裂的情况。这种方案更适合父监管模式，因为supervisor可以直接使用Props对象创建它的子actor。比如fileWatcherSupervisor可以直接使用它创建的logProcessingSupervisor的ActorRef去创建fileWatcher。以下是实现代码：

![监管层级实现代码](/imgs/akka/akka_4_supervisor_hierarchy_code.png)

Props对象在actor之间来回传递，它包含了创建子actor所需要的依赖信息，这样每个actor在创建子actor的时候就不用关心细节了。最顶层的actor是由`system.actorFor`方法创建爱你的，因为我们想让系统中其他由supervisor创建的actor都沿着它这条发源地走下去。下一节我们会了解一下创建actor的具体过程：主要就是使用Props对象，调用`context.actorOf(props)`方法创建一个子actor，然后把当前actor作为每个新生actor的supervisor。

#### 4.3.2 预订义监管策略

创建app的顶层actor的是/user路径，它被`user guardian`监管。user guardian的默认监管策略是一旦发生异常，就直接重启它的子actor。actor的默认监管策略可以通过实现supervisorStrategy方法进行重写。在supervisorStrategy中有两个预订义的监管策略：defaultStrategy和stoppingStrategy。顾名思义，前者是所有的actor的默认监管策略，如果不重写的话，就一直使用这个策略：
![默认监管策略](/imgs/akka/akka_4_default_strategy.png)

 - Decider对异常模式匹配选择一个Directive
 - Directive包含：Stop, Start, Resume, Escalate这几种
 - defaultDecider返回OneForOneStrategy

有时我们只需要停掉出错的actor，有时一旦有一个子actor出错我们就需要停掉所有的子actor。OneForOneStrategy代表这子actor不会有相同的命运，只有挂掉的子actor会被Decider裁决。还有AllForOneStrategy，他就是只要有一个子actor出问题，所有的子actor就都需要遭受裁决。


下图是stoppingStrategy的代码：
![stoppingStrategy代码](/imgs/akka/akka_4_stopping_strategy.png)

发生任何异常stoppingStrategy都会停掉所有的子actor。如果不是发生Exception，而是Error呢？任何监管策略处理不了的Throwable都会向上抛给当前actor的父supervisor，抛到最顶层，也就是user guardian那里还处理不了的话就会导致整个actor system挂掉。

#### 4.3.3 自定义策略

每个app为了容错都需要定义一些自己的策略。前面我们提到一个supervisor对待挂掉的actor有四种处理措施，下面我们还是以log处理app来举例。
1. Resume。 忽略错误，使用现在的actor继续执行message
2. Restart。 移除挂掉的actor实例，使用新的actor实例替换它
3. Stop。 永久终结这个actor
4. Escalate。 把失败报给自己的supervisor，让自己的父actor去处理。

首先看一下在log处理app中可能出现的异常。
```scala
@SerialVersionUID(1L)
  class DiskError(msg: String)
    extends Error(msg) with Serializable //当硬盘挂掉导致资源不可用时的一个不可恢复的Error

  @SerialVersionUID(1L)
  class CorruptedFileException(msg: String, val file: File)
    extends Exception(msg) with Serializable //日志文件损坏造成的Exception

  @SerialVersionUID(1L)
  class DbBrokenConnectionException(msg: String)
    extends Exception(msg) with Serializable //数据库连接断开的Exception
```

actor之间发送的message可以放在一个protocol对象中统一管理。

```scala
  object LogProcessingProtocol {
    // represents a new log file
    case class LogFile(file: File) //从fileWatcher接收到一个log文件。log processor要处理这个message
    // A line in the log file parsed by the LogProcessor Actor
    case class Line(time: Long, message: String, messageType: String) //LogFile里的一行数据。数据库的writer需要将这个数据写到数据库连接里
  }

}
```

首先从最底层看一下可能出现DbBrokenConnectionException的dbWriter。

```scala
class DbWriter(databaseUrl: String) extends Actor {
    val connection = new DbCon(databaseUrl)

    import LogProcessingProtocol._
    def receive = {
      case Line(time, message, messageType) =>
        connection.write(Map('time -> time,
          'message -> message,
          'messageType -> messageType)) //向数据库写数据就可能引发这个异常
    }
  }
```

DbWriter是由DbSupervisor监管的，前面说到DbSupervisor会把所有的message转发给dbWriter。
```scala
class DbSupervisor(writerProps: Props) extends Actor {
    override def supervisorStrategy = OneForOneStrategy() {
      case _: DbBrokenConnectionException => Restart //发生DbBrokenConnectionException的时候执行Restart策略
    }
    val writer = context.actorOf(writerProps) //supervisor使用一个Props对象创建dbWriter
    def receive = {
      case m => writer forward (m) //supervisor把收到的所有的message转发给dbWriter ActorRef
    }
  }
```

再向上一些，就是logProcessor。
```scala
  class LogProcessor(dbSupervisor: ActorRef)
    extends Actor with LogParsing {
    import LogProcessingProtocol._
    def receive = {
      case LogFile(file) =>
        val lines = parse(file) //读取文件就可能引发异常，导致当前actor挂掉
        lines.foreach(dbSupervisor ! _) //把文件的每一行都发送给dbSupervisor
    }
  }
```

当出现文件损坏问题的时候，它的supervisor采取措施如下：
```scala
  class LogProcSupervisor(dbSupervisorProps: Props)
    extends Actor {
    override def supervisorStrategy = OneForOneStrategy() {
      case _: CorruptedFileException => Resume //出现CorruptedFileException.的时候，跳过损坏的文件，继续执行
    }
    val dbSupervisor = context.actorOf(dbSupervisorProps) //创建database的supervisor，而且被当前supervisor监管
    val logProcProps = Props(new LogProcessor(dbSupervisor))
    val logProcessor = context.actorOf(logProcProps) //通过Props创建logProcessor

    def receive = {
      case m => logProcessor forward (m) //把所有的message都转发给logProcessor ActorRef
    }
  }
```

再向上一个层级，FileWatcher:
```scala
class FileWatcher(sourceUri: String,
                    logProcSupervisor: ActorRef)
    extends Actor with FileWatchingAbilities {
    register(sourceUri) //使用file的watching API注册监听文件变化
    import FileWatcherProtocol._
    import LogProcessingProtocol._

    def receive = {
      case NewFile(file, _) => //利用file watching API，当发现新文件的时候发送出去
        logProcSupervisor ! LogFile(file) //发送给logProcSupervisor
      case SourceAbandoned(uri) if uri == sourceUri =>
        self ! PoisonPill //如果出现硬盘损坏问题，fileWatcher就自杀
    }
  }
```

fileWatcher这个actor自杀之后，由于DiskError没有被定义处理策略，所以就会自动向上传递。这是一个不可恢复的error，所以FileWatchingSupervisor决定停止所有的actor，使用了AllForOneStrategy：
```scala
class FileWatcherSupervisor(sources: Vector[String],
                               logProcSuperProps: Props)
    extends Actor {

    var fileWatchers: Vector[ActorRef] = sources.map { source =>
      val logProcSupervisor = context.actorOf(logProcSuperProps)
      val fileWatcher = context.actorOf(Props(
        new FileWatcher(source, logProcSupervisor)))
      context.watch(fileWatcher) //对filewatcher实施监控
      fileWatcher
    }

    override def supervisorStrategy = AllForOneStrategy() {
      case _: DiskError => Stop //如果出现DiskError，就停掉所有的fileWatcher
    }

    def receive = {
      case Terminated(fileWatcher) => //确认这是fileWatcher的Terminated message
        fileWatchers = fileWatchers.filterNot(w => w == fileWatcher)
        if (fileWatchers.isEmpty) self ! PoisonPill //所有的fileWatcher都停掉之后，服毒自尽
    }
  } 
```

OneForOneStrategy和AllForOneStrategy都提供了maxNrOfRetries和withinTimeRange作为构造参数，分别代表重试几次停止和停止超时时间，按需设置。设置之后，如果重试几次或者超时么有停止，就自动向上传递。下面是一个数据库supervisor的例子：
```scala
class DbImpatientSupervisor(writerProps: Props) extends Actor {
    override def supervisorStrategy = OneForOneStrategy(
      maxNrOfRetries = 5,
      withinTimeRange = 60 seconds) { //如果5次restart或者60s内没有解决这个问题，就自动上传这个问题给父级supervisor
        case _: DbBrokenConnectionException => Restart
      }
    val writer = context.actorOf(writerProps)
    def receive = {
      case m => writer forward (m)
    }
  }
```

4.4 总结
---
容错是Akka相当给力的一部分，也是这个工具对于并发处理的重要组成。'Let it Crash'的哲学并不是主张把所有可能发生的问题都忽略，它也不能处理掉所有的问题。相反，编程人员是希望有恢复措施的，但是怎样使这措施顺利执行是空前的。这一届中我们使用log处理app讲解了容错：
- 监管意味着我们的恢复代码与功能代码的隔离
- 基于message的Actor模型意味着即便某个actor挂掉，我们还是能继续工作
- 我们可以忽略、丢弃、重启：自由选择
- 我们甚至可以将自己处理不了的错误直接抛给父级supervisor

有了akka，容错变得很简单。下一章我们要构建几个不同类型的基于actor的app，看一下怎样提供configuration, logging于发布之类的功能。

