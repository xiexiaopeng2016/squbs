# 编排DSL

编排是服务的主要用例之一，你是否尝试以尽可能多的并发性编排多个服务调用，因此可以获得尽可能好的响应时间，或者你尝试进行多个业务操作，数据写入，数据读取，服务调用等，依赖于彼此等等。
简洁地描述业务逻辑的能力对于使服务易于理解和维护非常重要。编排DSL - `squbs-pattern`的一部分 - 将使异步代码易于编写、阅读和推理。

## 依赖

编排DSL是`squbs-pattern`的一部分。要使用编排DSL，请添加如下依赖：

```scala
"org.squbs" %% "squbs-pattern" % squbsVersion,
"com.typesafe.akka" %% "akka-contrib" % akkaVersion
```

## 开始

让我们从一个简单但完整的编排示例开始。这个编排由3个相互关联的异步任务组成：

1. 加载请求此编排的查看用户。
2. 加载条目。详细信息依赖于查看的用户。
3. 生成条目视图，基于用户和条目数据。

让我们深入到流和细节。

#### Scala

```scala
    // 1. 定义编排actor.
class MyOrchestrator extends Actor with Orchestrator {

    // 2. 提供将接收请求消息的初始expectOnce块.
  expectOnce {
    case r: MyOrchestrationRequest => orchestrate(sender(), r)
  }
  
    // 3. 定义编排 - orchestration函数.
  def orchestrate(requester: ActorRef, request: MyOrchestrationRequest) {
    
    // 4. 根据业务逻辑的需要，使用管道(>>)组合编排流.
    val userF = loadViewingUser
    val itemF = userF >> loadItem(request.itemId)
    val itemViewF = (userF, itemF) >> buildItemView
    
    // 5. 结束并发回编排的结果.   
    for {
      user <- userF
      item <- itemF
      itemView <- itemViewF
    } {
      requester ! MyOrchestrationResult(user, item, itemView)
      context.stop(self)
    }
    
    // 6. 确保停止编排actor, 通过调用
    //    context.stop(self).
  }
  
    // 7. 按照以下模式实现编排功能.
  def loadItem(itemId: String)(seller: User): OFuture[Option[Item]] = {
    val itemPromise = OPromise[Option[Item]]
  
    context.actorOf(Props[ItemActor]) ! ItemRequest(itemId, seller.id)
  
    expectOnce {
      case item: Item => itemPromise success Some(item)
      case e: NoSuchItem => itemPromise success None
    }
  
    itemPromise.future
  }
  
  def loadViewingUser: OFuture[Option[User]] = {
    val userPromise = OPromise[Option[User]]
    ...
    userPromise.future
  }
  
  def buildItemView(user: Option[User], item: Option[Item]): OFuture[Option[ItemView]] = {
    ...
  }
}
```

#### Java
由于Java中的语言限制，相同的实现看起来相当冗长：

```java
    // 1. Define the orchestrator actor.
public class MyOrchestrator extends AbstractOrchestrator {

    // 2. Provide the initial expectOnce in the constructor. It will receive the request message.
    public MyOrchestrator() {
        expectOnce(ReceiveBuilder.match(MyOrchestrationRequest.class, r -> orchestrate(r, sender())).build());
    }
  
    // 3. Define orchestrate - the orchestration function.
    public void orchestrate(MyOrchestrationRequest request, ActorRef requester) {
    
        // 4. Compose the orchestration flow as needed by the business logic.
        static CompletableFuture<Optional<User>> userF = loadViewingUser();
        static CompletableFuture<Optional<Item>> itemF = 
            userF.thenCompose(user -> 
                loadItem(user, request.itemId));
        static CompletableFuture<Optional<ItemView>> itemViewF =
            userF.thenCompose(user ->
            itemF.thenCompose(item ->
                buildItemView(user, item)));
    
        // 5. Conclude and send back the result of the orchestration. 
        userF.thenCompose(user ->
        itemF.thenCompose(item ->
        itemViewF.thenAccept(itemView -> {
            requester.tell(new MyOrchestrationResult(user, item, itemView), self());
            context().stop(self());
        })));
    
        // 6. Make sure to stop the orchestrator actor by calling
        //    context.stop(self).
    }
  
    // 7. Implement the orchestration functions as in the following patterns.
    private CompletableFuture<Optional<Item>> loadItem(User seller, String itemId) {
        CompletableFuture<Optional<Item>> itemF = new CompletableFuture<>();
        context().actorOf(Props.create(ItemActor.class))
            .tell(new ItemRequest(itemId, seller.id), self());
        expectOnce(ReceiveBuilder.
            match(Item.class, item ->
                itemF.complete(Optional.of(item)).
            matchEquals(noSuchItem, e ->
                itemF.complete(Optional.empty())).
            build()
        );
        return itemF;
    }
    
    private CompletableFuture<Optional<User>> loadViewingUser() {
        CompletableFuture<Optional<User>> userF = new CompletableFuture<>();
        ...
        return userF.future
    }
    
    private CompletableFuture<Optional<ItemView>> buildItemView(Optional<User> user, Optional<Item> item) {
        ...
    }
}
```

