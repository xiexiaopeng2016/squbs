# Unicomplex & Cube 引导程序

squbs自带一个默认的引导类`org.squbs.unicomplex.Bootstrap`。可以从IDE、命令行、sbt、甚至Maven启动。引导程序扫描类路径并在每个加载的jar资源中查找`META-INF/squbs-meta.&lt;ext&gt;`。如果squbs元数据可用, 则jar资源将被视为squbs cube或扩展, 并根据元数据声明进行初始化。然后, 引导程序首先初始化扩展、cube, 最后是服务处理程序, 不管它们在类路径中的顺序如何。

在正常情况下, 引导细节没有多少意义。然而，可能需要以不同的方式通过编程来引导squbs。这在需要定制配置和需要并行运行的测试用例中尤其常见。有关详细信息, 请参阅[测试](testing.md)。引导squbs的语法如下：

**选项 1)** 使用自定义配置启动

```
UnicomplexBoot(customConfig)
  .createUsing {(name, config) => ActorSystem(name, config)}
  .scanResources()
  .initExtensions
  .stopJVMOnExit
  .start()
```

**选项 2)** 使用默认配置启动

```
UnicomplexBoot {(name, config) => ActorSystem(name, config)}
  .scanResources()
  .initExtensions
  .stopJVMOnExit
  .start()  
```

让我们看一下每个组件。

1. 创建`UnicomplexBoot(boot)`对象。通过将一个自定义配置(Config)或者actorSystemCreator闭包传入`UnicomplexBoot.apply()`来完成。

2. 上面示例中显示的配置对象为`customConfig`。这是从Typesafe配置库的解析函数中获得的配置对象。此配置对象尚未与`reference.conf`合并。它是可选的，替代其它`application.conf`配置中的定义。

大多数用例都希望以这种方式创建ActorSystem, 因此不需要提供函数。`createUsing`可以完全避免。

3. ActorSystem创建者传递一个函数或闭包来创建ActorSystem。实际创建发生在启动阶段(下面第7项)。默认的函数是`{(name, config) => ActorSystem(name, config)}`。传入的`name`是从配置文件中读取的ActorSystem名称。这个`config`是与任何提供的配置合并后加载的配置对象。大多数用例都希望以这种方式创建ActorSystem, 因此不需要提供函数。`createUsing` 完全可以避免。

`.scanResources("component1/META-INF/squbs-meta.conf", "component2/META-INF/squbs-meta.conf")`将扫描你的类路径以及另外所给资源。如果你不想扫描类路径，传入`withClassPath = false`或者在资源列表前的第一个参数仅传入`false`：`.scanResources(withClassPath = false, "component1/META-INF/squbs-meta.conf", "component2/META-INF/squbs-meta.conf")`。

4. 使用`scanResources()`函数扫描组件查找cube、服务或者扩展。这是强制性的，否则将没有组件启动。如果没有传递参数, squbs引导程序将扫描其类加载器。测试用例可能希望只扫描某些组件。这通过可以传入另外的`squbs-meta.conf`文件位置(作为`scanResources`的一个变量参数)完成，比如`.scanResources("component1/META-INF/squbs-meta.conf", "component2/META-INF/squbs-meta.conf")`。将扫描你的类路径和另外所给的资源。如果你不想扫描类路径，传入`withClassPath = false`，或者`false`作为在资源列表前的第一个参数: `.scanResources(withClassPath = false, "component1/META-INF/squbs-meta.conf", "component2/META-INF/squbs-meta.conf")`。

5. 使用`initExtension`函数初始化扩展。这将初始化扫描到的所有扩展。扩展的初始化将在ActorSystem创建前完成。对于多个Unicomplex的用例(即多个ActorSystem)，同一扩展不能多次初始化。一个扩展只能被一个测试用例使用。在某些测试用例中, 我们根本不希望初始化扩展, 也完全不会调用`initExtension`。

6. 在退出时停止JVM。这是通过调用`stopJVMOnExit`函数来实现的。这个选项通常不应该用于测试用例。它用于squbs的引导程序，以确保squbs关闭并正确退出。它是由squbs的引导使用, 以确保squbs关闭和正确退出。

