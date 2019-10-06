# 重试阶段

### 概述

一些流用例可能需要在失败响应之后重试请求。squbs提供了一个`Retry`Akka流阶段，用于向需要为任何失败请求添加重试功能的流添加重试功能。

### 依赖

将以下依赖项添加到您的`build.sbt`或scala构建文件：

```
"org.squbs" %% "squbs-ext" % squbsVersion
```

### 用法

重试阶段功能通过`BidiFlow`提供，可以通过`join`操作符连接到流。重试`BidiFlow`将对下游的任何故障执行指定的最大重试次数。失败是由传入的失败决定函数或失败(如果没有提供失败决定函数)来决定的。如果所有重试尝试都失败，则发出该请求的最后一次失败。重试阶段需要为每个元素携带一个`Context`。这是_唯一标识_每个元素所必需的。请查看[用于惟一Id映射的上下文](#context-to-unique-id-mapping) 章节了解更多细节。

##### Scala

```scala
val retry = Retry[String, String, Long](max = 10)
val flow = Flow[(String, Long)].map { case (s, ctx) => (findAnEnglishWordThatStartWith(s), ctx) }

Source("a" :: "b" :: "c" :: Nil)
  .zipWithIndex
  .via(retry.join(flow))
  .runWith(Sink.foreach(println))
```

##### Java

```java
final BidiFlow<Pair<String, Long>,
                Pair<String, Long>,
                Pair<Try<String>, Long>,
                Pair<Try<String>, Long>, NotUsed> retry = Retry.create(2);

final Flow<Pair<String, Long>, Pair<Try<String>, Long>, NotUsed> flow =
        Flow.<Pair<String, Long>>create()
                .map(p -> Pair.create(findAnEnglishWordThatStartsWith.apply(p.first()), p.second()));

Source.from(Arrays.asList("a", "b", "c"))
        .zipWithIndex()
        .via(retry.join(flow))
        .runWith(Sink.foreach(System.out::println), mat);
```

#### 用于唯一ID映射的上下文

`Context`本身可以用作唯一的id。但是，在许多场景中，`Context`包含的内容比惟一id本身更多，或者惟一id可能作为映射从`Context`检索。squbs允许不同的选项提供一个唯一的id：

   * `Context` 它本身是一种可以用作唯一id的类型，例如，`Int`，`Long`，`java.util.UUID`
   * `Context` 扩展`UniqueId.Provider`并实现`def uniqueId`
   * `Context` 是用`UniqueId.Envelope`包装的
   * `Context` 通过调用函数映射到唯一ID

使用前三个选项，可以直接通过上下文检索唯一的id。对于最后一个选项，`Retry`允许通过`RetrySettings`传入一个函数。

###### Scala

```scala
case class MyContext(s: String, id: Long)

val settings =
  RetrySettings[String, String, MyContext](maxRetries = 3)
    .withUniqueIdMapper(context => context.id)
    
val retry = Retry(settings)    
```

###### Java

```java
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

final RetrySettings settings =
        RetrySettings.<String, String, MyContext>create(3)
                .withUniqueIdMapper(ctx -> ctx.id());
                
final BidiFlow<Pair<String, MyContext>,
               Pair<String, MyContext>,
               Pair<Try<String>, MyContext>,
               Pair<Try<String>, MyContext>, NotUsed> retry = Retry.create(settings);
```

##### 重要提示

如前所述，`RetryStage`使用`Context`来标识应该重试的元素。如果下游修改了`Context`，使其不能再用于惟一地标识元素，则跟踪的元素将永远不会被删除，本质上成为内存泄漏。此外，这些元素将永远不会被重试，并且流很可能无法实现(materialize)一个值。因此，理解上述概念并确保下游不会对其产生不利影响是**非常**重要的。

#### 失败决定者

默认情况下，对于重试而言，来自连接的`Flow`的任何`Failure`都被视为失败。然而，`Retry`阶段也通过`RetrySettings`接受一个可选的`failureDecider`参数，以便更好地控制连接的`Flow`中哪些元素实际上应该被视为应该重试的故障。

##### Scala

一个类型`Try[Out] => Boolean`的函数可以通过`withFailureDecider`调用来提供给`RetrySettings`。下面是一个例子，与任何`Failure`的消息一样，带失败状态码的http响应也被认为是一个失败：

```scala
val failureDecider = (tryResponse: Try[HttpResponse]) => tryResponse.isFailure || tryResponse.get.status.isFailure

val settings =
  RetrySettings[HttpRequest, HttpResponse, MyContext](max = 3)
    .withFailureDecider(failureDecider)

val retry = Retry(settings)

```

##### Java

一个`Function<Try<Out>, Boolean>`函数可以通过`withFailureDecider`调用来提供给`RetrySettings`。下面是一个例子，与任何`Failure`的消息一样，带状态码`400`的`Success`的`HttpResponse`也被认为是一个失败：

```java
final Function<Try<HttpResponse>, Boolean> failureDecider =
        tryResponse -> tryResponse.isFailure() || tryResponse.get().status().isFailure();

final RetrySettings settings =
        RetrySettings.<HttpRequest, HttpResponse, MyContext>create(1)
                .withFailureDecider(failureDecider);

final BidiFlow<Pair<HttpRequest, MyContext>,
               Pair<HttpRequest, MyContext>,
               Pair<Try<HttpResponse>, MyContext>,
               Pair<Try<HttpResponse>, MyContext>, NotUsed> retry = Retry.create(settings);

```

#### 带有延迟时间的重试

默认情况下，从连接的`Flow`中拉出的任何失败都会立即尝试重试。但是，`Retry`阶段也接受一个可选的延迟`Duration`参数，在每个后续重试尝试之间添加一个定时延迟。此持续时间是从将故障从连接流中提取到重新尝试(推入)连接流时的最小延迟时间。例如，创建一个在重试时延迟200毫秒的`Retry`阶段：

##### Scala

```scala
val settings = RetrySettings[String, String, Context](max = 3).withDelay(1 second)

val retry = Retry(settings)
```

##### Java

```java
final RetrySettings settings =
        RetrySettings.<String, String, Context>create(3)
                .withDelay(Duration.create("200 millis"));
        
final BidiFlow<Pair<String, Context>,
               Pair<String, Context>,
               Pair<Try<String>, Context>,
               Pair<Try<String>, Context>, NotUsed> retry = Retry.create(settings);

```

##### 指数回退

还可以指定一个可选的指数回退因子来增加每次后续重试尝试的延迟时间(最大延迟时间)。在下面的示例中，任何元素的第一次失败将在延迟200ms之后重试，然后在800ms之后重试任何第二次尝试。通常，使用公式`delay * N ^ exponentialBackOff`(其中N是重试次数)，重试延迟持续时间将继续增加。

###### Scala

```scala
val settings =
  RetrySettings[String, String, Context](max = 3)
    .withDelay(200 millis)
    .withExponentialBackoff(2)

val retry = Retry(settings)
```

###### Java

```java
final RetrySettings settings =
        RetrySettings.create<String, String, Context>.create(3)
                .withDelay(Duration.create("200 millis"))
                .withExponentialBackoff(2.0);
    

final BidiFlow<Pair<String, Context>,
               Pair<String, Context>,
               Pair<Try<String>, Context>,
               Pair<Try<String>, Context>, NotUsed> retry = Retry.create(settings);

```

##### 最大延迟

还可以指定可选的最大延迟持续时间，以提供指数回退延迟持续时间的上限。如果没有指定最大延迟，则指数回退将继续增加重试延迟持续时间，直到达到最大重试次数。

###### Scala

```scala
val settings =
  RetrySettings[String, String, Context](max = 3)
    .withDelay(200 millis)
    .withExponentialBackoff(2)
    .withMaxDelay(400 millis)

val retry = Retry(settings)
```

###### Java

```java
final RetrySettings settings =
        RetrySettings.create<String, String, Context>.create(3)
                .withDelay(Duration.create("200 millis"))
                .withExponentialBackoff(2.0)
                withMaxDelay(Duration.create("400 millis"));

final BidiFlow<Pair<String, Context>,
               Pair<String, Context>,
               Pair<Try<String>, Context>,
               Pair<Try<String>, Context>, NotUsed> retry = Retry.create(settings);
```

##### 配置背压阈值

如果连接的流不断返回失败，`Retry`将在等待重试的元素达到某个阈值时开始背压。默认情况下，阈值等于`Retry` Akka Stream `GraphStage`的内部缓冲区大小(请查看[Akka Stream属性](https://doc.akka.io/docs/akka/current/stream/stream-composition.html#attributes))。通过调用`withMaxWaitingRetries`，阈值可以独立于内部缓冲区大小：

##### Scala

```scala
val settings = RetrySettings[String, String, Context](max = 3).withMaxWaitingRetries(50)

val retry = Retry(settings)
```

##### Java

```java
final RetrySettings settings =
        RetrySettings.<String, String, Context>create(3)
                .withMaxWaitingRetries(50)
        
final BidiFlow<Pair<String, Context>,
               Pair<String, Context>,
               Pair<Try<String>, Context>,
               Pair<Try<String>, Context>, NotUsed> retry = Retry.create(settings);
```

#### 指标

`Retry`支持Codahale计数器用于状态和传递给它的元素的失败/成功率。可以通过使用名称和actor system调用`withMetrics`来启用指标。

以下计数器是已提供的：

  * Count of all retry elements ("<name>.retry-count")
  * Count of all failed elements ("<name>.failed-count")
  * Count of all successful elements ("<name>.success-count")
  * Current size of the retry registry
  * Current size of the retry queue

##### Scala

```scala
val settings = RetrySettings[String, String, Context](max = 3)
  .withMetrics("myRetry") // Takes implicit ActorSystem

val retry = Retry(settings)
```

##### Java

```java
final RetrySettings settings =
        RetrySettings.<String, String, Context>create(3)
                .withMetrics("myRetry", system)

final BidiFlow<Pair<String, Context>,
               Pair<String, Context>,
               Pair<Try<String>, Context>,
               Pair<Try<String>, Context>, NotUsed> retry = Retry.create(settings);
```
