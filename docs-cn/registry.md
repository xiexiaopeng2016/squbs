
# Actor注册

## 概述

Actor注册为squbs应用提供了一个简单的方法发送/接收消息给well-known actor，特别是跨cube。这在cube之间提供了一个额外的抽象层，允许actor在不了解其它actor的情况下找到它们。actor注册也可以作为一个外观模式，可以管理访问控制，安全性，跨模块的超时，甚至模拟非actor系统作为外部actor 。

### 概念

* Well-known actor是在[Unicomplex & Cube引导](bootstrap.md)章节描述的，通过`squbs-meta.conf`在cube中注册的actor。
* Scala的`ActorLookup`API和Java的`japi.ActorLookup`API用于发送/接收消息给Well-known actor。
* `ActorRegistry`是一个共用的外观模式actor，保有所有Well-known actor信息。

## 依赖

要使用actor注册，请将以下依赖项添加到您的`build.sbt`或scala构建文件：

```
"org.squbs" %% "squbs-actorregistry" % squbsVersion
```

## Well-Known Actor

Squbs的well-known actor定义在`META-INF/squbs-meta.conf`中的`squbs-actors`段，像[Unicomplex & Cube引导](bootstrap.md)描述的那样。actor注册扩展了这种注册，并提供了进一步的元数据，描述此actor可能消费和返回的消息类型：

* class-name:	actor的类名
* name: actor的注册名
* message-class: 请求/响应的消息类型

扩展类型信息的注册示例:

```
cube-name = org.squbs.TestCube
cube-version = "0.0.5"
squbs-actors = [
    {
      class-name = org.squbs.testcube.TestActor
      name = TestActor
      message-class = [
        {
          request = org.squbs.testcube.TestRequest
          response= org.squbs.testcube.TestResponse
        }
        {
          request = org.squbs.testcube.TestRequest1
          response= org.squbs.testcube.TestResponse1
        }
      ]
    }
]
```

## Scala API & 示例

* 发送消息(!/?/tell/ask)给请求消息类型注册为`TestRequest`的actor

  ```scala
  implicit val refFactory : ActorRefFactory = ...
  ActorLookup ! TestRequest(...)
  ```

* 发送消息(!/?/tell/ask)给请求消息类型注册为`TestRequest`，且响应消息类型为`TestResponse`的actor

  ```scala
  implicit val refFactory : ActorRefFactory = ...
  ActorLookup[TestResponse] ! TestRequest(...)
  ```

* 发送消息(!/?/tell/ask)给注册名为`TestActor`，且请求消息类型为`TestRequest`的actor

  ```scala
  implicit val refFactory : ActorRefFactory = ...
  ActorLookup("TestActor") ! TestRequest(...)
  ```

* 发送消息(!/?/tell/ask)给注册名为`TestActor`、请求消息类型为`TestRequest`，且响应消息类的类型为`TestResponse`的actor

  ```scala
  implicit val refFactory : ActorRefFactory = ...
  ActorLookup[TestResponse]("TestActor") ! TestRequest(...)  
  ```

* 解决到actorRef，其响应消息类型注册为`TestResponse`

  ```scala
  implicit val refFactory : ActorRefFactory = ...
  implicit val timeout : Timeout = ...
  ActorLookup[TestResponse].resolveOne
  ```

* 解决到actorRef，其注册名为`TestActor`

  ```scala
  implicit val refFactory : ActorRefFactory = ...
  implicit val timeout : Timeout = ...
  ActorLookup("TestActor").resolveOne
  ```
  
* 解决到actorRef，其注册名为`TestActor`，响应消息类型注册为`TestResponse`

  ```scala
  implicit val refFactory : ActorRefFactory = ...
  implicit val timeout : Timeout = ...
  ActorLookup[TestReponse]("TestActor").resolveOne
  ```
  
* 解决到actorRef，其注册名为`TestActor`，请求消息类型注册为`TestRequest`
 
  ```scala
  implicit val refFactory : ActorRefFactory = ...
  implicit val timeout : Timeout = ...
  val al= new ActorLookup(requestClass=Some(classOf[TestRequest]))
  al.resolveOne
  ```

