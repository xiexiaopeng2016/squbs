# 自定义流控制

Akka流/响应流需要与服务器的[运行时生命周期](lifecycle.md)集成。为此，通过[`PerpetualStream`](perpetualstream.md)基础结构提供了自动化或半自动化的集成。如果您需要对流进行更细粒度的控制，以下几节将介绍此类功能。

### 依赖

通常，您不需要添加额外的依赖项。这些类是以下依赖项的一部分：

```scala
"org.squbs" %% "squbs-unicomplex" % squbsVersion
```


## 制作一个对生命周期敏感的源

如果希望有一个与squb的生命周期事件完全相关的源，则可以使用`LifecycleManaged`来包装任何源。

**Scala**

```scala
val inSource = <your-original-source>
val aggregatedSource = LifecycleManaged().source(inSource)
```

**Java**

```java
final Source inSource = <your-original-source>
final Source aggregatedSource = new LifecycleManaged().source(inSource);
```

生成的源将是具体化为`(T, ActorRef)`的聚合源，其中`T`是`inSource`的具体化类型，并且`ActorRef`是触发的actor的具体化类型，其接收来自Unicomplex(squbs容器)的事件。

在生命周期变为生命周期之前，聚合源不会从原始源发出Active，并且在生命周期状态变为时停止发射元素并关闭流Stopping。

聚合的源不会从原始源发出，直到生命周期变得`Active`，并在生命周期状态变为`Stopping`后停止发出元素并关闭流。

## 自定义 Aggregated Triggered Source

如果您希望你的流可以启用/禁用自定义事件，则可以与自定义触发源集成，元素`true`将启用，`false`将禁用。

请注意，`Trigger`接受一个参数`eagerComplete`，其默认为`false`，在Scala中，但必须在Java中传递。如果`eagerComplete`设置为`false`，则触发源的actor的完成和/或终止将分离触发器。如果设置为true，则此类终止将完成流。

##### Scala

```scala
import org.squbs.stream.TriggerEvent._

val inSource = <your-original-source>
val trigger = <your-custom-trigger-source>.collect {
  case 0 => DISABLE
  case 1 => ENABLE
}

val aggregatedSource = new Trigger().source(inSource, trigger)
```

##### Java

```java
import static org.squbs.stream.TriggerEvent.DISABLE;
import static org.squbs.stream.TriggerEvent.ENABLE;

final Source<?, ?> inSource = <your-original-source>;
final Source<?, ?> trigger = <your-custom-trigger-source>.collect(new PFBuilder<Integer, TriggerEvent>()
    .match(Integer.class, p -> p == 1, p -> ENABLE)
    .match(Integer.class, p -> p == 0, p -> DISABLE)
    .build());

final Source aggregatedSource = new Trigger(false).source(inSource, trigger);
```

## 为触发器自定义生命周期事件

如果您想响应更多的生命周期事件，而不仅仅是`Active`和`Stopping`，例如，您希望`Failed`时，还要停止流，则可以修改生命周期事件映射。

##### Scala

```scala
import org.squbs.stream.TriggerEvent._

val inSource = <your-original-source>
val trigger = Source.actorPublisher[LifecycleState](Props.create(classOf[UnicomplexActorPublisher]))
  .collect {
    case Active => ENABLE
    case Stopping | Failed => DISABLE
  }

val aggregatedSource = new Trigger().source(inSource, trigger)
```

##### Java

```java
import static org.squbs.stream.TriggerEvent.DISABLE;
import static org.squbs.stream.TriggerEvent.ENABLE;

final Source<?, ?> inSource = <your-original-source>;
final Source<?, ActorRef> trigger = Source.<LifecycleState>actorPublisher(Props.create(UnicomplexActorPublisher.class))
    .collect(new PFBuilder<Integer, TriggerEvent>()
        .matchEquals(Active.instance(), p -> ENABLE)
        .matchEquals(Stopping.instance(), p -> DISABLE)
        .matchEquals(Failed.instance(), p -> DISABLE)
        .build()
    );

final Source aggregatedSource = new Trigger(false).source(inSource, trigger);
```
