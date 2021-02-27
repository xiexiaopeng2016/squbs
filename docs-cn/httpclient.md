# Akka HTTP 客户端 on Steroids

### 概述

`squbs-httpclient`项目在保持Akka HTTP API的同时，增加了[Akka HTTP主机级别客户端API](http://doc.akka.io/docs/akka-http/current/scala/http/client-side/host-level.html)操作方面的内容。下面是它添加的功能列表：

* [服务发现](#service-discovery-chain): 允许插入任何服务发现机制，并允许通过字符串标识符(例如`paymentserv`)解析HTTP端点。
* [逐个客户端配置](#per-client-configuration): 让每个客户端单独覆盖`application.conf`中的默认值。
* [管道](#pipeline): 允许一个`Bidi` Akka Stream流被注册为全局的或客户端单独的。 
* [Metrics](#metrics): 为每个客户端提供开箱即用的[Codahale Metrics](http://metrics.dropwizard.io/3.1.0/getting-started/)，**无需** AspectJ。
* [JMX Beans](#jmx-beans): 将每一个客户端的配置暴露为JMX Bean。
* [断路器](#circuit-breaker): 基于断路器提供流的弹性。

### 依赖

增加如下依赖到你的`build.sbt`或者其它scala构建文件：

```
"org.squbs" %% "squbs-httpclient" % squbsVersion
```

### 用法

`squbs-httpclient`项目保持Akka HTTP API。唯一的例外是创建主机连接池期间。不是使用`Http().cachedHostConnectionPool`，它定义了`ClientFlow`使用一组相同的参数(和一些其它可选的参数)。

##### Scala

类似于[Akka HTTP主机级别客户端API](http://doc.akka.io/docs/akka-http/current/scala/http/client-side/host-level.html#example)例子，Scala使用`ClientFlow`如下：

```scala
implicit val materializer = ActorMaterializer()
// construct a pool client flow with context type `Int`
val poolClientFlow = ClientFlow[Int]("sample") // Only this line is specific to squbs.  Takes implicit ActorSystem.

val responseFuture: Future[(Try[HttpResponse], Int)] =
  Source.single(HttpRequest(uri = "/") -> 42)
    .via(poolClientFlow)
    .runWith(Sink.head)
```

可以传递可选的设置，例如，`HttpsConnectionContext`和`ConnectionPoolSettings`给`ClientFlow`。

```scala
val clientFlow = ClientFlow("sample", Some(connectionContext), Some(connectionPoolSettings))
```

##### Java

同样，类似于[Akka HTTP主机级别客户端API](http://doc.akka.io/docs/akka-http/current/java/http/client-side/host-level.html#example)例子，Java使用`ClientFlow`如下：

```java
final ActorSystem system = ActorSystem.create();
final ActorMaterializer mat = ActorMaterializer.create(system);

final Flow<Pair<HttpRequest, Integer>, Pair<Try<HttpResponse>, Integer>, HostConnectionPool>
    clientFlow = ClientFlow.create("sample", system, mat); // Only this line is specific to squbs
    
final CompletionStage<Pair<Try<HttpResponse>, Integer>> responseFuture =
    Source.single(Pair.create(HttpRequest.create().withUri("/"), 42))
        .via(clientFlow)
        .runWith(Sink.head(), mat);
```

可以传递`Optional`设置，例如，`HttpsConnectionContext`和`ConnectionPoolSettings`，给`ClientFlow.create`。它也提供了一个流畅的API：

```java
final Flow<Pair<HttpRequest, Integer>, Pair<Try<HttpResponse>, Integer>, HostConnectionPool> clientFlow =
        ClientFlow.<Integer>builder()
                .withSettings(connectionPoolSettings)
                .withConnectionContext(connectionContext)
                .create("sample", system, mat);
```

#### HTTP模型

##### Scala

下面是Scala中`HttpRequest`创建的例子。请阅读[HTTP模型Scala文档](http://doc.akka.io/docs/akka-http/current/scala/http/common/http-model.html)了解更详细的内容：

```scala
import HttpProtocols._
import MediaTypes._
  
val charset = HttpCharsets.`UTF-8`
val userData = ByteString("abc", charset.nioCharset())
val authorization = headers.Authorization(BasicHttpCredentials("user", "pass"))

HttpRequest(
  PUT,
  uri = "/user",
  entity = HttpEntity(`text/plain` withCharset charset, userData),
  headers = List(authorization),
  protocol = `HTTP/1.0`)
```
##### Java

下面是Java中`HttpRequest`创建的例子。请阅读[HTTP模型Java文档](http://doc.akka.io/docs/akka-http/current/java/http/http-model.html)了解更详细的内容：

```java
import HttpProtocols.*;
import MediaTypes.*;

Authorization authorization = Authorization.basic("user", "pass");
HttpRequest complexRequest =
    HttpRequest.PUT("/user")
        .withEntity(HttpEntities.create(ContentTypes.TEXT_PLAIN_UTF8, "abc"))
        .addHeader(authorization)
        .withProtocol(HttpProtocols.HTTP_1_0);
```

### 服务发现链

`squbs-httpclient`在客户端池创建的时候不需要提供主机名/端口组合。相反，它允许注册一个服务发现链，通过运行已注册的服务发现机制，允许使用一个字符串标识符解析`HttpEndpoint`。例如，在上面的例子中，`"sample"`是客户端想要连接的服务逻辑名，配置的服务发现链将它解析到一个包括主机和端口的`HttpEndpoint`，例如，`http://akka.io:80`。

请注意，你仍然可以将一个有效的http URI作为字符串传递给`ClientFlow`，作为一个默认的解析器来解析有效的http URI，默认情况下是在服务发现链中预先注册的：

   * `ClientFlow[Int]("http://akka.io")` in Scala
   * `ClientFlow.create("http://akka.io", system, mat)` in Java

注册解析器有两种变体，如下所示。闭包风格允许更紧凑、可读的代码。然而，子类具有保持状态并根据此状态做出决策的能力：

##### Scala

注册函数类型`(String, Env) => Option[HttpEndpoint]`:

```scala
ResolverRegistry(system).register[HttpEndpoint]("SampleEndpointResolver") { (svcName, env) =>
  svcName match {
    case "sample" => Some(HttpEndpoint("http://akka.io:80"))
    case "google" => Some(HttpEndpoint("http://www.google.com:80"))
    case _ => None
  }
}
```

注册类继承于`Resolver[HttpEndpoint]`:

```scala
class SampleEndpointResolver extends Resolver[HttpEndpoint] {
  override def name: String = "SampleEndpointResolver"

  override def resolve(svcName: String, env: Environment): Option[HttpEndpoint] =
    svcName match {
      case "sample" => Some(Endpoint("http://akka.io:80"))
      case "google" => Some(Endpoint("http://www.google.com:80"))
      case _ => None
    }
}

// Register EndpointResolver
ResolverRegistry(system).register[HttpEndpoint](new SampleEndpointResolver)
```

##### Java

注册`BiFunction<String, Environment, Optional<HttpEndpoint>>`:

```java
ResolverRegistry.get(system).register(HttpEndpoint.class, "SampleEndpointResolver", (svcName, env) -> {
    if ("sample".equals(svcName)) {
        return Optional.of(HttpEndpoint.create("http://akka.io:80"));
    } else if ("google".equals(svcName))
        return Optional.of(HttpEndpoint.create("http://www.google.com:80"));
    } else {
        return Optional.empty();
    }
});

```

注册类继承于`AbstractResolver<HttpEndpoint>`:

```java
class SampleEndpointResolver extends AbstractResolver<HttpEndpoint> {
    String name() {
        return "SampleEndpointResolver";
    }

    Optional<HttpEndpoint> resolve(svcName: String, env: Environment) { 
        if ("sample".equals(svcName)) {
            return Optional.of(Endpoint.create("http://akka.io:80"));
        } else if ("google".equals(svcName))
            return Optional.of(Endpoint.create("http://www.google.com:80"));
        } else {
            return Optional.empty();
        }
    }
}    

// Register EndpointResolver
ResolverRegistry.get(system).register(HttpEndpoint.class, new SampleEndpointResolver());
```

你可以注册多个`Resolver`。链的执行顺序与注册顺序相反。如果解析器返回`None`，则表示无法解析它，并尝试下一个解析器。

如果已解析的端点是安全的，例如，https，则可以将`SSLContext`作为可选参数传递给`HttpEndpoint`。

也可以将一个可选的`Config`传递给`HttpEndpoint`用于覆盖默认的配置。然而，特定于客户机的配置具有比传递的配置更高的优先级。

请看[资源解析](resolver.md)中关于解析的详细信息。

### 单个客户端配置

Akka HTTP配置定义了配置的默认值。你可以覆盖`application.conf`中的默认值；然而，这将影响所有的客户端。要执行一个客户端特定的覆盖，Akka HTTP在`HostConnectionPool`流创建时，传入一个`ConnectionPoolSettings`。这也得到了squbs的支持。

除了以上所述，squbs允许在`application.conf`中覆盖特定的客户端。您只需要指定一个客户端名称的配置段落，其包含`type = squbs.httpclient`。然后，可以在段落中指定任何客户端配置。例如，如果我们想要覆盖`max-connections`设置，只针对上面的`"sample"`客户端，而不针对其他客户端，我们可以这样做:

```
sample {
  type = squbs.httpclient
  
  akka.http.host-connection-pool {
    max-connections = 10
  }
}
```

### 管道

我们通常需要在不同的客户端之间拥有共同的基础设施功能或组织标准。基础设施包括，但不限于，日志、指标收集、请求追踪、认证/授权、追踪、cookie管理、A/B测试等。随着squbs促进关注点分离，此类逻辑属于基础设施，而不是客户端实现。[squbs管道](pipeline.md)允许基础设施提供组件安装到客户端，而无需客户自己担心这些方面。请阅读[squbs管道](pipeline.md)了解更多细节。

一般来说，管道是一个`Bidi`流，充当squbs客户端和Akka HTTP层之间的桥梁。`squbs-httpclient`允许注册一个Bidi Akka Streams flow全局地对所有客户端，或者个别客户端。为了注册一个客户端特定管道，设置`pipeline`配置。可以通过设置`defaultPipeline`，打开/关闭默认管道(如果没有指定，默认为on)：

```
sample {
  type = squbs.httpclient
  pipeline = metricsFlow
  defaultPipeline = on
}
```

请阅读[squbs管道](pipeline.md)章节找出如何创建管道和配置默认管道。

### 指标

squbs自带预建的[管道](#pipeline)元素用于指标收集，且squbs giter8模板设置这些作为默认值。因此，每个squbs http客户端都可以开箱即用地收集Codahale Metrics，无需任何代码更改或配置。请注意，squbs指标收集不需要AspectJ或者其它运行时代码编写。下面指标在JMX可以默认可用：

   * Request Timer
   * Request Count Meter
   * Response Count Meter
   * A meter for each http response status code category: 2xx, 3xx, 4xx, 5xx
   * A meter for each exception type that was returned by `ClientFlow`.

可以通过`MetricsExtension(system).metrics`访问`MetricRegistry`。这使您可以创建更多的计量，计时器，直方图等，或传递给不同类型的指标报告者。

### JMX Bean

在解决问题时，系统配置的可见性是最重要的。`squbs-httpclient`为每个客户端注册一个JMX bean。JMX bean暴露所有的配置，例如端点、主机连接池设置，等等。bean的名称被设置为`org.squbs.configuration.${system.name}:type=squbs.httpclient,name=$name`。所以，如果actor system名称是`squbs`，客户端名称是`sample`，那么JMX bean名称将会是`org.squbs.configuration.squbs:type=squbs.httpclient,name=sample`。

### 断路器

squbs提供`CircuitBreakerBidi` Akka Streams `GraphStage`来为流提供断路器功能。这是为流实现的通用断路器。请查看[断路器](circuitbreaker.md)文档中的详细内容。

断路器有可能改变消息的顺序，所以它需要一个可随身携带的`Context`，类似`ClientFlow`。但是，除此之外，它还需要能够唯一地识别每个元素的内部机制。因此，传递给`ClientFlow`的`Context`或来自`Context`的映射需要能够唯一地标识每个元素。如果断路器已启用，并且传递给`ClientFlow`的`Context`没有唯一标识每个元素，那么你会经历意想不到的行为。请查看断路器文档中的[Context to Unique Id Mapping](circuitbreaker.md#context-to-unique-id-mapping)章节关于提供唯一id的详细内容。

默认情况下，断路器是禁用的。请查看[application.conf中的配置](#configuring-in-applicationconf)和[编程方式传递CircuitBreakerSettings](#passing-circuitbreakersettings-programmatically)章节来启用。

一旦启用，默认情况下，任何`Failure`或者一个带有400状态码的`Success`，递增失败计数。`CircuitBreakerState`的默认实现是`AtomicCircuitBreakerState`，其可以跨物化和跨流共享。这些可以通过[编程方式传递CircuitBreakerSettings](#passing-circuitbreakersettings-programmatically)进行定制。

##### application.conf中的配置

在客户端的特定配置中，您可以添加`circuit-breaker`并指定想要覆盖的配置。 其余的将使用默认设置。 请查看[这里](../squbs-ext/src/main/resources/reference.conf)的默认断路器配置。

```
sample {
  type = squbs.httpclient

  circuit-breaker {
    max-failures = 2
    call-timeout = 10 milliseconds
    reset-timeout = 100 seconds
  }
}
```

##### 编程方式传递CircuitBreakerSettings

你可以编程方式传递一个`CircuitBreakerSettings`实例。这个API让你传递一个自定义`CircuitBreakerState`和可选的fallback，失败判断者和唯一id映射函数。如果以编程方式传入一个`CircuitBreakerSettings`实例，那么在`application.conf`中的断路器设置将被忽略。

在下面的示例中，通过`withFallback`提供了fallback响应。默认的失败判断者是通过`withFailureDecider`重写，只考虑状态代码`>= 500`来增加断路器的故障计数：

###### Scala

```scala
import org.squbs.streams.circuitbreaker.CircuitBreakerSettings

val circuitBreakerSettings =
  CircuitBreakerSettings[HttpRequest, HttpResponse, Int](circuitBreakerState)
    .withFallback( _ => Try(HttpResponse(entity = "Fallback Response")))
    .withFailureDecider(
      response => response.isFailure || response.get.status.intValue() >= StatusCodes.InternalServerError.intValue)
    
val clientFlow = ClientFlow[Int]("sample", circuitBreakerSettings = Some(circuitBreakerSettings))    
```

###### Java

```java
import org.squbs.streams.circuitbreaker.japi.CircuitBreakerSettings;

final CircuitBreakerSettings circuitBreakerSettings =
        CircuitBreakerSettings.<HttpRequest, HttpResponse, Integer>create(circuitBreakerState)
                .withFallback(httpRequest -> Success.apply(HttpResponse.create().withEntity("Fallback Response")))
                .withFailureDecider(response ->
                        response.isFailure() || response.get().status().intValue() >= StatusCodes.INTERNAL_SERVER_ERROR.intValue());
```

##### 调用其它服务作为fallback

断路器使用中的一个常见场景是调用另一个服务作为fallback。调用另一个服务需要在fallback函数中有一个新的流物化；所以，我们建议用户将失败的快速消息传递到下游，并相应地进行分支。这可以在相同的流中定义fallback `ClientFlow`。
