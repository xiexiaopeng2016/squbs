
# 用于阻塞式API调用的阻塞式调度器

本主题不是关于一般的调度器，而是关于squbs特定的调度器配置。请查看[Akka文档](http://doc.akka.io/docs/akka/2.3.13/scala/dispatchers.html)中关于调度器的详细信息。

squbs增加了另外一个预先配置的调度器用于阻塞式调用。通常，它们用于对数据库的同步调用。`reference.conf`定义了`blocking-dispatcher`，如下所示：

```
blocking-dispatcher {
  # Dispatcher is the name of the event-based dispatcher
  type = Dispatcher
  # What kind of ExecutionService to use
  executor = "thread-pool-executor"
  thread-pool-executor {
    # Min number of threads to cap factor-based core number to
    core-pool-size-min = 2
    # The core pool size factor is used to determine thread pool core size
    # using the following formula: ceil(available processors * factor).
    # Resulting size is then bounded by the core-pool-size-min and
    # core-pool-size-max values.
    core-pool-size-factor = 3.0
    # Max number of threads to cap factor-based number to
    core-pool-size-max = 24
    # Minimum number of threads to cap factor-based max number to
    # (if using a bounded task queue)
    max-pool-size-min = 2
    # Max no of threads (if using a bounded task queue) is determined by
    # calculating: ceil(available processors * factor)
    max-pool-size-factor  = 3.0
    # Max number of threads to cap factor-based max number to
    # (if using a  bounded task queue)
    max-pool-size-max = 24
  }
  # Throughput defines the maximum number of messages to be
  # processed per actor before the thread jumps to the next actor.
  # Set to 1 for as fair as possible.
  throughput = 2
}
```

要让actor使用阻塞调度器，只需像下面的示例那样指定actor配置：

```
  "/mycube/myactor" {
    dispatcher = blocking-dispatcher
  }
```

如果没有任何actor使用`blocking-dispatcher`，`blocking-dispatcher`将不会被初始化，也不需要任何资源。

**警告:** 阻塞式调度器只能应用于阻塞式调用，否则性能可能受到严重影响。
