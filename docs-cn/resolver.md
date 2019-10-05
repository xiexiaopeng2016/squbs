# 资源解析

考虑到很少 - 如果有的话 - 实际应用程序可以在没有外部资源的情况下工作，环境感知的资源解析正成为应用程序基础结构的一个重要部分。squbs通过`ResolverRegistry`提供资源解析，并允许任何类型的资源通过名称和环境来解决。后者允许在生产、qa 和开发环境之间区分资源。

资源解析的示例是HTTP端点、消息端点和数据库。所有这些都由一个单一的注册处理。

### 依赖

解析器位于`squbs-ext`。将以下依赖项添加到您的`build.sbt`或scala构建文件:

```scala
"org.squbs" %% "squbs-ext" % squbsVersion
```

### 用法

`Resolver`基本用法是查找资源。需要提供类型，因为注册中心可以保存多种类型的资源，如HTTP端点、消息传递端点或数据库连接。本文档中的示例中使用了类型`URI`。查找调用，如下所示：

##### Scala

```scala
// To resolve a resource for a specific environment.
val resource: Option[URI] = ResolverRegistry(system).resolve[URI]("myservice", QA)

```

##### Java

```java
// To resolve a resource for a specific environment.
val resource: Optional<URI> = ResolverRegistry.get(system).resolve(URI.class, "myservice", QA.value());
```

### ResolverRegistry

`ResolverRegistry`是一个Akka扩展，遵循Akka扩展使用模式，在Scala和Java中。它可以托管各种类型的资源解析器，因此必须在注册时提供资源类型，通过将资源类型传递给`register`调用。可以注册多个同一类型和多个类型的解析器。

#### 注册解析器

有两种风格的API用于解析器的注册。一个是快捷API，允许传入闭包或lambda作为解析器。闭包或者lambda的返回类型必须是Scala中的`Option[T]`和Java中的`Optional<T>`。另一个完整的API使用Scala中的`Resolver[T]`或Java中的`AbstractResolver<T>`，`T`是资源类型。如下所示：

##### Scala

```scala
// To register a new resolver for type URI using a closure. Note the return
// type of the closure must be `Option[T]` or in this case `Option[URI]`
ResolverRegistry(system).register[URI]("MyResolver") { (svc, env) =>
  (svc, env) match {
    case ("myservice", QA) => Some(URI.create("http://myservice.qa.mydomain.com"))
    case ("myservice", Default) => Some(URI.create("http://myservice.mydomain.com"))
    case ("myservice2", QA) => Some(URI.create("http://myservice2.qa.mydomain.com"))
    case ("myservice2", Default) => Some(URI.create("http://myservice2.mydomain.com"))
    case _ => None
  }
}

// To register a new resolver for type URI by extending the `Resolver` trait
class MyResolver extends Resolver[URI] {
  def name: String = "MyResolver"
  
  def resolve(svc: String, env: Environment = Default): Option[URI] = {
    (svc, env) match {
      case ("myservice", QA) => Some(URI.create("http://myservice.qa.mydomain.com"))
      case ("myservice", Default) => Some(URI.create("http://myservice.mydomain.com"))
      case ("myservice2", QA) => Some(URI.create("http://myservice2.qa.mydomain.com"))
      case ("myservice2", Default) => Some(URI.create("http://myservice2.mydomain.com"))
      case _ => None
    }
  }
}

// Then just register the instance
ResolverRegistry(system).register[URI](new MyResolver)
```

##### Java 

```java
// To register a new resolver for type URI using a lambda. Note the return
// type of the lambda must be `Optional<T>` or in this case `Optional<URI>`
ResolverRegistry.get(system).register("MyResolver", (svc, env) -> {
    if ("myservice".equals(svc)) {
        if (QA.value().equals(env)) {
          return Optional.of(URI.create("http://myservice.qa.mydomain.com"));
        } else {
          return Optional.of(URI.create("http://myservice.mydomain.com"));
        }
    } else if ("myservice2".equals(svc)) {
        if (QA.value().equals(env)) {
          return Optional.of(URI.create("http://myservice2.qa.mydomain.com"));
        } else {
          return Optional.of(URI.create("http://myservice2.mydomain.com"));
        }    
    } else {
        return Optional.empty();
    }
});

// To register a new resolver for type URI by extending an abstract class
public class MyResolver extends AbstractResolver<URI> {
    @Override
    public String name() {
        return "MyResolver";
    }
    
    @Override
    public Optional<URI> resolve(String svc, Environment env) {
        if ("myservice".equals(svc)) {
            if (QA.value().equals(env)) {
                return Optional.of(URI.create("http://myservice.qa.mydomain.com"));
            } else {
                return Optional.of(URI.create("http://myservice.mydomain.com"));
            }
        } else if ("myservice2".equals(svc)) {
            if (QA.value().equals(env)) {
                return Optional.of(URI.create("http://myservice2.qa.mydomain.com"));
            } else {
                return Optional.of(URI.create("http://myservice2.mydomain.com"));
            }    
        } else {
            return Optional.empty();
        }
    }
}

// Then register MyResolver.
ResolverRegistry.get(system).register(URI.class, new MyResolver());
```

#### 发现链

资源发现遵循后进先出模型。最近注册的解析器优先于以前注册的解析器。`ResolverRegistry`沿着链一个一个地遍历，直到有一个解析器与资源所给的类型兼容或者链已搜索完毕。这种情况下，注册器将返回`None`(Scala API)和`Optional.empty()` (Java API)。

#### 类型兼容

`ResolverRegistry`在`resolve`调用时检查请求的类型。如果注册的解析器的类型是请求类型的同一类型或子类型，则该解析器将尝试按名称解析资源。

由于JVM类型擦除，注册的类型的类型参数或者请求的类型都没有统计。例如，一个注册类型`java.util.List<String>`会与类型`java.util.List<Int>`的`resolve`调用相匹配，因为类型参数`String`或`Int`运行时擦除了。由于这个限制，非常不推荐含有类型参数的类型用于注册和查找。结果是未定义 - 您可能会得到错误的资源。

为了简单起见，强烈建议不要使用类型层次结构。 所有注册的类型应该是不同的类型。

#### 资源解析

与注册类似，解析要求类型与注册类型兼容；已注册的类型必须是解析类型的相同或子类型。

##### Scala

```scala
// To resolve a resource with `Default` environment.
val resource: Option[URI] = ResolverRegistry(system).resolve[URI]("myservice")

// To resolve a resource for a specific environment.
val resource: Option[URI] = ResolverRegistry(system).resolve[URI]("myservice", QA)
```

##### Java

```java
val resource: Optional<URI> = ResolverRegistry.get(system).resolve(URI.class, "myservice", QA.value());
```

#### Un-registering a Resolver

取消注册是通过名称使用以下API来完成的。

##### Scala

```scala
ResolverRegistry(system).unregister("MyResolver")
```

##### Java

```java
ResolverRegistry.get(system).unregister("MyResolver");
```

#### 并发性的考虑

解析器注册和注销调用应当在初始化时以非并发的方式完成。没有对并发注册的保护，因此并发注册的结果是未定义的。在并发下注册或注销，你的解析器可能是注册的，也可能不是。

但是，解析调用是线程安全的，可以并发访问，而不受`ResolverRegistry`级别的限制。每个单独注册的解析器都需要是线程安全的。
