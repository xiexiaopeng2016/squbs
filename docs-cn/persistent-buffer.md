# 持久缓冲区

`PersistentBuffer`是一系列实用的Akka Streams流组件中的第一个。它的工作方式类似于Akka Streams缓冲区，不同之处在于，缓冲区的内容存储在一系列内存映射文件中，这些文件位于构建`PersistentBuffer`时给出的目录中。这允许缓冲区大小几乎是无限的，不使用JVM堆进行存储，并且同时具有每秒一百万条消息范围内的极佳性能。

## 依赖

要使持久缓冲区工作，需要以下依赖项：

```scala
"org.squbs" %% "squbs-pattern" % squbsVersion,
"net.openhft" % "chronicle-queue" % "4.16.5"
```

## 示例

以下示例显示了在流中的使用`PersistentBuffer`：

```scala
implicit val serializer = QueueSerializer[ByteString]()
val source = Source(1 to 1000000).map { n => ByteString(s"Hello $n") }
val buffer = new PersistentBuffer[ByteString](new File("/tmp/myqueue"))
val counter = Flow[Any].map( _ => 1L).reduce(_ + _).toMat(Sink.head)(Keep.right)
val countFuture = source.via(buffer.async).runWith(counter)

```

此版本显示相同的功能，在一个GraphDSL中：

```scala
implicit val serializer = QueueSerializer[ByteString]()
val source = Source(1 to 1000000).map { n => ByteString(s"Hello $n") }
val buffer = new PersistentBuffer[ByteString](new File("/tmp/myqueue"))
val counter = Flow[Any].map( _ => 1L).reduce(_ + _).toMat(Sink.head)(Keep.right)
val streamGraph = RunnableGraph.fromGraph(GraphDSL.create(counter) { implicit builder =>
  sink =>
    import GraphDSL.Implicits._
    source ~> buffer.async ~> sink
    ClosedShape
})
val countFuture = streamGraph.run()
```

## 背压

`PersistentBuffer`不会对上游产生背压。它将获取所有给它的流元素，并通过增加或旋转来增加存储队列文件的数量。它没有任何手段来确定缓冲区大小的限制或确定存储容量。下游背压根据Akka流和反应流的要求来确定。

如果`PersistentBuffer`阶段与下游得到了融合，`PersistentBuffer`不会缓冲，它实际上会产生背压。为了确保`PersistentBuffer`实际以自己的速度运行，在它后面添加一个`async`边界。

## 故障与恢复

由于它的持久性，`PersistentBuffer`可以从突然的流关闭，故障，JVM故障甚至潜在的系统故障中恢复。使用同一目录下的`PersistentBuffer`重新启动流，将开始发送存储在缓冲区中的元素，这些元素在新添加的元素之前还没有被消费。在上一次流失败或关闭时，已从缓冲区中消费但尚未完成处理的元素将会丢失。

由于缓冲区依靠本地存储，spindle或SSD，性能和持久性也取决于这个存储的持久性。系统故障或存储损坏可能导致缓冲区中的所有元素全部丢失。因此，重要的是要了解并假定此缓冲区的持久性不在数据库或其他脱离主机的持久性存储级别，以换取更高的性能。

Akka Streams在内部分批处理请求并缓冲记录。 PersistentBuffer保证到达`onPush`的记录的恢复和持久性，在Akka Stream阶段的内部缓冲区中尚未到达`onPush`的记录将在故障期间丢失。

## 提交保证

如果发生意外故障，从`PersistentBuffer`阶段发出但尚未到达`sink`的元素将丢失。有时，可能需要避免这种数据丢失。在这种情况下，在`sink`之前使用一个`commit`阶段可能会有所帮助。要添加一个`commit`阶段，使用`PersistentBufferAtLeastOnce`代替。请参阅以下示例，了解`commit`阶段用法：

 Please see below example for `commit` stage usage:  

```scala
implicit val serializer = QueueSerializer[ByteString]()
val source = Source(1 to 1000000).map { n => ByteString(s"Hello $n") }
val tempPath = new File("/tmp/myqueue")
val config = ConfigFactory.parseMap {
    Map(
      "persist-dir" -> s"${tempPath.getAbsolutePath}"
    )
  }
val buffer = new PersistentBufferAtLeastOnce[ByteString](config)
val commit = buffer.commit[ByteString]
val flowSink = // do some transformation or a sink flow with expected failure
val counter = Flow[Any].map( _ => 1L).reduce(_ + _).toMat(Sink.head)(Keep.right)
val streamGraph = RunnableGraph.fromGraph(GraphDSL.create(counter) { implicit builder =>
  sink =>
    import GraphDSL.Implicits._
    // ensures that records are reprocessed when something fails at tranform flow
    source ~> buffer ~> flowSink ~> commit ~> sink 
    ClosedShape
})
val countFuture = streamGraph.run()
```

请注意，`commit`并不能防止`sink`内部缓冲区中的消息丢失(或`commit`之后的任何其他阶段)。

### 提交顺序

`commit`阶段通常应按索引顺序接收元素。然而，流中的一个潜在的bug可能会导致一个元素被删除或没有按顺序到达`commit`阶段。默认的`commit-order-policy`被设置为`lenient`，以便让流在这种情况下继续运行。您可以将它设置为`strict`，以抛出一个`CommitOrderException`，并让`Supervision.Decider`决定采取什么行动。

## 空间管理

用于持久化队列的典型目录如下：

