# 超时策略

超时是每个异步系统和每个分布式系统的关键部分。它们通常是静态配置的，很难正确配置。squbs提供超时策略工具，作为squbs模式包的一部分，以帮助动态地确定给定策略的正确超时，而不是静态设置。然后可以使用策略中动态确定的超时值设置超时。请注意，超时策略仅帮助派生超时值，其使用本身不会导致超时。

## 依赖

将以下依赖项添加到您的`build.sbt`或scala构建文件:

```scala
"org.squbs" %% "squbs-pattern" % squbsVersion
```

## 简单的例子

在正常等待情况下，在下面的Scala代码中所示...

```scala
Await.ready(future, timeout.duration)
```

你可以改变这段代码，`await`块包含在一个闭包内，如下:

```scala
val policy = TimeoutPolicy(name = Some("mypolicy"), initial = 1 second)
val result = policy.execute(duration => {
  Await.ready(future, duration)
})
```

## Scala API

你可以创建`TimeoutPolicy`，只通过传递一个规则给它，如下：

1. 对于固定超时，实际上不需要指定规则。fixedRule是默认设置。

   ```scala
   val policy = TimeoutPolicy(name = Some("MyFixedPolicy"), initial = 1 second, rule = fixedRule)
   ```

2. 基于响应时间标准差的超时。

   ```scala
   val policy = TimeoutPolicy(name = Some("MySigmaPolicy"), initial = 1 second, rule = 3 sigma)
   ```

3. 基于响应时间百分比的超时。

   ```scala
   val policy = TimeoutPolicy(name = Some("MyPctPolicy"), initial = 1 second, rule = 95 percentile)
   ```

然后，您可以使用以下策略：

```scala
val result = policy.execute(duration => {
  Await.ready(future, duration)
})
```

或者另一种不使用闭包的形式。

```scala
val policy = TimeoutPolicy(name = Some("mypolicy"), initial = 1 second)
val tx = policy.transaction
Await.ready(future, tx.waitTime)
tx.end
```

`tx.end`很重要，因为它为超时策略监视的单个操作提供了闭包，当使用API的首选闭包版本时，会自动检测到该闭包。这是一个反馈循环，用于观察实际执行时间，并将此信息反馈给启发式(heuristics)。

## Java API

在Java API中，我们使用策略生成器创建超时策略，如下面的示例所示...

1. 对于固定超时。

   ```java
   TimeoutPolicy fixedTimeoutPolicy = TimeoutPolicyBuilder
       .create(Duration.ofMillis(INITIAL_TIMEOUT), fromExecutorService(es))
       .minSamples(1)
       .rule(TimeoutPolicyType.FIXED)
       .build();
   ```

2. 基于响应时间标准差的超时。
 
   ```java
   TimeoutPolicy sigmaTimeoutPolicy = TimeoutPolicyBuilder
       .create(Duration.ofMillis(INITIAL_TIMEOUT), system.dispatcher())
       .minSamples(1)
       .name("MySigmaPolicy")
       .rule(3.0, TimeoutPolicyType.SIGMA)
       .build();
   ```

3. 基于响应时间百分比的超时。

   ```java
   TimeoutPolicy percentileTimeoutPolicy = TimeoutPolicyBuilder
       .create(Duration.ofMillis(INITIAL_TIMEOUT),, system.dispatcher())
       .minSamples(1)
       .name("PERCENTILE")
       .rule(95, TimeoutPolicyType.PERCENTILE)
       .build();
   ```

然后，要使用超时策略，只需执行您的定时调用内包如下:

```java
policy.execute((Duration t) -> {
    return es.submit(timedCall).get(t.toMillis() + 20, MILLISECONDS);
});
```

或者，你可以使用非闭包版本的调用如下:

```java
TimeoutPolicy.TimeoutTransaction tx = policy.transaction();
try {
  return timedFn.get(Duration.ofMillis(tx.waitTime().toMillis()));
} catch (Exception e) {
  System.out.println(e);
} finally {
  tx.end();
}
```

`tx.end`很重要，因为它为超时策略监视的单个操作提供了闭包，当使用API的首选闭包版本时，会自动检测到该闭包。这是一个反馈循环，用于观察实际执行时间，并将此信息反馈给启发式(heuristics)。

## 超时策略中的启发式

默认的超时策略是固定超时，这意味着您总是在常规模式下获得初始值，在调试模式下获得调试值。但是，拥有超时策略的基本前提是提供基于启发式的超时。以下是超时策略中使用的主要概念。

在统计学上，[**68–95–99.7  rule**](http://en.wikipedia.org/wiki/68%E2%80%9395%E2%80%9399.7_rule)，也称为**三标准差规则**或**经验法则**，指出几乎所有的数值都在均值的三个标准差内，呈正态分布。
![empirical rule](http://upload.wikimedia.org/wikipedia/commons/a/a9/Empirical_Rule.PNG)

因此，如果您像下面这样声明您的超时策略:

```scala
val policy = TimeoutPolicy("mypolicy", initial = 1 second, rule = 2 sigma)
```

您将得到一个超时值，它将覆盖大约95%的响应时间，并通过使用2西格玛或95%的超时策略来切断5%的异常。这是通过调用`policy.execute(tiemout=>T)`或`policy.transaction.waitTime`来应用的。

### 重置统计

有三种方法可以重置/启动统计数据

1. 在构造`TimeoutPolicy`时设置`startOverCount`，当总事务数超过`startOverCount`时，将自动开始统计
2. 调用`policy.reset`来重置统计，你也可以在调用重置方法时给出新的`initial`和`startOverCount`。
3. 调用`TimeoutPolicy.resetPolicy("yourName")`在全局级别重置策略。

## 名称

超时策略的任何构造都采用可选名称。超时策略使用相同的名称与其他策略实例共享它们的指标。policy-by-name可以避免用户创建策略实例并将其传递给共享相同策略的所有用法。用户可以在任何使用点干净地复制策略创建，同时仍然收集指标。此外，在策略中使用名称可以通过调用`TimeoutPolicy.resetPolicy("name")`来集中清除指标。

**警告**: 不要对性质完全不同的策略使用相同的名称，因为这会打乱您的统计信息。结果可能无法预测。

## 调试

出于调试目的，在调试模式下执行超时策略时，超时策略中的默认超时时间为1,000秒。您可以通过向TimeoutPolicy传递一个`debug`参数来设置它，如下所示：

```scala
val policy = TimeoutPolicy(name = Some("mypolicy"), initial = 1 second, debug = 10000 seconds)
```
