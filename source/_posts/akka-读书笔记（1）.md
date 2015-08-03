title: akka in action 读书笔记（1）
date: 2015-07-16 02:45:52
tags: [akka]
---


## 什么是AKKA

akka：

- 并行处理请求
- 和service与client进行并发交互
- 异步响应
- 事件驱动编程模型

akka是被设计为一个工具而不是一个框架。框架是一个栈的某个独立的元素【比如UI层、web层等】，而akka可以用于这个栈中的任何部分，也可以为这些部分之间提供连接。
akka的每个module是由jar包组成的，已经尽量较少了各个jar之间的耦合，甚至akka-actor除了scala标准包没有任何其他依赖。另外akka还提供了一些集成其他组件(例如camel, zeromq等)的module。下图是akka技术栈的主要组件，也解释了client如果省去一些细节麻烦：
![Figure 1.1 The Akka Stack](/imgs/akka/akka1.1.png)

akka自身提供运行时环境，有一个很小的叫做play mini的内核，可以做的事情简直让你叹为观止。akka应用主要是由actor组成，基于actor编程模型，借鉴自函数式编程语言ERLANG和Fanton。

---

### 并发

问题：
考虑一个卖票的程序，如果一百万人同时向100个售票处买票，但是只有一个地方打印票。
![Figure 1.2 Buying Tickets](/imgs/akka/akka1.2.png)

![Figure 1.3 买票人、售票处、打印中心的关系](/imgs/akka/akka1.3.png)

面对以上问题，好多人都同时征用打印机资源。解决方案有两个。对于一个基于线程的程序来说，最简单的实现就是直接fork，这样马上就会出错。如果是线程池的话，也可能每个请求都花比较长的时间而造成timeout，客户端还是会失败。另外，如果几个线程互相争抢公用的资源同样会造成不良后果。基于message的程序就不会有上面的问题，因为都是不可变的，没有共享状态。


![Figure 1.4 Buying sequence 买票人排队等待](/imgs/akka/akka1.4.png) 
使用线程的话有三个原因可能导致灾难性失败：

- thread starvation 当所有的线程都无暇顾及新request的时候
- race conditions 当某个共享资源被多个线程同时修改，可能会产生混乱的结果
- deadlock

![Figure 1.5 TicketingAgent sequence 售票处排队等待](/imgs/akka/akka1.5.png) 
这样我们可以获得三个好处：

- event和ticket在售票处和打印室中间以不可变的消息状态传递
- 售票处的打印请求被打印室异步放入队列，不用等待打印方法完成或者接受确认
- 售票处和打印室相互之间不是直接使用相互的引用，而是记录联系对方的地址，他们之间通过相互传递message来进行通讯，这些message暂时被放置于一个mailbox中，后面会被按照放入的次序被处理掉。

那么售票处怎么拿到票呢？打印室会给每个售票处发送取票信息，每个Event都会包含N多票和这个Event的状态信息。有很多种方式发送message给所有的售票处，这里我们使用的是让售票处之间相互传递的方式。售票处们也都知道各自的位置，会传达给目标(我们会在akka中常常见到chain的形式，从简单的传递工作状态到分布式的基于actor的责任链设计模式)
![Figure 1.6 Distributing tickets](/imgs/akka/akka1.6.png) 
如果一个售票处取了event中的属于自己的票后还有剩余的，就继续传递下去，代表还有其他售票处没有拿到应得的票。如果所有的售票处都看过这个event了，一段时间后仍然没人认领，那就让这些票过期。

基于消息传递的模式有两个好处，一个是不用管任何锁，另外售票处也不用干等着response，也就是打印的票完成，再去处理下一个，现在打印室打印完了会通知售票处取票。即便打印室器材坏了，售票处仍然可以继续卖票，只要把新的打印室的地址标成原来打印室的地址即可。

---
### 容灾

基于message的方式可以允许在系统部分失灵的情况下保证系统继续有效运行。原因之一就是独立性，**每个actor并不直接进行相互之间的交互**。message是被发送到mailbox里的，爱谁处理谁处理，爱咋处理咋处理，sender发完就完事儿了。同样的事儿对于调用对象方法就行不通了。

![Figure 1.7 Message Passing vs. Function Calling 消息传递与方法调用流程对比](/imgs/akka/akka1.7.png)

独立性还带来了另外的好处。如果某个actor挂了，他的地址被另一个actor所替代，会怎样呢？只要原来的actor被新的actor正确替换了，那么所有这个地址上的message都会被归到新actor名下。错误会被包含在处理不恰当的老的actor中，而新actor只要处理这个老actor产生的message就可以了。（在这种方式下，出错的老actor在抛出异常后就不能继续处理message了，它要**自杀然后魂飞魄散**）。类似Sping的这些框架在container/bean级别做了这些处理，而akka是在运行时提供的可替代性(这就是actor的let it crash思想):**我们事先就做好了准备，一旦某个handler会因为某些莫须有的理由处理不好他们的任务，这时就把它的任务交给别人来避免灾难性的失败**。