```
$ ls -l
-rw-r--r--  1 squbs_user     110054053  83886080 May 17 20:00 20160518.cq4
-rw-r--r--  1 squbs_user     110054053      8192 May 17 20:00 tailer.idx
```

一旦所有读者都成功处理读取队列的操作，队列文件将被自动删除。

## 配置

可以通过仅传递保存所有缺省配置的持久目录的一个位置来创建队列。这在上面的所有例子中都可以看到。或者，也可以通过在构造时传递`Config`对象来创建它。`Config`对象是一个标准的[HOCON](https://github.com/typesafehub/config/blob/master/HOCON.md)配置。下面的例子展示了如何使用`Config`来构造一个`PersistentBuffer`：

```scala
val configText =
  """
    | persist-dir = /tmp/myQueue
    | roll-cycle = xlarge_daily
    | wire-type = compressed_binary
    | block-size = 80m
  """.stripMargin
val config = ConfigFactory.parseString(configText)

// Construct the buffer using a Config.
val buffer = new PersistentBuffer[ByteString](config)
```

以下配置属性用于`PersistentBuffer`

```sh
persist-dir = /tmp/myQueue # Required
roll-cycle = daily         # Optional, defaults to daily
wire-type = binary         # Optional, defaults to binary
block-size = 80m           # Optional, defaults to 64m
index-spacing = 16k        # Optional, defaults to roll-cycle's spacing 
index-count = 16           # Optional, defaults to roll-cycle's count
commit-order-policy = lenient # Optional, default to lenient
```

滚动周期可以用小写字母或大写字母指定。支持的`roll-cycle`值如下:

Roll Cycle  | Capacity
------------|---------
MINUTELY    | 每分钟6400万个条目
HOURLY      | 每小时2.56亿个条目
SMALL_DAILY | 每天5.12亿个条目
DAILY       | 每天40亿个条目
LARGE_DAILY | 每天320亿个条目
XLARGE_DAILY| 每天2万亿个条目
HUGE_DAILY  | 每天256万亿个条目

可以使用大写或小写指定Wire-type。支持的值`wire-type`如下：

* TEXT
* BINARY
* FIELDLESS_BINARY
* COMPRESSED_BINARY
* JSON
* RAW
* CSV

内存大小诸如`block-size`和`index-spacing`是根据[在HOCON规范中定义的内存大小格式](https://github.com/typesafehub/config/blob/master/HOCON.md#size-in-bytes-format)指定的。

## 序列化

需要为`PersistentBuffer[T]`隐式提供一个`QueueSerializer[T]`，如上面的例子所示：

```scala
implicit val serializer = QueueSerializer[ByteString]()
```

`QueueSerializer[T]()`调用为您的目标类型生成一个序列化器。它取决于底层基础设施的序列化和反序列化。

### 实现一个序列化器

为了在队列中控制细粒度的持久格式，您可能需要实现自己的序列化器，如下所示:

```scala
case class Person(name: String, age: Int)

class PersonSerializer extends QueueSerializer[Person] {

  override def readElement(wire: WireIn): Option[Person] = {
    for {
      name <- Option(wire.read().`object`(classOf[String]))
      age <- Option(wire.read().int32)
    } yield { Person(name, age) }
  }

  override def writeElement(element: Person, wire: WireOut): Unit = {
    wire.write().`object`(classOf[String], element.name)
    wire.write().int32(element.age)
  }
}
```

要使用这个序列化器，只需在构造`PersistentBuffer`之前隐式地声明它，如下所示:

```scala
implicit val serializer = new PersonSerializer()
val buffer = new PersistentBuffer[Person](new File("/tmp/myqueue")
```

## 广播缓冲区

`BroadcastBuffer`是持久缓冲区的变体。除了流元素被广播到多个输出端口外，它的工作原理与`PersistentBuffer`类似。因此，它是缓冲区和广播阶段的组合。配置使用一个名为`output-ports`的附加参数，该参数指定输出端口的数量。

当要根据下游需求的速度以独立的速率从每个输出端口发出流元素时，特别需要广播缓冲区。

```scala
val configText =
  """
    | persist-dir = /tmp/myQueue
    | roll-cycle = xlarge_daily
    | wire-type = compressed_binary
    | block-size = 80m
    | output-ports = 3
  """.stripMargin
val config = ConfigFactory.parseString(configText)

// Construct the buffer using a Config.
val bcBuffer = new BroadcastBuffer[ByteString](config)
``` 

## 示例

```scala
implicit val serializer = QueueSerializer[ByteString]()

val in = Source(1 to 100000)
val flowCounter = Flow[Any].map(_ => 1L).reduce(_ + _).toMat(Sink.head)(Keep.right)

val streamGraph = RunnableGraph.fromGraph(GraphDSL.create(flowCounter) { implicit builder =>
      sink =>
        import GraphDSL.Implicits._
        val buffer = new BroadcastBufferAtLeastOnce[ByteString](config)
        val commit = buffer.commit[ByteString]
        val bcBuffer = builder.add(buffer.async)
        val mr = builder.add(merge)
        in ~> transform ~> bcBuffer ~> commit ~> mr ~> sink
                           bcBuffer ~> commit ~> mr
                           bcBuffer ~> commit ~> mr
        ClosedShape
    })
    
val countFuture = streamGraph.run()
```
## Credits

`PersistentBuffer`利用了[Chronicle-Queue](https://github.com/OpenHFT/Chronicle-Queue)4.x作为高性能内存映射队列持久性。
