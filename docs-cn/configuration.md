# 配置参考

下面列出了在`reference.conf`中定义的squbs配置：

```
squbs {

  # Name of the actor system for squbs to create.
  actorsystem-name = "squbs"

  # graceful stop timeout
  # default timeout for one actor to process a graceful stop
  # if extends the trait org.squbs.lifecycle.GracefulStopHelper
  default-stop-timeout = 3s

  # An external configuration directory to supply external application.conf. The location of this directory
  # is relative to the working directory of the squbs process.
  external-config-dir = squbsconfig

  # An external configuration file name list. Any file with the name in the list under the external-confi-dir will be
  # loaded during squbs initialization for Actor System settings. Implicit "application.conf" will be loaded
  # besides this file name list
  external-config-files = []

  # Service infra configuration.
  service-infra {
    # Maximum amount of time to wait for all listeners to be started.
    timeout = 60s
    # Maximum amount of time each listener is given to start.
    listener-timeout = 10s
  }
}

default-listener {

  # All squbs listeners carry the type "squbs.listener"
  type = squbs.listener

  # Add aliases for the listener in case the cube's route declaration binds to a listener with a different name.
  # Just comma separated names are good, like...
  # aliases = [ foo-listener, bar-listener ]
  aliases = []

  # Service bind to particular address/interface. The default is 0.0.0.0 which is any address/interface.
  bind-address = "0.0.0.0"

  # Whether or not using full host name for address binding
  full-address = false

  # Service bind to particular port. 8080 is the default.
  bind-port = 8080

  # Listener uses HTTPS?
  secure = false

  # HTTPS needs client authorization? This configuration is not read if secure is false.
  need-client-auth = false

  # Any custom SSLContext provider? Setting to "default" means platform default.
  ssl-context = default

  # Which materializer to use for HTTP streams.  default-materializer is used if not specified
  # materializer = default-materializer
}

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

default-materializer {
  # All squbs materializers carry the type "squbs.materializer"
  type = squbs.materializer

  # The class with createMaterializer function to create a materializer
  class = org.squbs.unicomplex.DefaultMaterializer
}
```

## 阻塞调度器

squbs的`reference.conf`定义了一个`blocking-dispatcher`用于阻塞I/O调用。这是一个标准的Akka调度器配置。请查看Akka文档中的[dispatchers](http://doc.akka.io/docs/akka/2.3.13/scala/dispatchers.html)了解更多细节。

## 监听器

监听器定义端口绑定和此端口绑定的行为，比如安全性、身份验证等。squbs `reference.conf`提供了一个默认的监听器由。这可以由应用程序提供`application.conf`文件或在其外部配置目录中提供`application.conf`文件来重写。请查看[引导squbs](bootstrap.md#configuration-resolution)章节，了解squbs如何读取配置文件的细节。

一个监听器在配置文件根级别声明。通常名称遵循`*-listener`模式，但这不是必须的。将条目定义为监听器的是`type`字段。它必须设置为`squbs.listener`。请参阅前面的`default-listener`示例，了解如何配置新监听器监听不同的端口。

一个已声明的监听器不会启动，除非服务路由将自身附加到此监听器。换言之，只声明监听器不会自动导致监听器启动，除非监听器真正使用。

## 物化器

一个squbs物化器仅仅是一个Akka Stream `Materializer`，其在配置中指定。这允许squbs保留所有物化器的注册表，以便：

   * 通过Akka扩展从不同位置访问`Materializer`，如下所示：

     **Scala**
   
     ```scala
     implicit val mat = Unicomplex(system).materializer("default-materializer")
     ```
   
     **Java**
   
     ```java
     final Materializer mat = Unicomplex.get(system).materializer("default-materializer")
     ```

   * 物化器可以从[squbs监听器](#listeners)引用。
   * 通过相应的设置，应用程序在用的物化器，可在JMX被报告。

在squbs的`reference.conf`中提供了一个默认的物化器。物化器的创建是惰性的。只有使用的才会真正创建。

物化器在配置文件根级别声明。通常名称遵循`*-materializer`模式，但这不是必须的。将条目定义为物化器是物化器条目下的`type`字段。它必须设置为`squbs.materializer`。请参阅前面的`default-materializer`示例，了解如何配置新物化器。

## 管道

如果已定义，为`pre-processing`的每一个请求和`post-processing`的每一个响应安装了默认管道。服务可以指定一个不同的管道，或者根本不指定，就像在[引导squbs](bootstrap.md#services)中描述的那样。应用或基础设施可以为`pre-processing`需求(例如日志和追踪)实现他们自己的管道。请查看[请求/响应管道](pipeline.md)章节了解详细描述。