## Java API & Samples

* Create your initial ActorLookup object.

  ```java
  // Pass in context() from and Actor or the ActorSystem
  
  private ActorLookup<?> lookup = ActorLookup.create(context());
  ```

* Send message (tell/ask) to an actor which registered its request message class type as "TestRequest"

  ```java
  lookup.tell(new TestRequest(...), self());
  ```

* Send message (tell/ask) to an actor which registered its request message class type as "TestRequest", and response message class type as "TestResponse"

  ```java
  lookup.lookup(TestResponse.class).tell(new TestRequest(...), self())
  ```

* Send message (tell/ask) to actor which registered its name as "TestActor", and request message class type as "TestRequest"

  ```java
  lookup.lookup("TestActor").tell(new TestRequest(...), self())
  ```

* Send message (tell/ask) to actor which registered name as "TestActor", request message class type as "TestRequest", and response message class type as "TestResponse"

  ```java
  lookup.lookup("TestActor", TestResponse.class).tell(new TestRequest(...), self())
  ```

* Resolve to actorRef which registered its response message class type as "TestResponse"

  ```java
  lookup.lookup(TestResponse.class).resolveOne(timeout)
  ```

* Resolve to actorRef which registered its name as "TestActor"  

  ```java
  lookup.lookup("TestActor").resolveOne(timeout)
  ```
  
* Resolve to actorRef which registered its name as "TestActor", and response message class type as "TestReponse" 

  ```java
  lookup.lookup("TestActor", TestResponse.class).resolveOne(timeout)
  ```
  
* Resolve to actorRef which registered its name as "TestActor", and request message class type as "TestRequest". This uses the full lookup signature with optional fields using the type `Optional<T>`. So the name and the request class needs to be wrapped into an `Optional` or pass Optional.empty() if it is not part of the query. The response type is always needed. If any response is accepted, set the response type to `Object.class` to identify that any subclass of `java.lang.Object` is good.
 
  ```java
  lookup.lookup(Optional.of("TestActor"), Optional.of(TestRequest.class), Object.class)
  ```

## 响应类型

通过`ActorLookup`响应类型发现(当提供响应类型时)发现的actor的响应类型在查找的结果中保持。虽然程式化的响应类型对于`tell`或`!`不太重要，但是在`ask`或`?`上变得很重要。`ask` 的返回类型通常是`Future[Any]`在Scala中或`CompletionStage<Object>`在Java中。

然而，返回类型从一个`ask`或`?`当查找ActorLookup时，它携带响应类型。
然而`ask`或`?`在ActorLookup上，回类型是查找到的返回类型。如果使用响应类型 T 进行查找, 您将得到Future[T]如下所示。

However the return type from an `ask` or `?` on ActorLookup carries the response type when looked up with it. So you'll get a `Future[T]` or `CompletionStage<T>` if you lookup with response type `T` as can be demonstrated in the examples below. No further MapTo is needed:

### Scala

```scala
// In this example, we show full type annotation. The Scala compiler is able
// to infer the type if you just pass one side, i.e. the type parameter at
// ActorLookup, or the type annotation on the val f declaration.

val f: Future[TestResponse] = ActorLookup[TestResponse] ? TestRequest(...)
```

### Java

```java
CompletionStage<TestResponse> f = lookup.lookup(TestResponse.class).ask(new TestRequest(...), timeout)
```

## 错误处理

不像`actorSelection`，它将向死信发送消息。`ActorLookup`将回答`ActorNotFound`，如果想要的actor系统中不存在或未注册。在Java API中，`ActorNotFound`通常封装`java.util.concurrent.ExecutionException`，并可在`ExecutionException`中由`getCause()`访问。

## 监控

这是一个为每个well-known actor创建的JMX Bean。名为`org.squbs.unicomplex:type=ActorRegistry,name=${actorPath}`
