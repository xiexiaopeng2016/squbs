# 测试squbs应用

squbs支持Scala的ScalaTest 3.X，TestNG和Java测试框架的JUnit

## 依赖

要使用文档中提到的测试工具，只需简单的添加如下依赖在你的`build.sbt`文件或者Scala构建脚本：

```scala
"org.squbs" %% "squbs-testkit" % squbsVersion
```

另外，您还应该根据是否需要在测试中包含以下依赖项：

```scala
// Testing RouteDefinition...
"com.typesafe.akka" %% "akka-http-testkit" % akkaHttpVersion % "test",

// Using JUnit...
"junit" % "junit" % junitV % "test",
"com.novocode" % "junit-interface" % junitInterfaceV % "test->default",

// Using TestNG
"org.testng" % "testng" % testngV % "test",
```

## CustomTestKit

`CustomTestKit`用于启动一个完整的squbs实例，用于测试应用的各个部分。`CustomTestKit`使用简单，默认情况下不需要配置，但允许自定义和灵活的测试。通过`CustomTestKit`，你可以使用不同配置启动任意数量的`ActorSystem`和`Unicomplex`实例(每个一个`ActorSystem`) - 都在相同的JVM。一些特性如下：

   * 它自动将actor system名称简单的设置为spec/test类名称，以确保`ActorSystem`实例不发生冲突。但是，它还允许在构造函数中传递actor system名称。
   * 继承`CustomTestKit`的测试可以在同一JVM中并行运行。
   * 自动启动和关闭squbs。
   * 默认启动`src/main/resources/META-INF/squbs-meta.conf`和`src/test/resources/META-INF/squbs-meta.conf`中的well-known actor和服务。但是，允许在构造函数中传递被扫描的`resources`。
   * 允许在构造器中传递自定义配置。

以下是在ScalaTest中应用`CustomTestKit`的实例：

```scala
import org.squbs.testkit.CustomTestKit

class SampleSpec extends CustomTestKit with FlatSpecLike with Matchers {
   it should "start the system" in {
      system.startTime should be > 0L
   }
}
``` 

TestNG和JUnit用于支持Java用户：

```java
import org.squbs.testkit.japi.CustomTestKit

public class SampleTest extends CustomTestKit {

    @Test
    public void testSystemStartTime() {
        Assert.assertTrue(system().startTime() > 0L);
    }
}
```

### 传递配置给`CustomTestKit`

如果你想要自定义actor system配置，你可以传递一个`Config`对象给`CustomTestKit`：

```scala
object SampleSpec {
  val config = ConfigFactory.parseString {
      """
        |akka {
        |  loglevel = "DEBUG"
        |}
      """.stripMargin
  }
}

class SampleSpec extends CustomTestKit(SampleSpec.config) with FlatSpecLike with Matchers {

  it should "set akka log level to the value defined in config" in {
    system.settings.config.getString("akka.loglevel") shouldEqual "DEBUG"
  }
}
```

以下是测试的TestNG/JUnit版本:

```java
import org.squbs.testkit.japi.CustomTestKit;

public class SampleTest extends CustomTestKit {

    public SampleTest() {
        super(TestConfig.config);
    }

    @Test
    public void testAkkaLogLevel() {
        Assert.assertEquals(system().settings().config().getString("akka.loglevel"), "DEBUG");
    }

    private static class TestConfig {
        private static Config config = ConfigFactory.parseString("akka.loglevel = DEBUG");
    }
}
```