你可以在这里停下来，稍后再回来阅读剩下的内容，以便深入了解。尽管如此，你仍然可以走得更远，满足你的好奇心。

## 依赖

添加如下依赖到你的`build.sbt`或scala构建文件：

```
"org.squbs" %% "squbs-pattern" % squbsVersion
```

## 核心概念

### 编排

`Orchestrator`是一个由actor继承用于支持编排功能的特质。从技术上讲，它是[Aggregator](http://doc.akka.io/docs/akka/snapshot/contrib/aggregator.html)的子特质，并提供其所有功能。此外，它还提供了功能和语法，允许有效的编排组合，以及在下面详细讨论的创建编排功能时经常需要的实用工具。要使用编排，actor可以简单地继承`Orchestrator`特质。

```scala
import org.squbs.pattern.orchestration.Orchestrator

class MyOrchestrator extends Actor with Orchestrator {
  ...
}
```

`AbstractOrchestrator`是Java超类，面向Java用户。由于Java不支持特质混入，所以它在幕后组合了Actor和编排。因此，Java编制将如下所示:

```java
import org.squbs.pattern.orchestration.japi.AbstractOrchestrator;

public class MyOrchestrator extends AbstractOrchestrator {
    ...
}
```

与`Aggregator`类似，编排通常不声明Akka actor接收块，但允许`expect/expectOnce/unexpect`块来定义在任何点上预期的响应。这些预期代码块通常在`orchestration`函数内部使用。

### Scala: 编排的Future和Promise

编排的`promise`和`future`与[这里](http://docs.scala-lang.org/overviews/core/futures.html)描述的[`scala.concurrent.Future`](http://www.scala-lang.org/api/2.11.4/index.html#scala.concurrent.Future)和[`scala.concurrent.Promise`](http://www.scala-lang.org/api/2.11.4/index.html#scala.concurrent.Promise$)非常相似，只是名字改成了`OFuture`和`OPromise`，标志着它们应当在actor里用于编排。编制版本与工件的并发版本的区别在于没有并发行为。它们在签名中不使用(隐式)`ExecutionContext`，也不用于执行。它们还缺少一些显式异步执行闭包的函数。在actor内部使用时，它们的回调将永远不会在actor范围之外调用。这将消除由于回调而从不同线程同时修改Actor状态的危险。此外，它们还包括性能优化，假设它们总是在Actor内部使用。

**注意:** 不要传递一个`OFuture`到actor外面。提供了隐式转换用于转换`scala.concurrent.Future`和`OFuture`。

```scala
import org.squbs.pattern.orchestration.{OFuture, OPromise} 
```

### Java: `CompletableFuture`

Java的`CompletableFuture`及其**同步**回调被用作值的占位符，以便在编排过程中得到满足。同步回调确保发生在它的线程上的future的处理已经完成。在编排模型中，它将会是编排actor被安排接收和处理消息的线程。这确保了没有并发地掩盖actor的状态。所有回调都在actor的范围内处理。

```java
import java.util.concurrent.CompletableFuture;
```

### 异步的编排函数

编排函数是一个被编排流调用，去执行单一编排任务的函数，例如进行服务或数据库调用。一个编排函数必须符合以下准则：

1. 它必须以非Future参数作为输入。根据Scala的当前限制，参数的数目最多可达22。在所有情况下, 这些函数不应该有那么多参数。java风格的编排没有暴露这个限制。在所有情况下，这些函数不应该有那么多参数。

2. Scala函数可能[柯里化](http://docs.scala-lang.org/tutorials/tour/currying.html)，将直接输入与管道(future)输入分隔开。管道输入必须是在柯里化函数中最后一组参数。

3. 它必须导致异步执行。异步执行通常是通过发送一条由不同的actor处理的消息来完成的。

4. Scala实现必须返回一个`OFuture`(编排future)。Java实现必须返回一个`CompletableFuture`。

让我们来看看几个编排功能的例子:

#### Scala

```scala
def loadItem(itemId: String)(seller: User): OFuture[Option[Item]] = {
  val itemPromise = OPromise[Option[Item]]
  
  context.actorOf(Props[ItemActor]) ! ItemRequest(itemId, seller.id)
  
  expectOnce {
    case item: Item => itemPromise success Some(item)
    case e: NoSuchItem => itemPromise success None
  }
  
  itemPromise.future
}
```

这个例子，函数是柯里化的。`itemId`参数是同步传递的，而`seller`是异步传递的。

#### Java

```java
private CompletableFuture<Optional<Item>> loadItem(User seller, String itemId) {
    CompletableFuture<Optional<Item>> itemF = new CompletableFuture<>();
    context().actorOf(Props.create(ItemActor.class))
        .tell(new ItemRequest(itemId, seller.id), self());
    expectOnce(ReceiveBuilder.
        match(Item.class, item ->
            itemF.complete(Optional.of(item)).
        matchEquals(noSuchItem, e ->
            itemF.complete(Optional.empty())).
        build()
    );
    return itemF;
}
```

我们首先创建一个`OPromise`(Scala)或`CompletableFuture` (Java)来保存最终的值。然后，我创建一个`ItemRequest`，并发送给另外一个actor。此actor现在将异步获取`item`。一旦我们发送了请求，我们就用`expectOnce`注册一个回调。`expectOnce`里面的代码是一个`Receive`，它将根据`ItemActor`发回的响应执行。在任何情况下，它将让`promise`变成`success`或让`CompletableFuture`变成`complete`。最后，我们发出`future`。不返回一个`promise`的原因是它是可变的。我们不想从函数中返回一个可变的对象。在其上调用`future`将提供`OPromise`的不可变视图，即`OFuture`。不幸的是，这不适用于Java。

下面的示例在逻辑上与第一个示例相同，只是使用`ask`而不是`tell`:

#### Scala

```scala
private def loadItem(itemId: String)(seller: User): OFuture[Option[Item]] = {
  
  import context.dispatcher
  implicit val timeout = Timeout(3 seconds)
  
  (context.actorOf[ItemActor] ? ItemRequest(itemId, seller.id)).mapTo[Option[Item]]
}
```

此例中，ask `?` 操作返回了一个[`scala.concurrent.Future`](http://www.scala-lang.org/api/2.11.4/index.html#scala.concurrent.Future)。`Orchestrator`特质提供了`scala.concurrent.Future`和`OFuture`的隐式转换，所以ask `?`的结果转换为这个函数声明的返回类型`OFuture` 且无需显示调用转换。

#### Java

```java
private CompletableFuture<Optional<Item>> loadItem(User seller, String itemId) {
    CompletableFuture<Optional<Item>> itemF = new CompletableFuture<>();
    Timeout timeout = new Timeout(Duration.create(5, "seconds"));
    ActorRef itemActor = context().actorOf(Props.create(ItemActor.class));
    ask(itemActor, new ItemRequest(itemId, seller.id), timeout).thenComplete(itemF);
    return itemF;
}
```

`AbstractOrchestrator`为`ask`提供了方便的函数，允许使用`thenComplete`操作将结果直接填充到`CompletableFuture`中。

这个时候用`ask`或`?`，似乎需要编写更少的代码，但它既不性能，也不像`expect/expectOnce`那样灵活 。`expect`块中的逻辑也可用于结果的进一步转换。同样可以通过`ask`返回的future使用回调来实。但是，由于以下原因，无法轻松地补偿性能：

1. Ask将创建一个新的actor作为响应接收者。
2. 从[`scala.concurrent.Future`](http://www.scala-lang.org/api/2.11.4/index.html#scala.concurrent.Future)到`OFuture`的转换以及Java API的`fill`操作需要将消息发送回`orchestrator`，从而添加一个消息跳转，同时增加了延迟和CPU。

测试显示，当使用`ask`，而不是`expect/expectOnce`时，有更高的延迟和CPU利用率。

### 组合

#### Scala

管道，或者`>>`符号使用一个或多个编排future `OFuture`，并使其结果作为`orchestration`函数的输入。当所有代表函数输入的 `OFuture`都被解决时，`orchestration`函数调用将异步发生。

管道是编排DSL的主要组件，它允许根据其输入和输出组合编排功能。编排流是由编排声明隐式确定的，或者通过管道来声明编排流。

当多个`OFuture`被输送到一个编排函数时，`OFuture`需要以逗号分隔并括在圆括号中，构造一个`OFuture`的元组作为输入。元组中的元素个数，它们的`OFuture`类型必须与函数参数和类型匹配，或者是在[柯里化](http://docs.scala-lang.org/tutorials/tour/currying.html)的情况下的最后一组参数，否则编译将失败。此类错误通常也由IDE捕获。

下面的例子展示了一个简单的编排声明以及使用前面章节声明的`loadItem`函数的流：

```scala
val userF = loadViewingUser
val itemF = userF >> loadItem(request.itemId)
val itemViewF = (userF, itemF) >> buildItemView
```

上面的代码可以按如下描述：

* 首先调用`loadViewingUser`(不带输入参数)。
* 当查看用户变成可用时，使用查看用户作为调用`loadItem`的输入(在本例中，前面有个itemId可用)。本例中，`loadItem`遵循上面声明的编排函数的确切签名。
* 当`user`和`item`可用时，调用`buildItemView`。

#### Java

使用`CompletableFuture.thenCompose()`函数可以完成多个`CompletableFuture`的组合。每个`thenCompose`接受一个`lambda`，使用解决的的`future`值作为输入。这将在`CompletableFuture`完成时调用。

最好是用一个例子来描述这样的组成：

```java
static CompletableFuture<Optional<User>> userF = loadViewingUser();
static CompletableFuture<Optional<Item>> itemF = 
    userF.thenCompose(user -> 
        loadItem(user, request.itemId));
static CompletableFuture<Optional<ItemView>> itemViewF =
    userF.thenCompose(user ->
    itemF.thenCompose(item ->
        buildItemView(user, item)));
```

上述流可被描述如下:

* 首先调用`loadViewingUser`。
* 当查看用户变为可用时，使用查看用户作为调用`loadItem`的输入(在本例中，前面有个itemId可用)。
* 当`user`和`item`可用时，调用`buildItemView`。

## 编排实例生命周期

编排通常是一次性的actor。它们接收初始请求，然后根据调用的编排函数发出的请求进行多个响应。

为了允许一个编排器服务多个编排请求，编排器必须结合每个请求的输入和响应，并根据不同的请求隔离它们。这将使其开发更加复杂化，并且最终结果很可能不是一个清晰的编排表述，我们在这些例子中可以看到的。创建一个新的actor是足够廉价的，我们能够容易地为每个编排请求创建一个新的编排actor。

编制回调的最后一部分应该停止actor。在Scala中，通过调用`context.stop(self)`或者`context stop self` 如果建议使用中缀表示法。ava实现需要调用`context().stop(self());`。


## 完成编排流

在这里，我们把上述所有的概念放在一起。用更完整的解释来重复上面的例子：

#### Scala

```scala
    // 1. Define the orchestrator actor.
class MyOrchestrator extends Actor with Orchestrator {

    // 2. Provide the initial expectOnce block that will receive the request message.
    //    After this request message is received, the same request will not be
    //    expected again for the same actor.
    //    The expectOnce likely has one case match which is the initial request and
    //    uses the request arguments or members, and the sender() to call the high
    //    level orchestration function. This function is usually named orchestrate.
  expectOnce {
    case r: MyOrchestrationRequest => orchestrate(sender(), r)
  }
  
    // 3. Define orchestrate. Its arguments become immutable by default
    //    allowing developers to rely on the fact these will never change.
  def orchestrate(requester: ActorRef, request: MyOrchestrationRequest) {
  
    // 4. If there is anything we need to do synchronously to setup for
    //    the orchestration, do this in the first part of orchestrate.
  
    // 5. Compose the orchestration flow using pipes as needed by the business logic.
    val userF = loadViewingUser
    val itemF = userF >> loadItem(request.itemId)
    val itemViewF = (userF, itemF) >> buildItemView
    
    // 6. End the flow by calling functions composing the response(s) back to the
    //    requester. If the composition is very large, it may be more readable to
    //    use for-comprehensions rather than a composition function with very large
    //    number of arguments. There may be multiple such compositions in case partial
    //    responses are desired. This example shows the use of a for-comprehension
    //    just for reference. You can also use an orchestration function with
    //    3 arguments plus the requester in such small cases.
    
    for {
      user <- userF
      item <- itemF
      itemView <- itemViewF
    } {
      requester ! MyOrchestrationResult(user, item, itemView)
      context.stop(self)
    }
    
    // 7. Make sure the last response stops the orchestrator actor by calling
    //    context.stop(self).
  }
  
    // 8. Implement the asynchronous orchestration functions inside the
    //    orchestrator actor, but outside the orchestrate function.
  def loadItem(itemId: String)(seller: User): OFuture[Option[Item]] = {
    val itemPromise = OPromise[Option[Item]]
  
    context.actorOf[ItemActor] ! ItemRequest(itemId, seller.id)
  
    expectOnce {
      case item: Item    => itemPromise success Some(item)
      case e: NoSuchItem => itemPromise success None
    }
  
    itemPromise.future
  }
  
  def loadViewingUser: OFuture[Option[User]] = {
    val userPromise = OPromise[Option[User]]
    ...
    userPromise.future
  }
  
  def buildItemView(user: Option[User], item: Option[Item]): OFuture[Option[ItemView]] = {
    ...
  }
}
```

#### Java

```java
    // 1. Define the orchestrator actor.
public class MyOrchestrator extends AbstractOrchestrator {

    // 2. Provide the initial expectOnce in the constructor. It will receive the request message.
    public MyOrchestrator() {
        expectOnce(ReceiveBuilder.match(MyOrchestrationRequest.class, r -> orchestrate(r, sender())).build());
    }
  
    // 3. Define orchestrate - the orchestration function.
    public void orchestrate(MyOrchestrationRequest request, ActorRef requester) {

        // 4. If there is anything we need to do synchronously to setup for
        //    the orchestration, do this in the first part of orchestrate.
  
        // 5. Compose the orchestration flow as needed by the business logic.
        static CompletableFuture<Optional<User>> userF = loadViewingUser();
        static CompletableFuture<Optional<Item>> itemF = 
            userF.thenCompose(user -> 
                loadItem(user, request.itemId));
        static CompletableFuture<Optional<ItemView>> itemViewF =
            userF.thenCompose(user ->
            itemF.thenCompose(item ->
                buildItemView(user, item)));
    
        // 6. Conclude and send back the result of the orchestration. 
        userF.thenCompose(user ->
        itemF.thenCompose(item ->
        itemViewF.thenAccept(itemView -> {
            requester.tell(new MyOrchestrationResult(user, item, itemView), self());
            context().stop(self());
        })));
    
        // 7. Make sure to stop the orchestrator actor by calling
        //    context.stop(self).
    }
  
    // 8. Implement the orchestration functions as in the following patterns.
    private CompletableFuture<Optional<Item>> loadItem(User seller, String itemId) {
        CompletableFuture<Optional<Item>> itemF = new CompletableFuture<>();
        context().actorOf(Props.create(ItemActor.class))
            .tell(new ItemRequest(itemId, seller.id), self());
        expectOnce(ReceiveBuilder.
            match(Item.class, item ->
                itemF.complete(Optional.of(item)).
            matchEquals(noSuchItem, e ->
                itemF.complete(Optional.empty())).
            build()
        );
        return itemF;
    }
    
    private CompletableFuture<Optional<User>> loadViewingUser() {
        CompletableFuture<Optional<User>> userF = new CompletableFuture<>();
        ...
        return userF;
    }
    
    private CompletableFuture<Optional<ItemView>> buildItemView(Optional<User> user, Optional<Item> item) {
        ...
    }
}
```

## 编排函数的复用

#### Scala

编排函数通常依赖于`Orchestrator`特质提供的工具，无法独立运行。但是，在许多情况下，需要在多个编排器中，以不同的编排方式重用编排函数。在这种情况下，重要的是将编排函数分成不同的特质，这些特质将被混入到每个编排器中。特质必须能够访问对编排功能，并且需要一个自我引用到`Orchestrator`。下面显示了这样一个特质的示例:

```scala
trait OrchestrationFunctions { this: Actor with Orchestrator =>

  def loadItem(itemId: String)(seller: User): OFuture[Option[Item]] = {
    ...
  }
}
```

上面例子中的`this: Actor with Orchestrator`是一个类型化的自引用。它告诉Scala编译器，这个特质只能够混入到一个同样也是`Orchestrator`的`Actor`，因此可以访问`Actor`和`Orchestrator`提供的工具。

要在编排器中使用`OrchestrationFunctions`特质，你只需将这个特质混入到一个编排器中，如下所示：

```scala
class MyOrchestrator extends Actor with Orchestrator with OrchestrationFunctions {
  ...
}
```

#### Java

Java需要一个单独的层次结构，不能支持特征或多重继承。重用是通过扩展`AbstractOrchestrator`，实现编排功能，而剩下的抽象部分由编排的具体实现来实现的，如下所示：

```java
abstract class MyAbstractOrchestrator extends AbstractOrchestrator {

    protected CompletableFuture<Optional<Item>> loadItem(User seller, String itemId) {
        CompletableFuture<Optional<Item>> itemF = new CompletableFuture<>();
        ...
        return itemF;
    }
    
    protected CompletableFuture<Optional<User>> loadViewingUser() {
        CompletableFuture<Optional<User>> userF = new CompletableFuture<>();
        ...
        return userF;
    }
    
    protected CompletableFuture<Optional<ItemView>> buildItemView(Optional<User> user, Optional<Item> item) {
        ...
    }
}
```

编配器的具体实现只是从上面的`MyAbstractOrchestrator`扩展而来，并实现了不同的编配。

## 确保响应的唯一性

当使用`expect`或`expectOnce`时，我们受到单个`expect`块的模式匹配功能的限制，它的作用域有限，无法区分多个编排函数中的多个`expect`块之间的匹配。在同一编排函数中声明`expect`之前，接收到的消息与发送的请求消息之间没有逻辑链接。对于复杂的编排，我们可能会遇到消息混淆的问题。响应没有与正确的请求关联，也没有正确处理。这是解决这个问题有几个策略：

如果初始消息的接收者，和响应消息的发送者是唯一的，那么匹配可能包括对消息的发送者的引用，就像在下面的示例模式匹配中一样。

#### Scala

```scala
def loadItem(itemId: String)(seller: User): OFuture[Option[Item]] = {
  val itemPromise = OPromise[Option[Item]]
  
  val itemActor = context.actorOf(Props[ItemActor])
  itemActor ! ItemRequest(itemId, seller.id)
  
  expectOnce {
    case item: Item    if sender() == itemActor => itemPromise success Some(item)
    case e: NoSuchItem if sender() == itemActor => itemPromise success None
  }
  
  itemPromise.future
}
```

#### Java

```java
private CompletableFuture<Optional<Item>> loadItem(User seller, String itemId) {
    CompletableFuture<Optional<Item>> itemF = new CompletableFuture<>();
    ActorRef itemActor = context().actorOf(Props.create(ItemActor.class));
    itemActor.tell(new ItemRequest(itemId, seller.id), self());
    expectOnce(ReceiveBuilder.
        match(Item.class, item -> itemActor.equals(sender()), item ->
            itemF.complete(Optional.of(item)).
        matchEquals(noSuchItem, e -> itemActor.equals(sender()), e ->
            itemF.complete(Optional.empty())).
        build()
    );
    return itemF;
}
```

或者，`Orchestrator`特质提供了一个消息`id`生成器，它在与actor实例组合时是唯一的。我们可以使用这个`id`生成器生成一个唯一的消息`id`。接受此类消息的Actor只需要将此消息`id`作为响应消息的一部分返回。下面的示例显示了使用消息`id`生成器的编排函数。

#### Scala

```scala
def loadItem(itemId: String)(seller: User): OFuture[Option[Item]] = {
  val itemPromise = OPromise[Option[Item]]
  
  // Generate the message id.
  val msgId = nextMessageId  
  context.actorOf(Props[ItemActor]) ! ItemRequest(msgId, itemId, seller.id)
  
  // Use the message id as part of the response pattern match. It needs to
  // be back-quoted as to not be interpreted as variable extractions, where
  // a new variable is created by extraction from the matched object.
  expectOnce {
    case item @ Item(`msgId`, _, _) => itemPromise success Some(item)
    case NoSuchItem(`msgId`, _)     => itemPromise success None
  }
  
  itemPromise.future
}
```

#### Java

```java
private CompletableFuture<Optional<Item>> loadItem(User seller, String itemId) {
    CompletableFuture<Optional<Item>> itemF = new CompletableFuture<>();
    long msgId = nextMessageId();
    context().actorOf(Props.create(ItemActor.class))
        .tell(new ItemRequest(msgId, itemId, seller.id), self());
    expectOnce(ReceiveBuilder.
        match(Item.class, item -> item.msgId == msgId, item ->
            itemF.complete(Optional.of(item)).
        match(NoSuchItem.class, e -> e.msgId == msgId, e ->
            itemF.complete(Optional.empty())).
        build()
    );
    return itemF;
}
```
