# 请求/响应管道

### 概述

我们经常需要在不同的服务/客户端之间有公共的基础设施功能或组织标准。这样的基础设施包括但不限于，日志、指标收集、请求追踪、认证/授权、追踪、cookie管理、A/B测试等。

随着squbs促进关注点分离，此类逻辑属于基础设施，而不是客户端实现。squbs管道允许基础设施提供组件安装到服务/客户端，而无需服务/客户自己操心。

一般而言，一个squbs管道是一个Bidi Flow，扮演如下之间的桥梁：

  * Akka HTTP层和squbs服务:
    * 从Akka HTTP发送到squbs服务的所有请求消息都将通过管道
    * 反之亦然，从squbs服务发送的所有响应消息将通过管道
  * squbs客户端和Akka HTTP主机连接池flow：
    * 从squbs客户端发送到Akka HTTP连接池的所有请求消息将通过管道
    * 反之亦然，从Akka HTTP连接池到squbs客户端所有响应消息将通过管道

### 管道声明

通过以下配置指定的默认pre/post流将自动连接到服务器/客户端管道，除非个别服务/客户端配置中`defaultPipeline`设置为`off`：

```
squbs.pipeline.server.default {
    pre-flow = defaultServerPreFlow
    post-flow = defaultServerPostFlow
}

squbs.pipeline.client.default {
    pre-flow = defaultClientPreFlow
    post-flow = defaultClientPostFlow
}
```

#### 为服务声明管道

在`squbs-meta.conf`中，你可以为服务指定一个管道：

```
squbs-services = [
  {
    class-name = org.squbs.sample.MyActor
    web-context = mypath
    pipeline = dummyflow
  }
]
```

如果没有用于`squbs-service`的自定义管道，则只需省略。

使用上述配置，管道将如下所示：

```
                 +---------+   +---------+   +---------+   +---------+
RequestContext ~>|         |~> |         |~> |         |~> |         | 
                 | default |   |  dummy  |   | default |   |  squbs  |
                 | PreFlow |   |  flow   |   | PostFlow|   | service | 
RequestContext <~|         |<~ |         |<~ |         |<~ |         |
                 +---------+   +---------+   +---------+   +---------+
```

`RequestContext`基本上是围绕`HttpRequest`和`HttpResponse`的包装，这也允许携带上下文信息。

#### 为客户端声明管道

在`application.conf`中，你可以为客户端指定一个管道：

```
sample {
  type = squbs.httpclient
  pipeline = dummyFlow
}
```

如果没有用于`squbs-client`的自定义管道，则只需省略。

使用上述配置，管道将如下所示：


```
                 +---------+   +---------+   +---------+   +----------+
RequestContext ~>|         |~> |         |~> |         |~> |   Host   | 
                 | default |   |  dummy  |   | default |   |Connection|
                 | PreFlow |   |  flow   |   | PostFlow|   |   Pool   | 
RequestContext <~|         |<~ |         |<~ |         |<~ |   Flow   |
                 +---------+   +---------+   +---------+   +----------+
```


### Bidi Flow配置

一个 bidi flow可以如下方式指定：

```
dummyflow {
  type = squbs.pipelineflow
  factory = org.squbs.sample.DummyBidiFlow
}
```

* type: 将配置标识为一个`squbs.pipelineflow`。
* factory: 创建`BidiFlow`的工厂类。

一个简单的`DummyBidiFlow`示例如下所示：

##### Scala

```scala
class DummyBidiFlow extends PipelineFlowFactory {

  override def create(context: Context)(implicit system: ActorSystem): PipelineFlow = {
     BidiFlow.fromGraph(GraphDSL.create() { implicit b =>
      val inbound = b.add(Flow[RequestContext].map { rc => rc.withRequestHeader(RawHeader("DummyRequest", "ReqValue")) })
      val outbound = b.add(Flow[RequestContext].map{ rc => rc.withResponseHeader(RawHeader("DummyResponse", "ResValue"))})
      BidiShape.fromFlows(inbound, outbound)
    })
  }
}
```

##### Java

```java
public class DummyBidiFlow extends AbstractPipelineFlowFactory {

    @Override
    public BidiFlow<RequestContext, RequestContext, RequestContext, RequestContext, NotUsed> create(Context context, ActorSystem system) {
        return BidiFlow.fromGraph(GraphDSL.create(b -> {
            final FlowShape<RequestContext, RequestContext> inbound = b.add(
                    Flow.of(RequestContext.class)
                            .map(rc -> rc.withRequestHeader(RawHeader.create("DummyRequest", "ReqValue"))));
            final FlowShape<RequestContext, RequestContext> outbound = b.add(
                    Flow.of(RequestContext.class)
                            .map(rc -> rc.withResponseHeader(RawHeader.create("DummyResponse", "ResValue"))));

            return BidiShape.fromFlows(inbound, outbound);
        }));
    }
}
```

#### 中止流(flow)

在某些情况下，管道中的一个阶段可能需要中止流并返回一个`HttpResponse`，例如，在进行身份验证/授权时。在这种情况下，应跳过管道的其余部分，并且请求不应到达squbs服务。要跳过其余的流：

* 需要将流添加到具有`abortable`的生成器中，例如，`b.add(authorization abortable)`。
* 当你需要中止时，在`RequestContext`上调用带有`HttpResponse`的`abortWith`。

下面`DummyAbortableBidiFlow `例子，`authorization`是一个带有`abortable`的bidi flow，并且当用户没有授权的时候中止流：

##### Scala

