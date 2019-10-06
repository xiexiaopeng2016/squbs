# 永久流

### 概述

`PerpetualStream`允许声明一个流，该流在服务器启动时启动，并在服务器停止时优雅地停止，而不会丢失消息。它通常用于来自Kafka或JMS的消息消费者，但也用作通过HTTP请求接收的多个流的数据的整合点。

`PerpetualStream`可以通过各种方式进行自定义以满足您的流的需求。在下面列出的部分中讨论了这些内容：

* [基本用途](#basic-use)
* [覆盖生命周期状态以运行流](#override-lifecycle-state-to-run-the-stream)
* [Shutdown重写](#shutdown-overrides)
* [Kill开关重写](#kill-switch-overrides)
* [接收并转发消息到流 Receiving](#receiving-and-forwarding-a-message-to-the-stream)
* [处理流错误](#handling-stream-errors)
* [连接永久流与HTTP流](#connecting-a-perpetual-stream-with-an-http-flow)

### 依赖

`PerpetualStream`是squbs内核的一部分。通常，您不需要添加额外的依赖项。这些类是以下依赖项的一部分：

```scala
"org.squbs" %% "squbs-unicomplex" % squbsVersion
```

### 用法

这些`PerpetualStream`被暴露为`PerpetualStream`特质(Scala)和`AbstractPerpertualStream`的抽象类(Java)。为简便起见，我们将两者都称为`PerpetualStream`。

#### 基本用途

##### Scala

流利用`PerpetualStream`将希望具体化为某些已知类型，从而使`PerpetualStream`中的钩子能够无缝工作，以最小的自定义重写。选项包括：

* 具体化为一个`Future[_]`，意味着一个任何类型的`future`。在这种情况下，来自`PerpetualStream`的共享`killSwitch`应该被嵌入或`shutdown()`将应该被重写。
* 具体化为一个`(KillSwitch, Future[_])`元组。`KillSwitch`将用于启动流的关闭。
* 具体化为一个`List`或任何`Product`(`List`，`Tuple`都是`Product`的子类型)其中第一个元素是一个`KillSwitch`，最后一个元素是一个`Future`。

仍然可以使用具有不同物化值的流，但是`shutdown()`需要被重写。

表现良好的流的常见例子如下：

```scala
class WellBehavedStream extends PerpetualStream[Future[Done]] {

  def generator = Iterator.iterate(0) { p => 
    if (p == Int.MaxValue) 0 else p + 1 
  }

  val source = Source.fromIterator(generator _)

  val ignoreSink = Sink.ignore
  
  override def streamGraph = RunnableGraph.fromGraph(GraphDSL.create(ignoreSink) {
    implicit builder =>
      sink =>
        import GraphDSL.Implicits._
        source ~> killSwitch.flow[Int] ~> sink
        ClosedShape
  })
}
```

另外，以下代码显示了另一个符合`PerpetualStream`物化，其第一个元素为`KillSwitch`：

```scala
class WellBehavedStream2 extends PerpetualStream[(KillSwitch, Future[Done])] {

  def generator = Iterator.iterate(0) { p => 
    if (p == Int.MaxValue) 0 else p + 1 
  }

  val source = Source.fromIterator(generator _)

  val ignoreSink = Sink.ignore
  
  override def streamGraph = RunnableGraph.fromGraph(
    GraphDSL.create(KillSwitch.single[Int], ignoreSink)((_,_)) { implicit builder =>
      (kill, sink) =>
        import GraphDSL.Implicits._
        source ~> kill ~> sink
        ClosedShape
  })
}
```

就是这样。这些流的行为良好，因为它们实现为接收器的物化值，在第一个示例中为一个`Future[Done]`，在第二个示例中为`(KillSwitch, Future[Done])`。

##### Java

Streams making use of `AbstractPerpetualStream` will want to materialize to certain known types, allowing the hooks in `AbstractPerpetualStream` to work seamlessly with minimal amount of custom overrides. The options are:

* Materialize to a `CompletionStage<?>`, meaning a Java `CompletionStage` of any type. In this case the shared `killSwitch` from `AbstractPerpetualStream` should be embedded or `shutdown()` would need to be overridden.
* Materialize to a `Pair<KillSwitch, CompletionStage<?>>`. The `KillSwitch` will be used for initiating the shutdown of the stream.
* Materialize to a `java.util.List` where the first element is a `KillSwitch` and the last element is a `CompletionStage`.

Streams with different materialized values can still be used but `shutdown()` needs to be overridden.

Common examples for well behaved streams can be seen below:

```java
public class WellBehavedStream extends AbstractPerpetualStream<CompletionStage<Done>> {

    Sink<Integer, CompletionStage<Done>> ignoreSink = Sink.ignore();
  
    @Override
    public RunnableGraph<CompletionStage<Done>>> streamGraph() {
        return RunnableGraph.fromGraph(GraphDSL.create(ignoreSink, (builder, sink) -> {
            SourceShape<Integer> source = builder.add(
                    Source.unfold(0, i -> {
                        if (i == Integer.MAX_VALUE) {
                            return Optional.of(Pair.create(0, i));
                        } else {
                            return Optional.of(Pair.create(i + 1, i));
                        }
                    })
            );
                        
            FlowShape<Integer, Integer> killSwitch= builder.add(killSwitch().<Integer>flow());

            builder.from(source).via(killSwitch).to(sink);
            
            return ClosedShape.getInstance();
        }));
    }
```

Alternatively, the following code shows another conformant `PerpetualStream` materializing its first element as `KillSwitch`:

```java
public class WellBehavedStream2 extends
        AbstractPerpetualStream<Pair<KillSwitch, CompletionStage<Done>>> {

    Sink<Integer, CompletionStage<Done>> ignoreSink = Sink.ignore();
  
    @Override
    public RunnableGraph<Pair<KillSwitch, CompletionStage<Done>>>> streamGraph() {
        return RunnableGraph.fromGraph(GraphDSL.create(KillSwitches.<Integer>single(),
            ignoreSink, Pair::create, (builder, kill, sink) -> {
                SourceShape<Integer> source = builder.add(
                        Source.unfold(0, i -> {
                            if (i == Integer.MAX_VALUE) {
                                return Optional.of(Pair.create(0, i));
                            } else {
                                return Optional.of(Pair.create(i + 1, i));
                            }
                        })
                );

                builder.from(source).via(kill).to(sink);
            
                return ClosedShape.getInstance();
            }));
    }
```

That's it. These streams are well behaved because they materialize to the sink's materialized value, which is a `CompletionStage<Done>` in the first example, or a `Pair<KillSwitch, CompletionStage<Done>>` in the second one.

#### 覆盖生命周期状态来运行流

可能会有这样的情况，一个流需要在一个不同于`active`的生命周期中被具体化。在这种情况下，请覆盖`streamRunLifecycleState`，例如：

##### Scala

```scala
override lazy val streamRunLifecycleState: LifecycleState = Initializing
```

##### Java

```java
@Override
public LifecycleState streamRunLifecycleState() {
    return Initializing.instance();
}
```

#### Shutdown重写

有时无法定义行为良好的流。例如，`Sink`可能没有物化为`Future`或`CompletionStage`或者您需要在关闭时进行进一步清理。为此，可以重写`shutdown`，按以下代码：

##### Scala

```scala
override def shutdown(): Future[Done] = {
  // Do all your cleanup
  // For safety, call super
  super.shutdown()
  // The Future from super.shutdown may not mean anything.
  // Feel free to create your own future that identifies the
  // stream being done. Return your Future instead.
}
```

##### Java

```java
@Override
public CompletionStage<Done> shutdown() {
    // Do all your cleanup
    // For safety, call super
    super.shutdown();
    // The Future from super.shutdown may not mean anything.
    // Feel free to create your own future that identifies the
    // stream being done. Return your Future instead.
}
```

`shutdown`需要执行以下操作：

1. 动流的关闭。
2. 进行任何其他清理。
3. 返回future，其将在流处理完毕后完成。

注意：始终建议调用`super.shutdown`。这个调用没有任何伤害或其他副作用。

##### Alternate Shutdown Mechanisms

`source`可能不会物化为`KillSwitch`，并且提供了比使用`killSwitch`更好的关闭方式。在这种情况下，只需使用`source`的关闭机制并覆盖`shutdown`来启动源的关闭。`killSwitch`仍未使用。

#### Kill开关重写

如果`killSwitch`需要在多个流之间共享，则可以重写`killSwitch`以反映共享的实例。

##### Scala

```scala
override lazy val killSwitch = mySharedKillSwitch
```

##### Java

```java
@Override
public SharedKillSwitch killSwitch() {
    return KillSwitches.shared("myKillSwitch");
}
```

#### 接收和转发消息到流

有些流从actor消息获取输入。虽然某些流配置可以物化到源的`ActorRef`，但很难定位(address)该actor。由于`PerpetualStream`本身是一个actor，因此它可以具有众所周知的地址/路径并转发消息到流的源。为此，我们需要重写`receive`或`createReceive()`，如下所示：

##### Scala

```scala
override def receive = {
  case msg: MyStreamMessage =>
    val (sourceActorRef, _) = matValue
    sourceActorRef forward msg
}
```

##### Java

```java
@Override
public Receive createReceive() {
    return receiveBuilder()
            .match(MyStreamMessage.class, msg -> {
                ActorRef sourceActorRef = matValue().first();
                sourceActorRef.forward(msg, getContext());
            })
            .build();
}
```

#### 处理流错误

`PerpetualStream`默认行为恢复，在错误没有被流阶段未捕获时。导致错误的消息将被忽略。如果需要其他行为，则重写`decider`。

##### Scala

```scala
override def decider: Supervision.Decider = { t => 
  log.error("Uncaught error {} from stream", t)
  t.printStackTrace()
  Restart
}
```

##### Java

```java
@Override
public akka.japi.function.Function<Throwable, Supervision.Directive> decider() {
    return t -> {
        log().error("Uncaught error {} from stream", t);
        t.printStackTrace();
        return Supervision.restart();
    };
}
```

`Restart`将重新启动有错误的阶段，而不关闭流。请参[阅监督策略](http://doc.akka.io/docs/akka/current/scala/stream/stream-error.html#Supervision_Strategies)以了解可能的策略。

#### 连接永久流与HTTP流

Akka HTTP允许定义一个`Flow[HttpRequest, HttpResponse, NotUsed]`，每个http连接都会物化它。在某些情况下，应用程序需要将http流连接到一个长时间运行的流，该流只需要具体化一次(例如，发布到Kafka)。在这种情况下，Akka HTTP通过 [`MergeHub`](http://doc.akka.io/docs/akka/current/scala/stream/stream-dynamic.html#dynamic-fan-in-and-fan-out-with-mergehub-broadcasthub-and-partitionhub)启用端到端流。squbs提供了实用程序，使用`MergeHub`连接一个http流和一个`PerpetualStream`。

以下示例`PerpetualStream`实现 - 两个Scala和两个Java等价物，都使用`MergeHub`。类型参数`Sink[MyMessage, NotUsed]`描述了`RunnableGraph`实例的入口，其将被http流部(flow part)用作一个终点(`Sink`)，在下面的`HttpFlowWithMergeHub`中。首先是一个简单的逻辑框架：

##### Scala

```scala
class PerpetualStreamWithMergeHub extends PerpetualStream[Sink[MyMessage, NotUsed]] {

  override lazy val streamRunLifecycleState: LifecycleState = Initializing
  

  /**
    * Describe your graph by implementing streamGraph
    *
    * @return The graph.
    */
  override def streamGraph= MergeHub.source[MyMessage].to(Sink.ignore)
}
```

##### Java

```java
public class PerpetualStreamWithMergeHub extends AbstractPerpetualStream<Sink<MyMessage, NotUsed>> {

    @Override
    public LifecycleState streamRunLifecycleState() {
        return Initializing.instance();
    }

    /**
     * Describe your graph by implementing streamGraph
     *
     * @return The graph.
     */
    @Override
    public RunnableGraph<Sink<MyMessage, NotUsed>> streamGraph() {
        return MergeHub.of(MyMessage.class).to(Sink.ignore());
    }
}
```

从外部预期的角度(通过http流)这个类被视为终端`Sink[MyMessage, NotUsed]`，这意味着`PerpetualStreamWithMergeHub`期望在其入口接收`MyMessage`，并不会放出任何东西，即它的出口被堵塞了。
从内部来看，`MergeHub`是`MyMessage`的源。这些信息被传递给 `Sink.ignore`，它们什么都不是。

`MergeHub.source[MyMessage]`生成运行时实例，具有类型为`Sink[MyMessage, NotUsed]`的入口，它们符合`PerpetualStream[Sink[MyMessage, NotUsed]]`类型参数。`.to(Sink.ignore)`使用一个堵塞的出口完成或"关闭"这个`Shape`。最终结果是一个`RunnableGraph[Sink[MyMessage, NotUsed]]`实例。

使用GraphDSL的更复杂的例子：

##### Scala

```scala
final case class MyMessage(ip:String, ts:Long)
final case class MyMessageEnrich(ip:String, ts:Long, enrichTs:List[Long])

class PerpetualStreamWithMergeHub extends PerpetualStream[Sink[MyMessage, NotUsed]]  {

  override lazy val streamRunLifecycleState: LifecycleState = Initializing
  
  // inlet - destination for MyMessage messages
  val source = MergeHub.source[MyMessage]
 
  //outlet - discard messages
  val sink = Sink.ignore
  
  //flow component, which supposedly does something to MyMessage
  val preprocess = Flow[MyMessage].map{inMsg =>
      val outMsg = MyMessageEnrich(ip=inMsg.ip, ts = inMsg.ts, enrichTs = List.empty[Long])
      println(s"Message inside stream=$inMsg")
      outMsg
  }
  
    // building a flow based on another flow, to do some dummy enrichment
  val enrichment = Flow[MyMessageEnrich].map{inMsg=>
      val outMsg = MyMessageEnrich(ip=inMsg.ip.replaceAll("\\.","-"), ts = inMsg.ts, enrichTs = System.currentTimeMillis()::inMsg.enrichTs)
      println(s"Enriched Message inside enrich step=$outMsg")
      outMsg
  }
    

  /**
    * Describe your graph by implementing streamGraph
    *
    * @return The graph.
    */
  override def streamGraph: RunnableGraph[Sink[MyMessage, NotUsed]] = RunnableGraph.fromGraph(
    GraphDSL.create(source) { implicit builder=>
        input =>  
          import GraphDSL.Implicits._
            
          input ~> killSwitch.flow[MyMessage] ~> preprocess ~> enrichment ~> sink
          
          ClosedShape
      })
}
```

##### Java

```java
public class PerpetualStreamWithMergeHub extends AbstractPerpetualStream<Sink<MyMessage, NotUsed>> {

    // inlet - destination for MyMessage messages
    Source<MyMessage, Sink<MyMessage, NotUsed>> source = MergeHub.of(MyMessage.class);

    @Override
    public LifecycleState streamRunLifecycleState() {
        return Initializing.instance();
    }

    /**
     * Describe your graph by implementing streamGraph
     *
     * @return The graph.
     */
    @Override
    public RunnableGraph<Sink<MyMessage, NotUsed>> streamGraph() {
        return RunnableGraph.fromGraph(GraphDSL.create(source, (builder, input) -> {
            
            FlowShape<MyMessage, MyMessage> killSwitch = builder.add(killSwitch().<MyMessage>flow());

            //flow component, which supposedly does something to MyMessage
            FlowShape<MyMessage, MyMessageEnrich> preProcess = builder.add(Flow.<MyMessage>create().map(inMsg -> {
                MyMessageEnrich outMsg = new MyMessageEnrich(inMsg.ip, inMsg.ts, new ArrayList<>());
                System.out.println("Message inside stream=" + inMsg);
                return outMsg;
            }));

            // building a flow based on another flow, to do some dummy enrichment
            FlowShape<MyMessageEnrich, MyMessageEnrich> enrichment =
                    builder.add(Flow.<MyMessageEnrich>create().map(inMsg -> {
                        inMsg.enrichTs.add(System.currentTimeMillis());
                        MyMessageEnrich outMsg = new MyMessageEnrich(inMsg.ip.replaceAll("\\.","-"),
                                inMsg.ts, inMsg.enrichTs);
                        System.out.println("Enriched Message inside enrich step=" + outMsg);
                        return outMsg;
                    }));

            //outlet - discard messages
            SinkShape<Object> sink = builder.add(Sink.ignore());

            builder.from(input).via(killSwitch).via(preProcess).via(enrichment).to(sink);

            return ClosedShape.getInstance();
        }));
    }
}
```

让我们看看所有的部分是如何落在一个地方：`streamGraph`期望返回`RunnableGraph`，其类型参数与`PerpetualStream[Sink[MyMessage, NotUsed]]`中描述的相同。我们的`source`是一个`MergeHub`，它期望接收一个`MyMessage`，这使得它的物化(运行时)类型为`Sink[MyMessage,NotUsed]`。我们的图形是从`source`开始构建的，通过将其作为参数传递给`GraphDSL.create(s:Shape)`构造函数。结果是一个`RunnableGraph[Sink[MyMessage, NotUsed]]`的实例，它是一个`ClosedShape`，带有`Sink`入口和被堵塞的出口。

查看此示例时，可能会混淆的部分是将`Sink`和`source`名称混合起来以引用相同的名称。它在英语中看起来有点奇怪。让我们再次使用外部和内部的预期解释：从外部预期来看，我们的组件被看作是一个`Sink[MyMessage, NotUsed]`。这是通过使用`MergeHub`完成的，从内部预期看，它是消息的一个来源，因此`val`将其命名为`source`。

相应地，在我们需要发出一些东西的时候，我们的`val sink`实际上会是一个带有类型`Source[MyMessage, NotUsed]`输出的形状。

让我们在`squbs-meta.conf`中添加上述`PerpetualStream`。有关更多详细信息，请参见[Well Known Actors](bootstrap.md#well-known-actors)。

```
cube-name = org.squbs.stream.mycube
cube-version = "0.0.1"
squbs-services = [
  {
    class-name = org.squbs.stream.HttpFlowWithMergeHub
    web-context = mergehub
  }
]
squbs-actors = [
  {
    class-name = org.squbs.stream.PerpetualStreamWithMergeHub
    name = perpetualStreamWithMergeHub
  }
]
```

##### Scala

HTTP的`FlowDefinition`可以像下面这样连接到`PerpetualStream`，通过扩展`PerpetualStreamMatValue`和使用`matValue`方法。`PerpetualStreamMatValue`的类型参数描述了在HTTP流和`MergeHub`之间流动的数据类型。(上面`PerpetualStreamWithMergeHub`的所有版本都期望接收`MyMessage`，即都具有类型`Sink[MyMessage, NotUsed]`的入口)。

```scala
class HttpFlowWithMergeHub extends FlowDefinition with PerpetualStreamMatValue[MyMessage] {

  override val flow: Flow[HttpRequest, HttpResponse, NotUsed] =
    Flow[HttpRequest]
      .mapAsync(1)(Unmarshal(_).to[MyMessage])
      .alsoTo(matValue("/user/mycube/perpetualStreamWithMergeHub"))
      .map { myMessage => HttpResponse(entity = s"Received Id: ${myMessage.id}") }
}
```

##### Java

HTTP的`FlowDefinition`可以像下面这样连接到`PerpetualStream`，通过直接扩展`FlowToPerpetualStream`而不是`FlowDefinition`。请注意`FlowToPerpetualStream`**是一个**`FlowDefinition`。我们使用`matValue`方法作为接收，将HTTP消息发送到`PerpetualStream`中定义的`MergeHub`。

```java
class HttpFlowWithMergeHub extends FlowToPerpetualStream {

    private final Materializer mat = ActorMaterializer.create(context().system());
    private final MarshalUnmarshal mu = new MarshalUnmarshal(context().system().dispatcher(), mat);

    @Override
    public Flow<HttpRequest, HttpResponse, NotUsed> flow() {
        return Flow.<HttpRequest>create()
                .mapAsync(1, req -> mu.apply(unmarshaller(MyMessage.class), req.entity()))
                .alsoTo(matValue("/user/mycube/perpetualStreamWithMergeHub"))
                .map(myMessage -> HttpResponse.create().withEntity("Received Id: " + myMessage.ip));
    }
}
```

让我们看看这里发生了什么：
`matValue`方法查找`/user/mycube/perpetualStreamWithMergeHub`下注册的`RunnableGraph`组件。这恰好是我们的`PerpetualStreamWithMergeHub`的引导实例。`alsoTo`期望`matValue`的结果是`MyMessage`的`Sink`，即`Sink[MyMessage, NotUsed]`。正如我们在上面看到的，这正是`PerpetualStreamWithMergeHub.streamGraph`将产生的。(记住我们的两个展望：这里`alsoTo`从外部展望`PerpetualStreamWithMergeHub`，看到的是`Sink[MyMessage, NotUsed]`。)
