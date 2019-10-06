# 运行时生命周期 & API

生命周期确实是基础结构的一个关注点。应用程序很少需要接触，甚至不知道系统的生命周期。系统组件、管理控制台、或者应用程序组件或actor，其需要长时间进行初始化，并且需要在系统可用于通信之前完全初始化，这就必须知道系统生命周期。后者包括的功能有缓存控制器、缓存加载器、设备初始化器，等等。

squbs运行时暴露了以下生命周期状态：

* **Starting** - squbs启动时的初始化状态。

* **Initializing** - Unicomplex已启动。服务启动中。Cubes启动中。等待初始化报告。

* **Active** - 准备就绪，可以开始工作和接受服务调用。

* **Failed** - Cubes没有正确启动。

* **Stopping** - 在Unicomplex收到`GracefulStop`消息。终止cube，actor并解绑服务。

* **Stopped** - squbs运行时停止。Unicomplex已终止。ActorSystem已终止。

## 生命周期钩子

多数actor不关心它们什么时候启动或者关闭。然而，可能有一类actor需要执行某些初始化, 然后才能到达接受常规通信的状态。同样地，某些actor也关心在关闭之前被通知，使它们可以正确地清理。生命周期钩子就是为此而存在的。 

可以让你的actor注册生命周期事件，通过发送 `ObtainLifecycleEvents(states: LifecycleState*)`给`Unicomplex()`。然后，一旦系统状态改变，你的actor将收到生命周期状态。

你也可以通过发送`SystemState`消息给`Unicomplex()`获得当前状态。你将得到上面状态的一种的响应。所有的系统状态继承于`org.squbs.unicomplex.LifecycleState`，并且都是`org.squbs.unicomplex`包的一部分，如下所示：

* `case object Starting extends LifecycleState`
* `case object Initializing extends LifecycleState`
* `case object Active extends LifecycleState`
* `case object Failed extends LifecycleState`
* `case object Stopping extends LifecycleState`
* `case object Stopped extends LifecycleState`
 

## 启动(Startup)钩子

一个希望参与初始化的actor必须在squbs元数据`META-INF/squbs-meta.conf`中注明如下:

```
cube-name = org.squbs.bottlecube
cube-version = "0.0.2"
squbs-actors = [
  {
    class-name = org.squbs.bottlecube.LyricsDispatcher
    name = lyrics
    with-router = false  # Optional, defaults to false
    init-required = true # Tells squbs we need to wait for this actor to signal they have fully started. Default: false
  }
```

`init-required`设置为`true`的任何actor需要发送一个`Initialized(report)`消息给cube监管者, 它们是这些well-known actor的父actor）。一旦所有的cube初始化成功，squbs运行时将变更到*Active*状态。这也意味着每个`init-required`设为`true`的actor提交了带`success`的`Initialized(report)`消息。如果任何一个cube通过`Initialized(report)`报告了初始化错误，squbs运行时将以*Failed*状态终结。

##### Scala

actor参与初始化发送一个`Initialized(report)`消息。`report`将是`Try[Option[String]]`类型，允许actor报告初始化成功和特有异常的失败。

##### Java

创建一个`Initialized(report)`的Java API如下所示:

```java
// Creates a successful InitReport without a description.
Initialized.success();

// Creates a successful InitReport with a description.
Initialized.success(String desc);

// Creates a failed InitReport given a Throwable as a reason.
Initialized.failed(Throwable e);
```

## 关闭(Shutdown)钩子

### 停止Actor

Scala特质`org.squbs.lifecycle.GracefulStopHelper`和Java抽象类`org.squbs.lifecycle.ActorWithGracefulStopHelper`让用户在他们actor的代码中获得优雅停止。你可以使用这些特质或抽象类，用下面的方法。

##### Scala

```scala
class MyActor extends Actor with GracefulStopHelper {
    ...
}
```

##### Java

```java
public class MyActor exteds ActorWithGracefulStopHelper {
    ...
}
```

这些特质/抽象类提供了一下辅助方法来支持squbs框架中的actor的优雅停止。

#### 停止超时

为了防止关闭过程被卡住，停止超时是指在强制停止actor之前，控制关闭过程所花费的最大时间。`stopTimeout`属性可以按如下方式重写:

##### Scala

```scala
/**
 * Duration that the actor needs to finish the graceful stop.
 * Override it for customized timeout and it will be registered to the reaper
 * Default to 5 seconds
 * @return Duration
 */
def stopTimeout: FiniteDuration =
  FiniteDuration(config.getMilliseconds("default-stop-timeout"), TimeUnit.MILLISECONDS)
```

##### Java

```java
@Override
public long getStopTimeout() {
    return config.getMilliseconds("default-stop-timeout");
}
```

你可以重写该方法，以指示此actor执行优雅停止大概需要多长时间。一旦actor启动, 它将用`StopTimeout(stopTimeout)`消息发送`stopTimeout`给父actor。如果您关心此消息，则可以在父actor有一个行为，用来处理此消息。

如果你在actor的Scala代码中混入了这个特质，你应当在`receive`方法中有一个行为处理`GracefulStop`消息，因为只有这种情况下你可以hook你的代码以执行一个优雅的停止(你不能添加趋向`PoisonPill`自定义行为)。监管者只传播`GracefulStop`消息给混入了`GracefulStopHelper`特质的子actor。子actor的实现应当在它们的`receive`代码块中处理这个消息。

同样的，Java类扩展`ActorWithGracefulStopHelper`期望在actor的`createReceive()`方法中处理`GracefulStop`。

squbs还在特质/抽象类中提供了以下2个默认策略。

##### Scala

```scala
  /**
   * Default gracefully stop behavior for leaf level actors
   * (Actors only receive the msg as input and send out a result)
   * towards the `GracefulStop` message
   *
   * Simply stop itself
   */
  protected final def defaultLeafActorStop: Unit
```

```scala
  /**
   * Default gracefully stop behavior for middle level actors
   * (Actors rely on the results of other actors to finish their tasks)
   * towards the `GracefulStop` message
   *
   * Simply propagate the `GracefulStop` message to all actors
   * that should be stop ahead of this actor
   *
   * If some actors failed to respond to the `GracefulStop` message,
   * It will send `PoisonPill` again
   *
   * After all the actors get terminated it stops itself
   */
  protected final def defaultMidActorStop(dependencies: Iterable[ActorRef],
                                          timeout: FiniteDuration = stopTimeout / 2): Unit
```

##### Java

```java
/**
 * Default gracefully stop behavior for leaf level actors
 * (Actors only receive the msg as input and send out a result)
 * towards the `GracefulStop` message
 *
 * Simply stop itself
 */
protected final void defaultLeafActorStop();
```

```java
/**
 * Java API stopping non-leaf actors.
 * @param dependencies All non-leaf actors to be stopped.
 * @param timeout The timeout for stopping the actors.
 */
protected final void defaultMidActorStop(List[ActorRef] dependencies, long timeout, TimeUnit unit);
```

### 停止squbs的扩展

你可以在扩展关闭里增加自定义行为，通过覆盖`org.squbs.lifecycle.ExtensionLifecycle`中的`shutdown()` 方法。请注意，所有已安装的扩展中的此方法都将在actor系统终止后执行。如果任何扩展在关闭期间抛出异常，JVM将以-1退出。