需要注意的是：这是双向的。即便是上层失败了，那么下层可以继续工作直到新的领导人来接替工作。**不会有断裂的chain**，开发者不用关心任何chain可能会断裂的可能性，当然也不用针对每个可能性采取相应措施。一个失败的actor被替换的过程不会影响到任何其他的合作者，见下图

![Figure 1.8 Replacing a crashed printing office](/imgs/akka/akka1.8.png)
以上这种方案是akka中对于容错的一种方案**Restart** strategies,其他方案包括：**Resume**, **Stop**以及**Escalate**.Akka提供了选择容错方案的方法，既然akka控制所有的actors来处理message，它知道所有actor的地址，如果有异常发生，就检查一下针对这种异常应该采取怎样的容错策略并执行之。
容错并不意味着对所有的错误全部都完全恢复。**容错系统只是保证系统在部分失灵的情况下能够依然运行**。不同的错误有不同的处理策略，有的错误需要重启部分系统，有的错误或许在检查点不能处理需要上级帮忙....后面我们可以关注一下Akka怎样通过Supervision来处理这些案例。

拿个简单的异常处理来说，一般的程序遇到异常肯定要停下正在处理的工作，处理这个异常或者抛给上级。对于线程组之间肯定是不能共享某个异常的，除非你自己写一个这样的底层机制。异常不会被传到产生它的线程的线程组之外，也就是说我们必须找到在不同线程组之间交流异常的其他方式。一般来说如果一个线程遇到异常基本都是忽视继续执行或者直接停止。或许你可以在log里找到一些证据证明某个thread挂掉或者停止了，但是让系统的其他部分知道这些却是比较难了。当这些线程再分不到不同的机器上的时候，事情变得更加棘手。Akka提供了一种模型来处理错误，不管actor分布在多少机器。**Akka不仅仅可以很好的处理并发，对于容错同样也很在行!**

---
### scale up and out
scale up :在一个server上运行更多的TicketingAgent
scale out : 在更多的server上运行TicketingAgent

#### scale out
我们前面都是在一个JVM上运行的agent的例子。我们这次来看看怎样在多台机器上运行。
message的传递是通过地址来进行的，我们只需要改变地址链接actor的方式就可以了。

![Figure 1.9 Transparent Remote Actors (through Proxies)](/imgs/akka/akka1.9.png)

akka可以给一个位于remote server上的remote actor发送message，然后再通过网络把处理结果传递回来。TicketingAgent并不知道它在命令一个远程的打印室。其实在同一个jvm运行的内存实例也是基本一致的。唯一不一样的地方就是怎样找到remote actor reference，这**只要通过配置文件**就可以了。也就是说，**不用改变任何代码**我们就可以吧scale out切换为scale up。在akka中灵活改变actor地址是非常常见的。remote actor, 集群化甚至测试工具，都会用到。

同样的情况，对于共享可变状态的情形可就惨了，最常见的方案就是把JVM里的状态都push到一个db里去。最大的威胁是在不得不更多地联系控制的需求下，这个db会变得越来越牛逼，而中间层就鸡肋了：所有的class都是DTO或者无状态无实际操作的东西。最黑暗的地方就是当需要更多复杂的处理的时候，db同样也要面临这些改变，这将是灾难性的。好了，贬低完了对方的不好，来看看我们牛逼的akka，你可以使用精心制作的控制系统来将错综复杂的处理逻辑不定转化为操作流。在某些架构中，message queue和web service会组合起来防止这些，但其实就是一种类似akka的模型。Akka帮我们省去了很多共享可变资源的麻烦，基于message模式，而且还可以基于同一段代码同时实现scale up和scale out。
#### scale up
如果我们只想在一个机器上提升性能并scale up呢？假设我为机器新增加了几个cpu核数。在共享可变资源的情况下就是添加线程数了。但是我们都知道，锁会导致资源争夺，就是说工作的线程数总是少于总数，因为有些线程不得不相互等待。尽量少共享就是说尽量少玩儿锁，这就是message机制的目的。使用message机制可以使用更少的线程，只要优化一下处理流程和message分发，性能还胜过使用锁。**每个线程都有一个用来存储运行时数据的栈**。不同的操作系统的栈的大小是不一样的，比如Linux 64位的一般是256K。**栈的大小是影响同时运行在某个server上线程数量的因素之一**。在Linux 64系统上大约4096个线程会用掉1G的内存。

actor运行在一个叫做dispatcher的抽象概念上，dispatcher关注使用哪个线程模型，还有处理mailbox里的message。跟线程池处理方式很像，只是线程池只管调度，而Akka的dispatcher/mailbox结合体处理并传递message。我们可以通过配置层指定dispatch策略，**不用修改任何代码**。