7. 通过调用`start()`启动Unicomplex。这是一个强制性的步骤。没有它，就没有ActorSystem启动，也就没有Actor能够运行。start调用会阻塞, 直到系统完全启动和运行, 或者产生一个超时。当`start`出现超时的时候, 某些组件可能仍在初始化, 使系统处于`Initializing`状态。然而，在超时的时候，任何单个组件故障都会将系统状态转换为`Failed`。这将允许系统组件(如系统诊断程序)运行并完成。默认的启动超时设置为60秒。对于期望超时的测试, 可以设置更小的值，通过向`start()`传入要求的超时，例如`start(Timeout(5 seconds))`或更短的`start(5 seconds)`，使用从duration到timeout的隐式转换。

# 配置解析

squbs选择一个应用程序配置, 并将其与类路径下的聚合的`application.conf`和`reference.conf`合并。正在合并的应用程序配置从以下顺序选择。

1. 如果创建boot对象时提供了一个配置，这个配置将被选中。即上例中的`customConfig`字段。

2. 如果在外部配置目录中提供了一个`application.conf`文件, 则此`application.conf`
将被选中。外部配置目录是通过`squbs.external-config-dir`属性来配置的，默认是`squbsconfig`。并不是说提供的配置或外部配置不能更改或覆盖此目录(因为目录本身是使用config属性确定的)。

3. 否则，将使用与应用程序一起提供的`application.conf`, 如果有的话。然后，回到`reference.conf`。

# Drop-in 模块化系统

squbs将应用划分到称为cubes的模块中。squbs中的模块旨在模块隔离中运行，也可以在扁平类路径上运行。模块化隔离旨在实现模块的真正松散耦合，而不会由于依赖关系而导致任何类路径冲突。

当前实现的引导程序基于扁平类路径。在引导时，squbs通过类路径扫描自动检测模块。扫描到的cube自动被检测并启动。

## Cube Jar文件

