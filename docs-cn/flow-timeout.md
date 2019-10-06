# 超时阶段

### 概述

一些流用例可能需要在限定的时间内处理流中的每个消息，否则发送超时失败消息。squbs引入了`Timeout`Akka流阶段，为流添加超时功能。

### 依赖

将以下依赖项添加到您的`build.sbt`或scala构建文件:

```
"org.squbs" %% "squbs-ext" % squbsVersion
```

### 用法

超时功能以`BidiFlow`的形式提供，可以通过`join`操作符连接到一个流。超时`BidiFlow`发送一个`Try`到下游：

   * 如果输出`msg`是在包装的流的超时范围内提供的，则将`Success(msg)`往下传递。
   * 否则，一个`Failure(FlowTimeoutException())`将推送到下游  


如果没有下游需求，则不会将超时消息下推。此外，除非有下游需求，否则不会进行超时检查。因此，根据需求场景，即使在超时时间之后，一些消息也可能作为`Success`发送。

超时精度最好为10ms，以避免不必要的计时器调度周期。

对于保证消息排序的流和不保证消息排序的流，都支持超时特性。对于不保证消息排序的流，包装的流需要携带一个`context`来惟一地标识每个元素。虽然`BidiFlow`的用法对于两种类型的流都是相同的但是`BidiFlow`的创建是通过两个不同的api完成的。

#### 具有消息顺序保证的流

`TimeoutOrdered`用于创建一个超时`BidiFlow`来包装保持消息顺序的流。请注意，在消息顺序保证的情况下，可能会出现head-of-line阻塞的情况，一个较慢的元素可能会导致后续元素超时。

##### Scala

```scala
val timeout = TimeoutOrdered[String, String](1 second)
val flow = Flow[String].map(s => findAnEnglishWordThatStartWith(s))
Source("a" :: "b" :: "c" :: Nil)
  .via(timeout.join(flow))
  .runWith(Sink.seq)
```      

##### Java

```java
final Duration duration = Duration.ofSeconds(1);

final BidiFlow<String, String, String, Try<String>, NotUsed> timeout = TimeoutOrdered.create(duration);

final Flow<String, String, NotUsed> flow =
    Flow.<String>create().map(s -> findAnEnglishWordThatStartWith(s));
        
Source.from(Arrays.asList("a", "b", "c"))
    .via(timeout.join(flow))
    .runWith(Sink.seq(), mat);
```   

#### 没有消息顺序保证的流

