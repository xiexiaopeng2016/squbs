# 在运行时监控Actor

## 概述

`squbs-actormonitor`模块将监视附加到actor system中的每个actor。对于大量的actor，这可能会造成干扰。可以通过`application.conf`配置要监视的actor的数量。在生产中使用附加此模块的判断。此模块没有用户API。

## 依赖

在`build.sbt`或Scala构建文件添加如下依赖：

```
"org.squbs" %% "squbs-actormonitor" % squbsVersion
```

## 监控

每一个actor都有一个相应的`JMXBean(org.squbs.unicomplex:type=ActorMonitor,name=%actorPath)`来暴露actor的信息：

```
 trait ActorMonitorMXBean {
  def getActor: String
  def getClassName: String
  def getRouteConfig : String
  def getParent: String
  def getChildren: String
  def getDispatcher : String
  def getMailBoxSize : String
}
```

## 配置

下面是`squbs-actormonitor`的配置项：

```
squbs-actormonitor = {
  maxActorCount = 500
  maxChildrenDisplay = 20
}
```

一个JMX Bean `org.squbs.unicomplex:type=ActorMonitor` 暴露了Actor监控的配置。JMX Bean是只读的。

```
trait ActorMonitorConfigMXBean {
  def getCount : Int				//已经创建了的JMX bean的数目
  def getMaxCount: Int				//可以创建JMX bean的最大的数目
  def getMaxChildrenDisplay: Int		//每一个actor，可以暴露的子actor的最大数目
}
```