所有cube由一个含有cube本身逻辑的顶级jar文件表示。所有的cube都必须有cube元数据，存在于`META-INF/squbs-meta.<ext>`。支持的扩展名包括.conf, .json, .properties。遵守[Typesafe配置](https://github.com/typesafehub/config)格式。

至少，cube元数据要唯一标识cube和版本，并声明和配置以下一个或多个：

*Actor*: 标识squbs自动启动的`WellKnownActor`。

*Service*: 标识一个squbs服务。

*Extension*: 标识一个squbs框架扩展。扩展入口点必须继承于`org.squbs.lifecycle.ExtensionLifecycle`特质。
    
## 配置解析

当多个cube试图提供它们内部的`application.conf`时，为每个cube提供`application.conf`会导致问题。合并此类配置的优先级规则没有定义。推荐的做法是，cube仅提供一个`reference.conf`，并且可以被外部`application.conf`覆盖以进行部署。

## Well-Known Actors

Well-known actor 只是[Akka文档](http://doc.akka.io/docs/akka/2.3.13/scala.html)所定义的[Akka actors](http://doc.akka.io/docs/akka/2.3.13/scala/actors.html)。它们由一个监管者actor启动，其为每一个cube创建。监管者有和cube相同的名称。因此，任何well-known actor有一个路径`/CubeName/ActorName`，并可以用`ActorSelection`调用在`/user/CubeName/ActorName`下面查找。

一个well-known actor可以作为一个单独的actor或者一个with路由器启动。为了将一个well-known actor声明为一个路由器，增加：`with-router = true`到actor声明中。对well-known actor的路由器, 调度器和邮箱的配置在`reference.conf`或`application.conf`中完成，根据Akka文档。

下面是一个简单的cube声明`META-INF/squbs-meta.conf`，声明了一个well known actor：

```
cube-name = org.squbs.bottlecube
cube-version = "0.0.2"
squbs-actors = [
  {
    class-name = org.squbs.bottlecube.LyricsDispatcher
    name = lyrics
    with-router = false  # Optional, defaults to false
    init-required = false # Optional
  }
]
```

`init-required`参数用于那些需要发回其完全初始化状态的actor, 以便将系统视为已初始化。有关startup/initialization hooks的完整讨论, 请参阅[运行时生命周期 & API](lifecycle.md)文档的[Startup Hooks](lifecycle.md#startup-hooks)部分。

如果一个actor配置了`with-router`(with-router = true)和一个非默认调度器，其目的通常是将actor(routee)调度到非默认调度器上。路由器将占用well known actor的名称, 而不是routee(你的actor实现)的。路由器上设置的调度器只会影响路由器, 而不会影响routee。要影响routee, 您需要为routees创建一个单独的配置, 并将"/*"追加到该名称。接下来, 您要将routee中的调度器配置为下面的示例。

```
akka.actor.deployment {

  # Router configuration
  /bottlecube/lyrics {
    router = round-robin-pool
    resizer {
      lower-bound = 1
      upper-bound = 10
    }
  }

  # Routee configuration. Since it has a '*', the name has to be quoted.
  "/bottlecube/lyrics/*" {
    # Configure the dispatcher on the routee.
    dispatcher = blocking-dispatcher
  }
```

路由器概念、示例和配置已在[Akka文档](http://doc.akka.io/docs/akka/2.3.13/scala/routing.html)中记录。

## 服务

在[实现HTTP(S)服务](http-services.md)章节对服务有详细描述。在`META-INF/squbs-meta.conf`中声明的服务元数据，如下例所示：

```
cube-name = org.squbs.bottlesvc
cube-version = "0.0.2"
squbs-services = [
  {
    class-name = org.squbs.bottlesvc.BottleSvc
    web-context = bottles # You can also specify bottles/v1, for instance.
    
    # The listeners entry is optional, and defaults to 'default-listener'.
    listeners = [ default-listener, my-listener ]
    
    # Optional, defaults to a default pipeline.
    pipeline = some-pipeline
    
    # Optional, disables the default pipeline if set to off.  If missing, it is set to on.
    defaultPipeline = on
    
    # Optional, only applies to actors.
    init-required = false
  }
]
```

请看[服务注册](http-services.md#service-registration)的详细描述。

## 扩展

squbs中的扩展是用于环境启动的低层设施。扩展的初始化器需要继承于`org.squbs.lifecycle.ExtensionLifecycle`特质，并覆盖合适的回调。扩展有很大的能力来内省系统，并提供额外的功能，而这些squbs本身并没有提供。在同一个cube中，不能将扩展与actor或服务组合在一起。

扩展连续地启动, 一个接一个。扩展的提供者可以为扩展启动提供一个序号, 指定:
    sequence = [number]
在扩展定义中. 如果没有指定`sequence`，则默认为`Int.maxValue`。这意味着它将在所有提供了序号的扩展之后启动。如果有多个扩展不指定序号或指定了相同的序号, 则启动它们的顺序是随机的。关机顺序与启动顺序相反。

# 关闭

squbs运行时，可以向`Unicomplex()`发送`GracefulStop`消息来正确关闭。
 
默认的启动主方法，`org.squbs.unicomplex.Bootstrap`，注册了一个JVM关闭钩子，其发送`GracefulStop`消息给`Unicomplex`。因此，如果一个squbs应用使用了默认的main方法启动，当JVM收到一个`SIGTERM`时，系统会优雅的关闭。

如果某个其他监视进程负责关闭应用程序(例如JSW), 则可以将`org.squbs.unicomplex.Shutdown`设置为优雅的关闭系统的主要方法。这个`Shutdown`主方法也会发送一个`GracefulStop`消息给`Unicomplex`。

在某些用例中，最好为关机添加一个延迟。例如, 如果一个负载平衡器每5秒钟检查一次应用程序的健康状况, 并且应用程序在一次健康检查后关闭1秒, 应用程序将在接下来的4秒内继续收到请求, 直到下一次健康检查; 但是, 它无法满足这些请求。如果使用上述方法之一`org.squbs.unicomplex.Bootstrap`或`org.squbs.unicomplex.Shutdown`, 则可以为关机添加一个延迟，通过在配置中添加以下内容:

```
squbs.shutdown-delay = 5 seconds
```

有了以上配置，向`Unicomplex`发送的`GracefulStop`消息将按计划推迟5秒发送。

在收到`GracefulStop`消息后，`Unicomplex`actor将停止服务并传播`GracefulStop`消息给所有的cube监管者。每一个监管者负责停止它的cube内的actor(通过传播`GracefulStop`消息给它的想要执行优雅停止的孩子)，确认它们停止成功或者在超时后重发一个`PoisonPill`，然后停止它自己。一旦所有的cube监管者和服务都已停止，squbs系统关闭。然后，一个关闭钩将被调用，来停止所有的扩展，并最终退出JVM。

web容器目前没有标准的控制台, 允许squbs的用户自己构建。web控制台可以提供适当的用户关闭, 通过向Unicomplex发送停止消息, 如下所示:

```
  Unicomplex() ! GracefulStop
```
