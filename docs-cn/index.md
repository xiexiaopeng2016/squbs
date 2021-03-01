![image](img/squbs-logo-transparent.png)

## 介绍

squbs(发音"skewbs")是一套组件，支持在大型、托管的云环境中对Akka和Akka HTTP应用程序/服务进行标准化和操作化。它标准化了Akka应用程序如何部署在不同的环境中，以及它们如何连接到大型internet规模组织的操作环境。

## squbs组件

1. **Unicomplex**: 微型容器，引导并标准化Akka应用程序的部署以及它们的配置方式，允许PD以外的团队了解配置并根据需要调整应用程序的配置(部分在运行时)。此外，Unicomplex鼓励以一种灵活、松耦合的方式共存不同的模块，称为cube和/或操作工具，这种方式不会导致任何代码更改，从而包括新的操作工具或删除/更改某些操作工具。例如，在我们有混合云环境的情况下，例如私有云和公共云需要不同的操作工具，相同的代码库将同时工作，允许在部署时添加特定于环境的工具。

2. **TestKit**: 用于帮助测试为squbs编写的应用程序，甚至整个Akka应用程序。它提供了单元测试和小型负载测试设施，可以作为CI的一部分运行。

3. **ZKCluster**: 一种基于zookeeper、支持数据中心的集群库，允许集群应用程序或服务跨数据中心，并保留跨数据中心的可用性特征。这对于需要集群内部通信的应用程序是必需的。

4. **HttpClient**: 一个可操作的、简化的客户机，支持环境和端点解析，以适应不同的操作环境(QA、Prod)以及组织需求(Topo、direct)。

5. **Pattern**: 提供给用户的一组编程模式和dsl。
   1. 编排DSL允许开发人员以极其简洁的方式描述他们的编排顺序，同时异步运行整个编排，从而极大地简化了代码并减少了应用程序的延迟。
   2. 异步系统在很大程度上依赖于超时，固定的超时从来都不是正确的。TimeoutPolicy允许用户设置策略(如2.5 sigma)，而不是固定的超时值，并自行处理启发式，允许系统适应其运行条件。
   3. Validation提供了一个[Akka HTTP指令](http://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/directives/index.html)，通过使用[Accord Validation Library](http://wix.github.io/accord/)进行数据验证。
   4. PersistentBuffer提供了一个高性能的Akka Streams流缓冲区组件，它将其内容持久化到内存映射文件中，并在失败和重启后恢复内容。

6. **ActorRegistry**: 核心查找功能允许松耦合模块的参与者彼此查找，甚至将不同的服务建模为参与者。

7. **ActorMonitor**: 一个使用JMX报告系统中参与者的统计和行为的附加操作模块。任何JMX工具都可以看到这些统计信息

8. **Pipeline**: 允许对请求/响应过滤器进行排序和插入的基础设施。例如，它们用于安全性、速率限制、日志记录等。每个组件实际上彼此之间没有依赖关系。它们是真正松散耦合的。开发人员和组织可以自由选择环境所需的组件。

9. **Console**: 一个drop-in模块允许web访问系统和应用程序的统计信息，通过一个简单的web和服务接口返回精美的JSON。

## Getting Started

最简单的入门方法是从一个squbs模板创建一个项目。下面是目前可用的giter8模板:

* [squbs-scala-seed](https://github.com/paypal/squbs-scala-seed.g8): 创建squbs scala应用示例的模板
* [squbs-java-seed](https://github.com/paypal/squbs-java-seed.g8): 创建示例squbs java应用程序的模板

## Documentation

* [squbs的设计原则](principles_of_the_squbs_design.md)
* [Unicomplex & Cube 引导程序](bootstrap.md)
* [Unicomplex Actor 层次结构](actor-hierarchy.md)
* [运行时生命周期和API](lifecycle.md)
* [实现HTTP (S)服务](http-services.md)
* [Akka HTTP 客户端 on Steroids](httpclient.md)
* [请求/响应管道](pipeline.md)
* [编组和解组](marshalling.md)
* [配置](configuration.md)
* [测试squbs应用程序](testing.md)
* [使用ZooKeeper集群squbs服务](zkcluster.md)
* [阻塞调度器](blocking-dispatcher.md)
* [消息指南](messages.md)
* [Actor Monitor](monitor.md)
* [Orchestration DSL](orchestration_dsl.md)
* [Actor Registry](registry.md)
* [Admin Console](console.md)
* [Application Lifecycle Management](packaging.md)
* [Resource Resolution](resolver.md)
* Akka Streams `GraphStage`s:
    * [Persistent Buffer](persistent-buffer.md)
    * [Perpetual Stream](streams-lifecycle.md)
    * [Circuit Breaker](circuitbreaker.md)
    * [Timeout](flow-timeout.md)
    * [Deduplicate](deduplicate.md)
* [Timeout Policy](timeoutpolicy.md)
* [Validation](validation.md)

## Release Notes

Please find release notes at [https://github.com/paypal/squbs/releases](https://github.com/paypal/squbs/releases).

## Support & Discussions

Please join the discussion at  [![Join the chat at https://gitter.im/paypal/squbs](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/paypal/squbs?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)