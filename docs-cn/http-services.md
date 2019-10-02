# 实现HTTP(S)服务

[TOC]

## 概述

HTTP是最普遍的集成协议。它是web服务的基础，包括客户端和服务器端。Akka HTTP提供了强大的服务器和客户端API。squbs有意图保持这些api不改变。相反，squbs提供基础结构来允许生产就绪(production-ready)的使用这些API，通过为这些组件提供标准配置。比如HTTP监听器 - 其服务可以使用它来接收和处理请求；管道 - 允许日志、监视、身份验证/授权，在请求到达应用之前，以及在响应离开应用之后，进入wire之前。

squbs支持Akka HTTP，包括用于定义服务的低级别和高级别服务端API。两个API都达到了全生产化的支持，例如监听器、管道、日志和监控。另外，squbs支持Scala和Java风格的服务定义。这些服务处理器在类中声明，并通过`META-INF/squbs-meta.conf`文件(通过此文件中的`squbs-services`项)注册到squbs。每个服务样式都以相同的方式注册, 只需提供类名和配置。

所有的squbs服务定义都可以访问`context`字段，它是一种`akka.actor.ActorContext`用于访问actor system，scheduler和各种Akka设施。

## 依赖

启动服务器和注册服务定义都需要下面的依赖：

```
"org.squbs" %% "squbs-unicomplex" % squbsVersion
```


## 定义服务

服务可以在Scala或Java中定义，使用高级别或低级别API。服务定义类**必须具有无参构造函数**, 并且必须注册，以便处理传入的Http请求。

### 高级别 Scala API

高级别服务端API由Akka HTTP的`Route`和它的指令表示。为了使用一个`Route`来处理请求，只需要提供一个继承`org.squbs.unicomplex.RouteDefinition`特质，并提供`route`函数的类，如下所示：

```scala
import akka.http.scaladsl.server.Route
import org.squbs.unicomplex.RouteDefinition

class PingPongSvc extends RouteDefinition {

  override def route: Route = path("ping") {
    get {
      complete("pong")
    }
  }
  
  // Overriding the rejectionHandler is optional
  override def rejectionHandler: Option[RejectionHandler] =
    Some(RejectionHandler.newBuilder().handle {
      case ServiceRejection => complete("rejected")
    }.result())

  // Overriding the exceptionHandler is optional
  override def exceptionHandler: Option[ExceptionHandler] =
    Some(ExceptionHandler {
      case _: ServiceException => complete("exception")
    })
}
```