```scala
class DummyAbortableBidiFlow extends PipelineFlowFactory {

  override def create(context: Context)(implicit system: ActorSystem): PipelineFlow = {

    BidiFlow.fromGraph(GraphDSL.create() { implicit b =>
      import GraphDSL.Implicits._
      val inboundA = b.add(Flow[RequestContext].map { rc => rc.withRequestHeader(RawHeader("keyInA", "valInA")) })
      val inboundC = b.add(Flow[RequestContext].map { rc => rc.withRequestHeader(RawHeader("keyInC", "valInC")) })
      val outboundA = b.add(Flow[RequestContext].map { rc => rc.withResponseHeader(RawHeader("keyOutA", "valOutA"))})
      val outboundC = b.add(Flow[RequestContext].map { rc => rc.withResponseHeader(RawHeader("keyOutC", "valOutC"))})

      val inboundOutboundB = b.add(authorization abortable)

      inboundA ~>  inboundOutboundB.in1
                   inboundOutboundB.out1 ~> inboundC
                   inboundOutboundB.in2  <~ outboundC
      outboundA <~ inboundOutboundB.out2

      BidiShape(inboundA.in, inboundC.out, outboundC.in, outboundA.out)
    })
  }

  val authorization = BidiFlow.fromGraph(GraphDSL.create() { implicit b =>

    val authorization = b.add(Flow[RequestContext] map { rc =>
        if(!isAuthorized) rc.abortWith(HttpResponse(StatusCodes.Unauthorized, entity = "Not Authorized!"))
        else rc
    })

    val noneFlow = b.add(Flow[RequestContext]) // Do nothing

    BidiShape.fromFlows(authorization, noneFlow)
  })
}
```

##### Java

```java
public class DummyAbortableBidiFlow extends japi.PipelineFlowFactory {

    @Override
    public BidiFlow<RequestContext, RequestContext, RequestContext, RequestContext, NotUsed>
    create(Context context, ActorSystem system) {

        return BidiFlow.fromGraph(GraphDSL.create(b -> {
            final FlowShape<RequestContext, RequestContext> inboundA = b.add(
                Flow.of(RequestContext.class)
                    .map(rc -> rc.withRequestHeader(RawHeader.create("keyInA", "valInA"))));
            final FlowShape<RequestContext, RequestContext> inboundC = b.add(
                Flow.of(RequestContext.class)
                    .map(rc -> rc.withRequestHeader(RawHeader.create("keyInC", "valInC"))));
            final FlowShape<RequestContext, RequestContext> outboundA = b.add(
                Flow.of(RequestContext.class)
                    .map(rc -> rc.withRequestHeader(RawHeader.create("keyOutA", "valOutA"))));
            final FlowShape<RequestContext, RequestContext> outboundC = b.add(
                Flow.of(RequestContext.class)
                    .map(rc -> rc.withResponseHeader(RawHeader.create("keyOutC", "valOutC"))));

            final BidiShape<RequestContext, RequestContext> inboundOutboundB =
                b.add(abortable(authorization));

            b.from(inboundA).toInlet(inboundOutboundB.in1());
            b.to(inboundC).fromOutlet(inboundOutboundB.out1());
            b.from(outboundC).toInlet(inboundOutboundB.in2());
            b.to(outboundA).fromOutlet(inboundOutboundB.out2());

            return new BidiShape<>(inboundA.in(), inboundC.out(), outboundC.in(), outboundA.out());
        }));
    }

    final BidiFlow<RequestContext, RequestContext, RequestContext, RequestContext, NotUsed> authorization =
        BidiFlow.fromGraph(GraphDSL.create(b -> {
            final FlowShape<RequestContext, RequestContext> authorization = b.add(
                Flow.of(RequestContext.class)
                    .map(rc -> {
                        if (!isAuthorized()) {
                            rc.abortWith(HttpResponse.create()
                                .withStatus(StatusCodes.Unauthorized()).withEntity("Not Authorized!"));
                        } else return rc;
                    }));

            FlowShape<RequestContext, RequestContext, NotUsed> noneFlow = b.add(
                    Flow.of(RequestContext.class));

            return BidiShape.fromFlows(authorization, noneFlow);
        }));
}
```

一旦流添加了`abortable`，bidi flow就会被连接。此bidi flow检查是否存在`HttpResponse`并绕过或发送请求到下游。上述`DummyAbortableBidiFlow`看起来是这样：

```
                                                +-----------------------------------+
                                                |  +-----------+    +-----------+   |   +-----------+
                  +-----------+   +---------+   |  |           | ~> |  filter   o~~~0 ~>|           |
                  |           |   |         |   |  |           |    |not aborted|   |   | inboundC  | ~> RequestContext
RequestContext ~> | inboundA  |~> |         |~> 0~~o broadcast |    +-----------+   |   |           |
                  |           |   |         |   |  |           |                    |   +-----------+
                  +-----------+   |         |   |  |           | ~> +-----------+   |
                                  | inbound |   |  +-----------+    |  filter   |   |
                                  | outbound|   |                   |  aborted  |   |
                  +-----------+   |   B     |   |  +-----------+ <~ +-----------+   |   +-----------+
                  |           |   |         |   |  |           |                    |   |           |
RequestContext <~ | outboundA | <~|         | <~0~~o   merge   |                    |   | outboundC | <~ RequestContext
                  |           |   |         |   |  |           o~~~~~~~~~~~~~~~~~~~~0 <~|           |
                  +-----------+   +---------+   |  +-----------+                    |   +-----------+
                                                +-----------------------------------+

```