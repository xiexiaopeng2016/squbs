# 物化指标收集器阶段

### 概述

`MaterializationMetricsCollector`是一个Akka流`GraphStage`，用于收集流的物化指标：

   * 活动的物化数量
   * 新的物化创建率
   * 物化终止率(成功终止和失败的汇总)

`MaterializationMetricsCollector`的一个突出用例是服务器端Akka HTTP，其中每个新连接都会导致流物化，因此每个连接终止都会导致相关流物化的终止。因此，`MaterializationMetricsCollector`是与[squbs HTTP(S) service](HTTP -services.md)开箱即用的集成，实现它发布活动连接、连接创建和连接终止指标。

### 依赖

将以下依赖项添加到您的`build.sbt`或scala构建文件:

```
"org.squbs" %% "squbs-ext" % squbsVersion
```

### 用法

其用法与标准的Akka流阶段非常相似。在下面的例子中，您会看到JMX bean的名称包含：

   * `my-stream-active-count` 开始时`Count`的值是2，一旦物化终止就会下降到0
   * `my-stream-creation-count` 包含`Count`值为2。
   * `my-stream-termination-count` 仅在流终止后才会显示，并且`Count`最终值为2。


##### Scala

```scala
val stream = Source(1 to 10)
  .via(MaterializationMetricsCollector[Int]("my-stream"))
  .to(Sink.ignore)

stream.run()
stream.run() // Materializing the stream a second time, thus increasing the Count value to 2
```      

##### Java

```java
RunnableGraph<NotUsed> stream =
        Source.range(1, 10)
                .via(MaterializationMetricsCollector.create("my-stream", system))
                .to(Sink.ignore());

stream.run(mat);
stream.run(mat); // Materializing the stream a second time, thus increasing the Count value to 2
```   