actor是轻量的，因为他们运行于dispatcher之上，actor的数量不一定要跟线程数量成正比。actor比thread占用的空间要小得多，1G内存大约可以有270万个actor。针对不用的需求，有多种类型的dispatcher可选。actor之间可以共享同一个dispatcher，也可以自己有自己的。在调整性能的时候，可以通过配置灵活进行。


## Akka Actor 和 AkkaSystem

前面已经讨论了几个akka的主要概念：

- actor的地址是提供一个方向
- mailbox用来临时保存message和actor自身

我们下面讨论一下这些组件是怎样有机结合起来，解释一下akka底层的线程机制，并且解释一下akka怎样运行在这样的一套机制之下。Actor只是ActorSystem的一部分，ActorSystem负责提供一些组件，而且能够让actor之间相互能够找到彼此。

akka系统中需要保证：

- 不存在可变共享资源
- message不可变
- 异步发送message

当我们使用某个工具包或者框架的时候，这个东西肯定为我们提供了方便的直接可用的东西。像下图1.10中展示的一样，一个actor就是我们可以向他发送message的对象，我们并不关心message被怎样传递，谁来接收这些message。我们的代码只负责在接收到message之后进行对应的处理即可。还可以通过某个协议与其他人进行协作。

现在我们接触了actor，mailbox，address。akka是使用scala原生api构建的actor。scala API提供了actor trait给人继承来构建自己的actor。售票处就可以继承akka的Actor了，因为akka的Actor已经继承了scala的Actor trait，并且还有一些状态信息，还有一个内部队列用来存放票。address就是我们前面提到的ActorRef，也就是Actor reference的简写。这个售票处的ActorRef是给打印室发送message给售票处用的。在akka中，ActorRef有多种形式，只是不会显露出来而已。对于用户而言，只需要使用ActorRef的API就可以了。

下面我们通过图1.10走一个小小的例子，讲述一些我们的代码要做什么，ActorSystem会为我们做什么。
![Figure 1.10 Actor Request Cycle in Action ](/imgs/akka/akka1.10.png)
如你所见：我们向售票处actor要一个ActorRef，获取之后，就用它来发送请求，也就是买票(1至5步)。ActorSystem没有向我们暴露任何细节：怎样处理的这个request，mailbox在哪儿，message是否已经被传递了等等。

当然，前面我们提到的好处依然存在：我们并没有阻塞，干巴巴等待request被处理完毕；request中不包含任何共享的状态信息；实际的消息处理过程也没有对任何公共资源使用锁；我们不必搭理任何并发问题。

那么我们怎样从ActorSystem中获取一个ActorRef呢？ActorPath！
![Figure 1.11 ActorPath URI syntax ](/imgs/akka/akka1.11.png)
在上面的例子中，'poffice1'可以被换成'TicketingAgent2'。然后通过'TicketingAgent2'的ActorRef使用'../TicketingAgent3'还可以拿到它的兄弟节点actor的ActorRef。但是守卫actor只能是'user'。

akka中的**监督机制**就来源于这种层级制度。每个actor都自动是它的子节点的supervisor。当一个child挂了，parent就需要操心使用什么样的策略进行弥补。错误也可以被逐级向上抛出，直到上面的某个supervisor对这类错误感兴趣，处理掉。

akka**不推荐直接通过constructor来创建actor**，因为这样会破坏actor的层级。直接通过constructor创建的actor可以被直接访问并调用其方法，这也破坏了message传递过程中的并发保护。ActorSystem可以创建ActorRef，提供通用的方式来获取某个ActorRef，提供root actor来创建actor层级，并且把运行时其他组件与actor整合起来。

Actor的操作：

- CREATE
每个actor都可以创建自己的子actor，actor的拓扑结构是动态的。
- SEND
actor之间可以异步相互发送message到彼此的mailbox里。
- BECOME
actor的行为是可以动态改变的。每收到一条message，actor都可以以不同的方式处理下一条message，也就是切换它的行为，后面章节我们会遇到。
- SUPERVISE
actor监控自己的子actor，并且为这些孩子擦屁股。在第三章中，我们会见识到处理message和处理error的明确不同。


## 总结

我们接触了ActorSystem和Actor实现的理论与实践，见识到了akka强大的能力。下一章你会发现这么厉害的能力竟然只需消耗一点点儿能量：你可以很快的运行akka，消息管理及并发的复杂性交给akka来搞定。
综上：

- 基于消息传递让并发更加简单
- 并发的同时，我们还可以很简单地scale up 和scale out
- 我们可以任意扩展应用里的request和消息处理的组件
- 基于消息传递同样解决了容错
- 监管机制提供了一个并发并且容错的模型
- akka让我们的代码不用特别复杂，但是拥有强大的力量

下一章我们会启动运行我们的第一个akka应用。
