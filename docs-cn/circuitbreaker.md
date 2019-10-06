# 断路器

### 概述

Akka Streams和Akka HTTP是构建高度弹性系统的出色技术。它们提供了背压，以确保您不会使系统过载，并且一个组件的运行速度变慢不会导致其工作队列堆积并最终导致内存泄漏。但是，我们需要另一种保护措施，以确保我们的服务在出现外部或内部故障时保持响应，并具有替代路径来满足此类故障所引起的请求/消息。我们可以选择尝试另一个服务或获取缓存的结果，而不是对流和整个系统进行背压。

squbs引入了`CircuitBreaker` Akka Streams `GraphStage`，用于为流提供断路器功能。

### 依赖

将以下依赖项添加到您的`build.sbt`或scala构建文件：

```
"org.squbs" %% "squbs-ext" % squbsVersion
```

### 用法

断路器功能是由`BidiFlow`提供的，可以通过`join`操作符连接到流。`CircuitBreaker`可能会改变信息的顺序，因此需要携带`Context`。此外，它需要能够唯一地识别其每个元素的内部机制。需求是`Context`本身或来自`Context`的映射应该能够唯一地标识一个元素(查看[上下文用于惟一Id映射](#context-to-unique-id-mapping)章节了解更多细节)。

与`Context`一起，一个`Try`被推向下游：

线路(Circuit)是`Closed`:

   * 如果超时的包装流提供了输出`msg`，则`(Success(msg), context)`被传递到下游。
   * 否则，一个`(Failure(FlowTimeoutException()), context)`被传递到下游。


线路是`Open`:

   * 如果提供了一个`fallback`函数的结果，将与`context`一起被传递到下游
   * 否则，一个`(Failure(CircuitBreakerOpenException()), context)`被传递到下游


线路是`HalfOpen`:
   
   * 让第一个请求/元素通过包装的流，并且行为应与`Closed`状态相同。
   * 其余元素短路，其行为与`Open`状态相同。 	

断路器的状态保持在一个`CircuitBreakerState`实现中。`AtomicCircuitBreakerState`的默认实现是基于`Atomic`变量，这使得它可以跨多个流实现并发更新。

##### Scala

```scala
import org.squbs.streams.circuitbreaker.CircuitBreakerSettings

val state = AtomicCircuitBreakerState("sample", 2, 100 milliseconds, 1 second)
val settings = CircuitBreakerSettings[String, String, UUID](state)
val circuitBreaker = CircuitBreaker(settings)

val flow = Flow[(String, UUID)].mapAsyncUnordered(10) { elem =>
  (ref ? elem).mapTo[(String, UUID)]
}

Source("a" :: "b" :: "c" :: Nil)
  .map(s => (s, UUID.randomUUID()))
  .via(circuitBreaker.join(flow))
  .runWith(Sink.seq)
```

##### Java

对于Java，使用来自`org.squbs.streams.circuitbreaker.japi`包的`CircuitBreakerSettings`：

```java
import org.squbs.streams.circuitbreaker.japi.CircuitBreakerSettings;

final CircuitBreakerState state =
        AtomicCircuitBreakerState.create(
                "sample",
                2,
                FiniteDuration.apply(100, TimeUnit.MILLISECONDS),
                FiniteDuration.apply(1, TimeUnit.SECONDS),
                system.dispatcher(),
                system.scheduler());

final CircuitBreakerSettings<String, String, UUID> settings = CircuitBreakerSettings.create(state);

final BidiFlow<Pair<String, UUID>,
               Pair<String, UUID>,
               Pair<String, UUID>,
               Pair<Try<String>, UUID>, NotUsed> circuitBreaker = CircuitBreaker.create(settings);

final Flow<Pair<String, UUID>, Pair<String, UUID>, NotUsed> flow =
        Flow.<Pair<String, UUID>>create()
                .mapAsyncUnordered(10, elem -> ask(ref, elem, 5000))
                .map(elem -> (Pair<String, UUID>)elem);


Source.from(Arrays.asList("a", "b", "c"))
        .map(s -> Pair.create(s, UUID.randomUUID()))
        .via(circuitBreaker.join(flow))
        .runWith(Sink.seq(), mat);
```

#### Fallback响应

`CircuitBreakerSettings`可选地采用回退功能，该功能在线路(circuit)为`Open`时被调用。

##### Scala

可以通过`withFallback`函数提供一个`In => Try[Out]`类型的函数：

```scala
import org.squbs.streams.circuitbreaker.CircuitBreakerSettings

val settings =
  CircuitBreakerSettings[String, String, UUID](state)
    .withFallback((elem: String) => Success("Fallback Response!"))
```

##### Java

可以通过`withFallback`函数提供一个`Function<In, Try<Out>>`类型的函数：

```java
import org.squbs.streams.circuitbreaker.japi.CircuitBreakerSettings;

CircuitBreakerSettings settings =
        CircuitBreakerSettings.<String, String, UUID>create(state)
                .withFallback(s -> Success.apply("Fallback Response!"));
```

#### 失败决定者

默认情况下，连接的`Flow`中的任何`Failure`都被认为是一个问题，并导致断路器故障计数增加。但是，`CircuitBreakerSettings`也接受一个可选的`failureDecider`来决定通过连接的`Flow`传递的元素是否被认为是失败的。例如，如果断路器与Akka HTTP流连接，一个包含状态码为500的内部服务器错误的`Success`HTTP响应应视为失败。

##### Scala

一个类型为`Try[Out] => Boolean`的函数可以通过`withFailureDecider`函数提供。下面是一个例子，与任何`Failure`消息一样，状态码`400`及以上的`Success``HttpResponse`也被视为失败：

```scala
import org.squbs.streams.circuitbreaker.CircuitBreakerSettings

val settings =
  CircuitBreakerSettings[HttpRequest, HttpResponse, UUID](state)
    .withFailureDecider(tryHttpResponse => tryHttpResponse.isFailure || tryHttpResponse.get.status.isFailure)
```

##### Java

一个类型为`Function<Try<Out>, Boolean>`的函数可以通过`withFailureDecider`函数提供。下面是一个例子，与任何`Failure`消息一样，状态码`400`及以上的`Success``HttpResponse`也被视为失败：

```java
import org.squbs.streams.circuitbreaker.japi.CircuitBreakerSettings;

CircuitBreakerSettings settings =
        CircuitBreakerSettings.<HttpRequest, HttpResponse, UUID>create(state)
                .withFailureDecider(
                        tryHttpResponse -> tryHttpResponse.isFailure() || tryHttpResponse.get().status().isFailure());
```

#### 从配置创建CircuitBreakerState

虽然`AtomicCircuitBreakerState`有用于配置的可编程API，但它也允许通过`Config`对象提供配置。`Config`可以部分地定义一些设置，其余的将退回到默认值。请查看[这里](../squbs-ext/src/main/resources/reference.conf)了解默认断路器配置。

##### Scala

```scala
val config = ConfigFactory.parseString(
  """
    |max-failures = 5
    |call-timeout = 50 ms
    |reset-timeout = 100 ms
    |max-reset-timeout = 2 seconds
    |exponential-backoff-factor = 2.0
  """.stripMargin)

val state = AtomicCircuitBreakerState("sample", config)
```

##### Java

```java
Config config = ConfigFactory.parseString(
        "max-failures = 5\n" +
        "call-timeout = 50 ms\n" +
        "reset-timeout = 100 ms\n" +
        "max-reset-timeout = 2 seconds\n" +
        "exponential-backoff-factor = 2.0");
        
final CircuitBreakerState state = AtomicCircuitBreakerState.create("sample", config, system);
```

#### 断路器跨具体化

请注意，在许多情况下，同一断路器实例用于同一流的多个具体化。对于这类场景，请确保使用一个可以并发修改的`CircuitBreakerState`实例。`AtomicCircuitBreakerState`的默认实现使用`Atomic`变量，可以跨多个具体化使用。将来可以介绍更多的实现。

#### 用于唯一ID映射的上下文

`Context`本身可以用作唯一的id。但是，在许多场景中，`Context`包含的内容比惟一id本身更多，或者惟一id可能作为映射从`Context`检索。squbs允许使用不同的选项来提供唯一的ID：

   * `Context` 它本身是一种可以用作唯一id的类型，例如，`Int`，`Long`，`java.util.UUID`
   * `Context` 扩展`UniqueId.Provider`并实现`def uniqueId`
   * `Context` 是用`UniqueId.Envelope`包装的
   * `Context` 通过调用函数映射到唯一ID


使用前三个选项，可以直接通过上下文检索唯一的id。对于最后一个选项，`CircuitBreakerSettings`允许提供一个函数。

##### Scala

可以通过`withUniqueIdMapper`函数提供一个`Context => Any`类型的函数：

```scala
import org.squbs.streams.circuitbreaker.CircuitBreakerSettings

case class MyContext(s: String, id: Long)

val settings =
  CircuitBreakerSettings[String, String, MyContext](state)
    .withUniqueIdMapper(context => context.id)
```

##### Java

可以通过`withUniqueIdMapper`函数提供一个`Function<Context, Any>`类型的函数：

```java
import org.squbs.streams.circuitbreaker.japi.CircuitBreakerSettings;

class MyContext {
    private String s;
    private long id;

    public MyContext(String s, long id) {
        this.s = s;
        this.id = id;
    }

    public long id() {
        return id;
    }
}

CircuitBreakerSettings settings =
        CircuitBreakerSettings.<String, String, MyContext>create(state)
                .withUniqueIdMapper(context -> context.id());
```

#### 通知

可以订阅一个`ActorRef`来接收所有`TransitionEvents`或它感兴趣的任何转换事件，例如`Closed`，`Open`，`HalfOpen`。

##### Scala

这是一个示例，注册一个`ActorRef`用于当线路切换到`Open`状态时接收事件：

```scala
state.subscribe(self, Open)
```

##### Java

这是一个示例，注册一个`ActorRef`用于当线路切换到`Open`状态时接收事件：

```java
state.subscribe(getRef(), Open.instance());
```

#### 指标

`CircuitBreakerState`保持Codahale指标:

* 成功次数
* 失败次数
* 短路次数

它也为断路器状态保持一个`Gauge`。

为了在实例之间区分指标，`CircuitBreakerState`实现需要传入一个名称。

`CircuitBreakerState`还允许传入一个`MetricRegistry`实例。如果没有传递`MetricRegistry`，它将在内部创建一个。

#### 线路Open状态对吞吐量的潜在影响

如果上游不控制吞吐量，一旦线路`Open`，流的吞吐量可能会暂时增加。下游需求将通过短路(short circuit)/回退(fallback)消息来处理，这可能(也可能不会)比它使用连接的`Flow`来处理一个元素所需的时间更少。为了消除这个问题，可以专门为断路器`Open`消息(或回退消息)应用一个节流阀(throttle)。