`Timeout`用于创建一个超时`BidiFlow`来包装不保证消息顺序的流。要惟一地标识每个元素及其对应的定时标记，应用程序定义的任何类型的`context`都需要由封装的流携带。需求是`Context`本身或来自`Context`的映射应该能够唯一地标识一个元素(查看[用于惟一Id的映射的上下文](#context-to-unique-id-mapping)章节了解更多细节)。

##### Scala

`Timeout[In, Out, Context](timeout: FiniteDuration)`用于创建一个无序的超时`BidiFlow`。
这个`BidiFlow`可以与任何接收一个`(In, Context)`并输出一个`(Out, Context)`的流连接。

```scala   
val timeout = Timeout[String, String, UUID](1 second) 
val flow = Flow[(String, UUID)].mapAsyncUnordered(10) { elem =>
  (ref ? elem).mapTo[(String, UUID)]
}

Source("a" :: "b" :: "c" :: Nil)
  .map { s => (s, UUID.randomUUID()) }
  .via(timeout.join(flow))
  .runWith(Sink.seq)
```

##### Java

以下API用于创建无序超时`BidiFlow`：

```java
public class Timeout {
	public static <In, Out, Context> BidiFlow<Pair<In, Context>, 
	                                          Pair<In, Context>,
	                                          Pair<Out, Context>,
	                                          Pair<Try<Out>, Context>,
	                                          NotUsed>
	create(FiniteDuration timeout);
}
```

这个`BidiFlow`可以与任何接受一个`akka.japi.Pair<In, Context>`并输出一个`akka.japi.Pair<Out, Context>`的流连接。

```java
final Duration duration = Duration.ofSeconds(1);

final BidiFlow<Pair<String, UUID>,
               Pair<String, UUID>,
               Pair<String, UUID>,
               Pair<Try<String>, UUID>, NotUsed> timeout = Timeout.create(duration);    

final Flow<Pair<String, UUID>, Pair<String, UUID>, NotUsed> flow =
        Flow.<Pair<String, UUID>>create()
                .mapAsyncUnordered(10, elem -> ask(ref, elem, 5000))
                .map(elem -> (Pair<String, UUID>)elem);

Source.from(Arrays.asList("a", "b", "c"))
        .map(s -> new Pair<>(s, UUID.randomUUID()))
        .via(timeout.join(flow))
        .runWith(Sink.seq(), mat);    
```

##### 用于唯一ID映射的上下文

`Context`本身可以用作唯一的id。但是，在许多场景中，`Context`包含的内容比惟一id本身更多，或者惟一id可能作为映射从`Context`检索。squbs允许不同的选项提供一个唯一的id：

   * `Context` 它本身是一种可以用作唯一id的类型，例如，`Int`，`Long`，`java.util.UUID`
   * `Context` 扩展`UniqueId.Provider`并实现`def uniqueId`
   * `Context` 是用`UniqueId.Envelope`包装的
   * `Context` 通过调用函数映射到唯一ID


使用前三个选项，可以直接通过上下文检索唯一的id。对于最后一个选项，`Timeout`允许通过`TimeoutSettings`传入到一个函数中：

###### Scala

```scala
case class MyContext(s: String, uuid: UUID)

val settings =
  TimeoutSettings[String, String, MyContext](1 second)
    .withUniqueIdMapper(context => context.uuid)

val timeout = Timeout(settings)

val flow = Flow[(String, MyContext)].mapAsyncUnordered(10) { elem =>
  (ref ? elem).mapTo[(String, MyContext)]
}

Source("a" :: "b" :: "c" :: Nil)
  .map( _ -> MyContext("dummy", UUID.randomUUID))
  .via(timeout.join(flow))
  .runWith(Sink.seq) 
```

###### Java

```java
class MyContext {
    private String s;
    private UUID uuid;

    public MyContext(String s, UUID uuid) {
        this.s = s;
        this.uuid = uuid;
    }

    public UUID uuid() {
        return uuid;
    }
}

final Duration duration = Duration.ofSeconds(1);

final TimeoutSettings settings =
    TimeoutSettings.<String, String, Context>create(duration)
            .withUniqueIdMapper(context -> context.uuid);

final BidiFlow<Pair<String, MyContext>,
               Pair<String, MyContext>,
               Pair<String, MyContext>,
               Pair<Try<String>, MyContext>,
               NotUsed> timeout = Timeout.create(settings);

final Flow<Pair<String, MyContext>, Pair<String, MyContext>, NotUsed> flow =
        Flow.<Pair<String, MyContext>>create()
                .mapAsyncUnordered(10, elem -> ask(ref, elem, 5000))
                .map(elem -> (Pair<String, MyContext>)elem);

Source.from(Arrays.asList("a", "b", "c"))
        .map(s -> new Pair<>(s, new MyContext("dummy", UUID.randomUUID())))
        .via(timeout.join(flow))
        .runWith(Sink.seq(), mat);
```

##### 清理回调

`Timeout`还提供了一个清理回调函数，可以通过`TimeoutSettings`传入。对于已经被认为超时的已发出的元素，将调用此函数。

一个示例，此功能的用例在Akka Http客户端使用`Timeout`时。如[请求/响应实体的流本质的影响](http://doc.akka.io/docs/akka-http/current/scala/http/implications-of-streaming-http-entity.html)所述，所有http响应必须消费或丢弃。通过传递一个清理回调以在超时请求完成时丢弃它们，我们可以避免阻塞流。

###### Scala
```scala
val akkaHttpDiscard = (response: HttpResponse) => response.discardEntityBytes()

val settings =
  TimeoutSettings[HttpRequest, HttpResponse, Context](1 second)
    .withCleanUp(response => response.discardEntityBytes())

val timeout = Timeout(settings)
```

###### Java
```java
final Duration duration = Duration.ofMillis(20);

final TimeoutSettings settings =
    TimeoutSettings.<HttpRequest, HttpResponse, Context>create(duration)
            .withCleanUp(httpResponse -> httpResponse.discardEntityBytes(materializer));

final BidiFlow<Pair<HttpRequest, UUID>, 
               Pair<HttpRequest, UUID>, 
               Pair<HttpResponse, UUID>, 
               Pair<Try<HttpResponse>, UUID>, 
               NotUsed> timeout = Timeout.create(settings);
```