除了定义`route`外, 你也可以提供一个[`RejectionHandler`](http://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/rejections.html#the-rejectionhandler)和一个[`ExceptionHandler`](http://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/exception-handling.html#exception-handling)，通过覆盖相应的
`rejectionHandler`和`exceptionHandler`方法。这些可以在上面的例子中看到。

请参考[Akka HTTP high-level API](http://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/index.html), [Routing DSL](http://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/overview.html), [Directives](http://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/directives/index.html), [Rejection](http://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/rejections.html), 和[Exception Handling](http://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/exception-handling.html) 文档以便充分利用这些API。

### 低级别 Scala API

使用Scala低级别API，只需要继承`org.squbs.unicomplex.FlowDefinition`并覆盖`flow`函数。`flow`需要是`Flow[HttpRequest, HttpResponse, NotUsed]`类型，使用了Akka HTTP提供的Scala DSL和模型，如下所示：

```scala
import akka.http.scaladsl.model.Uri.Path
import akka.http.scaladsl.model._
import akka.stream.scaladsl.Flow
import org.squbs.unicomplex.FlowDefinition

class SampleFlowSvc extends FlowDefinition {

  override def flow = Flow[HttpRequest].map {
    case HttpRequest(_, Uri(_, _, Path("ping"), _, _), _, _, _) =>
      HttpResponse(StatusCodes.OK, entity = "pong")
    case _ =>
      HttpResponse(StatusCodes.NotFound, entity = "Path not found!")
}
```

这提供了对Akka HTTP底层服务器端API的`Flow`表示的访问。请参考[Akka HTTP low-level API](http://doc.akka.io/docs/akka-http/current/scala/http/low-level-server-side-api.html#streams-and-http), [Akka Streams](http://doc.akka.io/docs/akka/current/scala/stream/stream-quickstart.html), 和[HTTP Model](http://doc.akka.io/docs/akka-http/current/scala/http/common/http-model.html#http-model-scala)文档了解有关构造更复杂的`Flow`的进一步信息。

### 高级别 Java API

The high-level server-side API is represented by Akka HTTP's `Route` artifact and its directives. To use a `Route` to handle requests, just provide a class extending the `org.squbs.unicomplex.RouteDefinition` trait and provide the `route` method as follows:

```java
import akka.http.javadsl.server.ExceptionHandler;
import akka.http.javadsl.server.RejectionHandler;
import akka.http.javadsl.server.Route;
import org.squbs.unicomplex.AbstractRouteDefinition;

import java.util.Optional;

public class JavaRouteSvc extends AbstractRouteDefinition {

    @Override
    public Route route() {
        return route(
                path("ping", () ->
                        complete("pong")
                ),
                path("hello", () ->
                        complete("hi")
                ));
    }

	// Overriding the rejection handler is optional
    @Override
    public Optional<RejectionHandler> rejectionHandler() {
        return Optional.of(RejectionHandler.newBuilder()
                .handle(ServiceRejection.class, sr ->
                        complete("rejected"))
                .build());
    }

    // Overriding the exception handler is optional
    @Override
    public Optional<ExceptionHandler> exceptionHandler() {
        return Optional.of(ExceptionHandler.newBuilder()
                .match(ServiceException.class, se ->
                        complete("exception"))
                .build());
    }
}
```

In addition to defining the `route`, you can also provide a [`RejectionHandler`](http://doc.akka.io/docs/akka-http/current/java/http/routing-dsl/rejections.html#the-rejectionhandler) and an [`ExceptionHandler`](http://doc.akka.io/docs/akka-http/current/java/http/routing-dsl/exception-handling.html#exception-handling) by overriding the `rejectionHandler` and `exceptionHandler` methods accordingly. These can be seen in the example above.

Please refer to the [Akka HTTP high-level API](http://doc.akka.io/docs/akka-http/current/java/http/routing-dsl/index.html), [Routing DSL](http://doc.akka.io/docs/akka-http/current/java/http/routing-dsl/overview.html), [Directives](http://doc.akka.io/docs/akka-http/current/java/http/routing-dsl/directives/index.html), [Rejection](http://doc.akka.io/docs/akka-http/current/java/http/routing-dsl/rejections.html), and [Exception Handling](http://doc.akka.io/docs/akka-http/current/java/http/routing-dsl/exception-handling.html) documentation to fully utilize these APIs.

### 低级别 Java API

To use the Java low-level API, just extend `org.squbs.unicomplex.AbstractFlowDefinition` and override the `flow` method. The `flow` needs to be of type `Flow[HttpRequest, HttpResponse, NotUsed]` using the Java DSL and model provided by Akka HTTP. Note the imports in the following:

```java
import akka.NotUsed;
import akka.http.javadsl.model.*;
import akka.stream.javadsl.Flow;
import org.squbs.unicomplex.AbstractFlowDefinition;

public class JavaFlowSvc extends AbstractFlowDefinition {

    @Override
    public Flow<HttpRequest, HttpResponse, NotUsed> flow() {
        return Flow.of(HttpRequest.class)
                .map(req -> {
                    String path = req.getUri().path();
                    if (path.equals(webContext() + "/ping")) {
                        return HttpResponse.create().withStatus(StatusCodes.OK).withEntity("pong");
                    } else {
                        return HttpResponse.create().withStatus(StatusCodes.NOT_FOUND).withEntity("Path not found!");
                    }
                });
    }
}
```

**Note:** The `webContext()` method as well as the `context()` method for accessing the actor context are provided by the `AbstractFlowDefinition` class.

This provides access to the `Flow` representation of the Akka HTTP low-level server-side API. Please refer to the [Akka HTTP low-level API](http://doc.akka.io/docs/akka-http/current/java/http/server-side/low-level-server-side-api.html#streams-and-http), [Akka Streams](http://doc.akka.io/docs/akka/current/java/stream/stream-quickstart.html), and the [HTTP model](http://doc.akka.io/docs/akka-http/current/java/http/http-model.html#http-model-java) documentation for further information on constructing more sophisticated `Flow`s.

## 服务注册

服务的元数据在`META-INF/squbs-meta.conf`中声明，如下所示。

```
cube-name = org.sample.sampleflowsvc
cube-version = "0.0.2"
squbs-services = [
  {
    class-name = org.sample.SampleFlowSvc
    web-context = sample # You can also specify bottles/v1, for instance.
    
    # The listeners entry is optional, and defaults to 'default-listener'.
    listeners = [ default-listener, my-listener ]
    
    # Optional, defaults to a default pipeline.
    pipeline = some-pipeline
    
    # Optional, disables the default pipeline if set to false.  If missing, it is set to on.
    defaultPipeline = on
    
    # Optional, only applies to actors.
    init-required = false
  }
]
```

class-name参数标识了服务定义类，其可以使用低级别或高级别API，且可以在Java或者Scala中实现。

web-context是一个字符串，其唯一标识了要派发给此服务的请求的web上下文。请参考[Web上下文](#the-web-context)部分的详细内容。

listeners参数是可选的，声明一个监听程序列表来绑定此服务。监听器绑定在下面的[监听器绑定](#listener-binding)部分中讨论。

pipeline是一组请求`pre-`和`post-`处理器，在请求被处理之前和之后。管道的名称可以由一个`pipeline`参数指定。随着`pipeline`的指定，一个针对请求/响应的`defaultPipeline`设置将一起插入到配置中。要禁用此服务的默认管道，你可以在`META-INF/squbs-meta.conf`中设置`defaultPipeline = off`。请参考[请求/响应管道](pipeline.md)的更多信息。

### 监听器绑定

与直接编程Akka HTTP不同, squbs通过其侦听器提供所有套接字绑定和连接管理。只需通过上面讨论的一个或多个API提供请求/响应处理, 并将它们的实现注册到squbs。这使得跨服务的绑定配置标准化, 并允许跨服务的统一配置管理。

监听器在`application.conf`或`reference.conf`中声明，通常放在项目的`src/main/resources`目录。监听器声明了接口、端口、HTTPS安全属性、和名称别名。并在[配置](configuration.md#listeners)中说明。

服务处理器将自身附加到一个或多个监听器。`listeners`属性是一个处理器应绑定的监听器或别名的列表。如果未定义监听器, 它将默认为`default-listener`。

通配符`"*"`(注意，它必须使用双引号，否则将不会被正确地解释)是一个特殊情况，它意味着将此处理器附加到所有激活的监听器上。但是，它本身不会激活任何监听器，如果一个监听器尚未被一个处理器的具体附加激活。如果某处理器需要激活默认监听器，并附加由其他处理器激活的其他监听器, 则这样的附加需要分别指定, 如下所示:

```
listeners = [ default-listener, "*" ]  // "*"表示其他处理器激活的监听器
```

## Web上下文

每个服务入口点都绑定到唯一的web上下文, 它是由`/`字符分隔的主要路径片段。例如，url `http://mysite.com/my-context/index`将会匹配上下文`"my-context"`，如果已注册。它也可以匹配根上下文，如果`"my-context"`没有注册。Web上下文不一定是路径的第一个斜杠分隔的片段。依赖于上下文注册，它可能匹配多个这样的片段。一个具体的例子是一个带有服务版本的URL。URL`http://mysite.com/my-context/v2/index`能以`my-context`或`my-context/v2`作为web上下文，依赖于注册的上下文是什么。如果`my-context`和`my-context/v2`都注册了，最长的匹配 - 在本例中`my-context/v2`将用于路由请求。这对于将不同版本的web接口或API分离到不同的cube/模块中非常有用。

已注册的web上下文**不能**以一个`/`字符开头。在多段上下文的情况下，它可以用`/`字符作为片段分隔符。并且允许`""`作为根上下文。如果多个服务匹配请求，则最长的上下文匹配优先。

当web上下文是注册在元数据中的时候，`route`，与特别是定义在低级别API中的`flow`，需要知道它所服务的web上下文。

* Java服务处理器类可以直接访问`webContext()`方法。
* Scala服务处理器类会想要混入`org.squbs.unicomplex.WebContext`特质。这样做会将以下字段添加到您的类中:

   ```scala
   val webContext: String
   ```

在构建对象时, webContext字段被初始化为在元数据中设置的已注册web上下文的值, 如下所示:

```scala
class SampleFlowSvc extends FlowDefinition with WebContext {

  def flow = Flow[HttpRequest].map {
    case HttpRequest(_, Uri(_, _, Path(s"$webContext/ping"), _, _), _, _, _) =>
      HttpResponse(StatusCodes.OK, entity = "pong")
    case _ =>
      HttpResponse(StatusCodes.NotFound, entity = "Path not found!")
  }
}
```

## 高级别路由API的规则和行为

1. **并发状态访问:** 提供的`route`可由多个连接使用，因此可用并发线程。如果`route`访问封装在`RouteDefinition`(Scala)或`AbstractRouteDefinition`(Java)类中的任何状态，需要注意的是，这种访问可以是并发的，包括读和写。这种读取或写入封装类中可变状态的访问是不安全的。在这种情况下, 强烈推荐使用Akka `Actor`。

2. **访问actor上下文:** 默认情况下，`RouteDefinition`/`AbstractRouteDefinition`可以访问`ActorContext`，通过使用`context`字段(Scala)或`context()`方法(Java)。这可以用于创建新的actor或访问其他actor。

3. **访问web上下文:** 对于Scala `RouteDefinition`，假如`WebContext`特质已经混入，它将可以访问`webContext`字段。Java `AbstractRouteDefinition`在所有用例中提供`webContext()`方法。这个字段/方法用于确定来自于root的web上下文或路径，在这个地方这个`RouteDefinition`/`AbstractRouteDefinition`处理请求。 

## 低级别流API的规则和行为

在实现一个`FlowDefinition`(Scala)或`AbstractFlowDefinition`(Java)时，您必须牢记一些规则：

1. **恰好一个(Exactly one)响应:** 应用程序的责任是为每个请求生成恰好一个响应。
2. **响应顺序:** 响应的顺序与相关联的请求的顺序相匹配，那个是有意义的，如果启用了HTTP管道，则多个传入请求的处理可能重叠的。

3. **并发状态的访问:** 流可以多次物化, 自己产生多个`Flow`实例。如果这些实例访问封装在`FlowDefinition`或`AbstractFlowDefinition`里的状态，需要注意的是这样的访问是并发的，包括读和写。这种读或写封装的类中的可变状态的访问是不安全的。这种情况，强烈建议使用Akka `Actor`。

4. **访问actor上下文:** 默认情况下，`FlowDefinition`/`AbstractFlowDefinition` 可以访问`ActorContext`，使用`context`字段(Scala)或`context()`方法(Java)。这可以用于创建新的actor或访问其他actor。

5. **访问web上下文:** 对于Scala `FlowDefinition`，假如`WebContext`特质已经混入，它将可以访问`webContext`字段。Java `AbstractFlowDefinition`在所有用例提供了`webContext()`方法。这个字段/方法用于确定来自root的web上下文或路径，在这个地方这个`FlowDefinition`/`AbstractFlowDefinition`处理请求。

6. **请求路径:** `HttpRequest`对象未被修改地交给这个流。`webContext`在请求的`Path`中。处理带有已知的`webContext`的请求，这是用户的工作(如上所见) 。换言之, 低级别API直接处理`HttpRequest`，需要手动将web上下文考虑到任何路径匹配。

## 度量(Metrics)

squbs自带预构建[pipeline](#pipeline)的元素用于度量集合，并且squbs giter8模板设置它们作为默认值。因此，每个squbs http(s)服务都可以在没有任何代码更改或配置的情况下, 收集[Codahale Metrics](http://metrics.dropwizard.io/3.1.0/getting-started/)开箱即用的。请注意, squbs度量集合不需要AspectJ或任何其他运行时代码的编写。默认情况下，在JMX上可以使用以下度量标准:

   * 请求级别度量:
      * Request Timer
      * Request Count Meter
      * A meter for each http response status code category: 2xx, 3xx, 4xx, 5xx
      * A meter for each exception type that was returned by the service.
  * Connection Level Metrics:
     * Active Connections Counter
     * Connection Creation Meter
     * Connection Termination Meter


你可以通过`MetricsExtension(system).metrics`访问`MetricRegistry`。这使您可以创建更多的计量器，计时器，直方图等，或传递给不同类型的度量报表。