以下部分显示的定制仅限于ScalaTest；但是，TestNG和JUnit也支持所有这些定制。对于定制，在TestNG/JUnit测试中提供一个`public`构造函数，并使用自定义参数调用`super`。检查[squbs-testkit/src/test/java/org/squbs/testkit/japi](https://github.com/paypal/squbs/tree/master/squbs-testkit/src/test/java/org/squbs/testkit/japi)了解更多TestNG和JUnit的实例.

特别是对于JUnit，避免在您的测试中设置actor system名称(尽管让`CustomTestKit`设置actor system名通常是一个好的实践)。JUnit为每个`@Test`方法创建一个新的fixture实例，这可能会导致actor system冲突。`AbstractCustomTestKit`通过向每个actor system名附加一个递增的整数来避免这种情况。

### 使用`CustomTestKit`启动well-known actor和服务

`CustomTestKit`自动启动`src/test/resources/META-INF/squbs-meta.conf`中配置的well-known actor和服务(查看[引导squbs](bootstrap.md#well-known-actors))。然而，如果你想要提供不同的资源路径，你可以传递一个资源的`Seq` *(Scala)* 或 `List` *(Java)* 给构造器。`withClassPath`控制是否扫描整个测试类路径以获得`META-INF/squbs-meta.conf`文件。

```scala
object SampleSpec {
	val resources = Seq(getClass.getClassLoader.getResource("").getPath + "/SampleSpec/META-INF/squbs-meta.conf")
}

class SampleSpec extends CustomTestKit(SampleSpec.resources, withClassPath = false)
  with FlatSpecLike with Matchers {
	
  // Write tests here	
}
```

请注意，`CustomTestKit`允许一起传递`config`和`resources`。

#### 测试中的端口绑定

启动服务需要端口绑定。为了防止端口冲突，应当让系统挑选一个端口，通过将监听器的`bind-port`设置0，例如，`default-listener.bind-port = 0`(`CustomTestKit`如果使用默认配置，对所有监听器设置`bind-port = 0`)。`squbs-testkit`自带名为`PortGetter`的`trait`，允许检索系统选择的端口。`CustomTestKit`自带的`PortGetter`已经混入，所以你可以使用`port`值。

```scala
class SampleSpec extends CustomTestKit(SampleSpec.resources)
  
  "PortGetter" should "retrieve the port" in {
    port should be > 0
  }
}

```

默认，`PortGetter`返回`default-listener`的端口，这是最常见的。如果你需要检索其它监听器绑定的端口，可以覆盖`listener`方法或者传递监听器名称给`port`方法：

```scala
class SampleSpec extends CustomTestKit(SampleSpec.resources)

  override def listener = "my-listener"

  "PortGetter" should "return the specified listener's port" in {
    port should not equal port("default-listener")
    port shouldEqual port("my-listener")
  }
}
```

### 手工`UnicomplexBoot`初始化

`CustomTestKit`也允许传入一个`UnicomplexBoot`实例。这允许完全定制引导。请看[引导squbs](bootstrap.md)章节查看详细信息。

```scala
object SampleSpec {
  val config = ConfigFactory.parseString(
    s"""
       |squbs {
       |  actorsystem-name = SampleSpec # should be unique to prevent collision with other tests running in parallel
       |  ${JMX.prefixConfig} = true # to prevent JMX name collision, if you are doing any JMX testing
       |}
    """.stripMargin
  )

  val resource = getClass.getClassLoader.getResource("").getPath + "/SampleSpec/META-INF/squbs-meta.conf"	
	
  val boot = UnicomplexBoot(config)
    .createUsing {(name, config) => ActorSystem(name, config)}
    .scanResources(resource)
    .initExtensions.start()
}

class SampleSpec extends CustomTestKit(SampleSpec.boot) with FunSpecLike with Matchers {

  // Write your tests here.
}
```

### 关闭

对于具有多个并行运行的`CustomTestKit`实例的大型测试，在测试后正确关机很重要。关机机制取决于测试框架以及如何构造`CustomTestKit`，如下所示：

#### ScalaTest

`CustomTestKit`自动关闭，除非你覆盖了`afterAll()`方法。不需要采取进一步的行动。

#### TestNG

使用`@AfterClass`注解来注释一个方法调用`shutdown()`，如下所示：

```java
    @AfterClass
    public void tearDown() {
        shutdown();
    }
```

#### JUnit

JUnit为类中的每个测试创建类的实例。这也意味着为每个测试方法创建一个新的`Unicomplex`实例。这可能是资源密集型的。要确保正确地关闭各个实例，请使用`@After` JUnit注解来注释一个方法调用`shutdown()`，如下所示：

```java
    @After
    public void tearDown() {
        shutdown();
    }
```

注意：如果您构造`CustomTestKit`时，在`super(boot)`调用上传递一个`UnicomplexBoot`对象，那么请注意如何以及何时关闭。如果`UnicomplexBoot`实例是为每个类创建的，这意味着一个实例用于所有测试方法，那么关闭也只需要发生一次。使用JUnit的`@AfterClass`注解来注释静态关闭方法。但是如果`UnicomplexBoot`实例是根据每个测试方法创建的 - 默认的行为，`@After`注释应该使用类似于`CustomTestKit`的默认构造。

## 使用Akka Http TestKit测试Akka Http路由

为了测试路由，`akka-http-testkit`需要添加到依赖中。请增加如下依赖：

```
"com.typesafe.akka" %% "akka-http-testkit" % akkaHttpV % "test"
```

### 用法

squbs testkit提供实用工具用于从`RouteDefinition`特质*(Scala)* 或`AbstractRouteDefinition`类*(Java)*构造路由，也有实用工具用于单机和整个系统模式，在这里基础设施和cube可以作为测试的一部分。

##### Scala

`TestRoute`用于从`RouteDefinition`构造和获得路由。要使用它，仅需将`RouteDefinition`作为一个类型参数传递给`TestRoute`。这将为您获得一个完全配置和功能路由用于测试DSL, 如下面的示例所示。

指定`RouteDefinition`

```scala
import org.squbs.unicomplex.RouteDefinition

class MyRoute extends RouteDefinition {

  val route =
    path("ping") {
      get {
        complete {
          "pong"
        }
      }
    }
}
```

实现测试，从`TestRoute[MyRoute]`获取路由：

```scala
import akka.http.scaladsl.testkit.ScalatestRouteTest
import org.scalatest.{Matchers, FlatSpecLike}
import org.squbs.testkit.TestRoute

class MyRouteTest extends FlatSpecLike with Matchers with ScalatestRouteTest {

  val route = TestRoute[MyRoute]

  it should "return pong on a ping" in {
    Get("/ping") ~> route ~> check {
      responseAs[String] should be ("pong")
    }
  }
}
```

或者，你也可能希望将web上下文传递到你的路由。这可以通过将其传递给`TestRoute`，如下所示：

```scala
  val route = TestRoute[MyRoute](webContext = "mycontext")
```

或者仅传递`"mycontext"`不带参数名称。没有参数的`TestRoute`签名等同于传递根上下文`""`。

##### Java

为了测试路由，测试需要为您选择的测试框架扩展适当的`RouteTest`抽象类。选择扩展`org.squbs.testkit.japi.TestNGRouteTest`如果您正在使用TestNG，或`org.squbs.testkit.japi.JUnitRouteTest`如果您正在使用JUnit。两者的用法是一样的。

一旦测试扩展了一个`RouteTest`类，你可以从`AbstractRouteDefinition`构造一个`TestRoute`，通过传递类到`testRoute(MyRoute.class)`调用，如下所示：

```java
TestRoute myRoute = testRoute(MyRouteDefinition.class);
```

这将为您获得测试DSL的完整配置和功能路由，如下面的示例所示。

指定`RouteDefinition`

```java
import org.squbs.unicomplex.AbstractRouteDefinition;

class MyRoute extends AbstractRouteDefinition {

    @Override
    public Route route() throws Exception {
        return path("ping", () ->
                get(() ->
                        complete("pong")
                )
        );
    }
}
```

实现测试，从`testRoute(MyRoute.class)`获取路由:

```java
import org.squbs.testkit.japi.JUnitRouteTest;

public class MyRouteTest extends JUnitRouteTest {

    @Test
    public void testPingPongRoute() {
        TestRoute myRoute = testRoute(MyRoute.class);
        myRoute.run(HttpRequest.GET("/ping"))
                .assertStatusCode(200)
                .assertEntity("pong");
    }
}
```

或者，您可能还希望将web上下文传递给路由。这可以通过以下方式将其传递给`TestRoute`来实现：

```java
TestRoute myRoute = testRoute("mycontext", MyRoute.class);
```

没有参数的`TestRoute`签名相当于传递根上下文`""`。

### Using TestRoute with CustomTestKit

在使用`TestRoute`测试时，可能会出现引导`Unicomplex`的需求，例如这个时候：

   * 一个squbs的well-known actor参与了请求处理。
   * 在请求处理期间使用[Actor注册](registry.md)。

将Akka的`TestKit`与`ScalatestRouteTest`一起使用可能会很棘手，因为它们初始化时有冲突。squbs提供了名为`CustomRouteTestKit` *(Scala)*和`JUnitCustomRouteTestKit` *(Java，用于每个测试框架)*的测试工具解决这个问题。`CustomRouteTestKit`支持由`CustomTestKit`提供的所有API。下面是通过`CustomRouteTestKit`使用`TestRoute`的例子：

##### Scala

```scala
class MyRouteTest extends CustomRouteTestKit with FlatSpecLike with Matchers {

  it should "return response from well-known actor" in {
    val route = TestRoute[ReverserRoute]
    Get("/msg/hello") ~> route ~> check {
      responseAs[String] should be ("hello".reverse)
    }
  }
}

class ReverserRoute extends RouteDefinition {
  import akka.pattern.ask
  import Timeouts._
  import context.dispatcher

  val route =
    path("msg" / Segment) { msg =>
      get {
        onComplete((context.actorSelection("/user/mycube/reverser") ? msg).mapTo[String]) {
          case Success(value) => complete(value)
          case Failure(ex)    => complete(s"An error occurred: ${ex.getMessage}")
        }
      }
    }
}
```

##### Java

```java
public class MyRouteTest extends TestNGCustomRouteTestKit {

    @Test
    public void testHelloReverse() {
        TestRoute route = testRoute(ReverserRoute.class);
        route.run(HttpRequest.GET("/msg/hello"))
                .assertStatusCode(200)
                .assertEntity("olleh");
    }
}
```

相应的测试路由如下:

```java
import akka.http.javadsl.server.Route;
import org.squbs.unicomplex.AbstractRouteDefinition;

public class ReverserRoute extends AbstractRouteDefinition {

    @Override
    public Route route() throws Exception {
        return path(segment("msg").slash(segment()), msg ->
                onSuccess(() ->
                        ask(context().actorSelection("/user/mycube/reverser"), msg, 5000L), response ->
                                complete(response.toString())
                        )
                )
        );
    }
}

```

**注意:** 要使用`CustomRouteTestKit`，请确保Akka Http testkit在您所描述的依赖项中，如[上面](#testing-akka-http-routes-using-akka-http-testkit)描述的那样。

#### 关闭

`CustomRouteTestKit`抽象类和特质是特定于测试框架的，因此，它们已经预先装备了测试框架钩子，以正确地启动和关闭。因此，不需要在测试用例中使用任何形式的`CustomRouteTestKit`来关闭`Unicomplex`。
