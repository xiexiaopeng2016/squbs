# 编组和解组

### 概述

编组和解组同时在客户端和服务器端使用。在服务器端，它用于将传入请求映射到Scala或Java对象，并将scala或Java对象映射到一个传出的响应。同样地，在客户端，它用于编组对象到一个发出HTTP请求和从一个传入的响应解组对象。有许多内容格式用于编组/解组，常见的是JSON和XML。参阅以下页面，以获取Akka HTTP中JSON编组和解组的快速示例:

* Scala - [spray-json Support](http://doc.akka.io/docs/akka-http/current/scala/http/common/json-support.html#spray-json-support)
* Java - [Jackson Support](http://doc.akka.io/docs/akka-http/current/java/http/common/json-support.html#json-support-via-jackson).

Akka HTTP提供了编组/解组工具，阐述于[Scala marshalling](http://doc.akka.io/docs/akka-http/current/scala/http/common/marshalling.html)/[unmarshalling](http://doc.akka.io/docs/akka-http/current/scala/http/common/unmarshalling.html)和[Java marshalling](http://doc.akka.io/docs/akka-http/current/java/http/common/marshalling.html)/[unmarshalling](http://doc.akka.io/docs/akka-http/current/java/http/common/unmarshalling.html)。此外，还有其它的开源编组和解组工具为Akka HTTP提供不同格式和使用不同对象实现编组/解组。

squbs为跨语言环境提供编组工具。例如，在处理既有Scala样例类，也有Java bean的混合对象层次结构时。对于简单、单语言的环境，请直接使用已提供的编组工具或其他开源编组工具。

此外，squbs还添加了一个用于手动编组/解组的Java API。手动访问编组器和解组器对于基于流的应用程序非常有用，在这些应用程序中，一些工作可能需要在一个流阶段完成。它还有助于测试编组器配置，以确保获得正确的格式。


本文档讨论squbs提供的编组器和解组器，以及可以用来手动调用这些编组器和解组器的工具。本文档**没有**涉及作为Akka HTTP路由DSL那部分的编组器和解组器的使用。

请查看Akka HTTP路由DSL [Scala](http://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/directives/marshalling-directives/index.html#marshallingdirectives)和[Java](http://doc.akka.io/docs/akka-http/current/java/http/routing-dsl/directives/marshalling-directives/index.html#marshallingdirectives-java)用于在路由DSL中使用编组器，包括本文档中提供的一个，在路由DSL中。

### 依赖

增加如下依赖到你的`build.sbt`或者scala构建文件：

```scala
"org.squbs" %% "squbs-ext" % squbsVersion,
"de.heikoseeberger" %% "akka-http-json4s" % heikoseebergerAkkaHttpJsonVersion,
"de.heikoseeberger" %% "akka-http-jackson" % heikoseebergerAkkaHttpJsonVersion,
"com.fasterxml.jackson.core" % "jackson-core" % jacksonVersion,
"com.fasterxml.jackson.core" % "jackson-databind" % jacksonVersion
```

以下是可选的组件，具体取决于您要使用的编组格式和库。它们可以组合起来支持多种格式。

```scala
// To use json4s native...
"org.json4s" %% "json4s-native" % "3.5.0",

// To use json4s jackson...
"org.json4s" %% "json4s-jackson" % "3.5.0",
  
// For Jackson marshalling of Scala case classes... 
"com.fasterxml.jackson.module" %% "jackson-module-scala" % jacksonVersion,
  
// For Jackson marshalling of immutable Java classes...
"com.fasterxml.jackson.module" % "jackson-module-parameter-names" % jacksonVersion,
```  

### 用法

#### JacksonMapperSupport

`JacksonMapperSupport`提供JSON编组/解组，基于广流行的Jackson库。它允许Jackson的`ObjectMapper`按全局和按每种类型配置。

请查看[Jackson数据绑定文档](http://wiki.fasterxml.com/JacksonFAQ#Data_Binding.2C_general)有关`ObjectMapper`配置的细节。

##### Scala

你仅需要在Scala代码中引入`JacksonMapperSupport._`，以便将它的隐式成员暴露在编组/解组的作用域内：

```scala
import org.squbs.marshallers.json.JacksonMapperSupport._
```

自动和手工编组工具都将隐式使用此包提供的编组工具。下面的代码展示了使用`ObjectMapper`配置`DefaultScalaModule`的各种方法：

```scala
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.module.scala.DefaultScalaModule
import org.squbs.marshallers.json.JacksonMapperSupport

/* To use DefaultScalaModule for all classes.
 */

JacksonMapperSupport.setDefaultMapper(
  new ObjectMapper().registerModule(DefaultScalaModule))


/* To register a 'DefaultScalaModule' for marshalling a
 * specific class, overrides global configuration for
 * this class.
 */

JacksonMapperSupport.register[MyScalaClass](
  new ObjectMapper().registerModule(DefaultScalaModule))
```

##### Java

编组器和解组器可以从`JacksonMapperSupport`中的`marshaller`和`unmarshaller`方法中获得，将类型的类实例传递给编组/解组的方法如下:

```java
import akka.http.javadsl.marshalling.Marshaller;
import akka.http.javadsl.model.HttpEntity;
import akka.http.javadsl.model.RequestEntity;
import akka.http.javadsl.unmarshalling.Unmarshaller;

import static org.squbs.marshallers.json.JacksonMapperSupport.*;

Marshaller<MyClass, RequestEntity> myMarshaller =
    marshaller(MyClass.class);

Unmarshaller<HttpEntity, MyClass> myUnmarshaller =
    unmarshaller(MyClass.class);
```

可将这些编组器和解组器用作[Akka HTTP Routing DSL](http://doc.akka.io/docs/akka-http/current/java/http/routing-dsl/overview.html)的一部分或[invoking marshalling/unmarshalling](#invoking-marshallingunmarshalling)的一部分，随后将在本文件中讨论。

以下示例使用`ObjectMapper`配置`DefaultScalaModule`：

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.module.scala.DefaultScalaModule;
import org.squbs.marshallers.json.JacksonMapperSupport;

/* Globally registers the 'DefaultScalaModule'.
 */
 
JacksonMapperSupport.setDefaultMapper(
  new ObjectMapper().registerModule(new DefaultScalaModule()));


/* This example below registers the 'DefaultScalaModule'
 * just for 'MyClass'
 */

JacksonMapperSupport.register(MyClass.class
  new ObjectMapper().registerModule(new DefaultScalaModule()));
```


#### XLangJsonSupport

XLangJsonSupport 通过委派编组和解组来增加跨语言支持：

* Json4s for Scala classes
* Jackson for Java classes

这些通常是每种语言的首选编组工具，因为它们支持特定于语言的约定，而无需进一步配置。它们通常对不同的约定也有更好的优化。

然而，使用Json4s或Jackson的决定是由传递给编组/解组的对象的类型决定的。如果您有混合对象层次结构，则可能仍需要配置编组/解组工具来支持不同的约定，如下所示：

* 引用Java Bean的Scala样例类。由于顶级对象是Scala样例类，因此将选择Json4s。但它不知道如何编组/解组Java Bean。需要将一个自定义序列化程序添加到Json4s来处理这些Java bean。
* 引用Scala样例类的Java Bean。由于顶级对象是Java Bean，因此将选择Jackson。Jackson默认不知道如何编组/解组样例类。你需要注册`DefaultScalaModule`到Jackson `ObjectMapper`来处理这样的样例。

编组/解组混合语言对象层次结构的一般准则：除非首选Json4s优化，否则更容易通过将`DefaultScalaModule`注册到`ObjectMapper`，来配置Jackson来处理Scala。

与`JacksonMapperSupport`一样，它支持编组和解组的每种类型配置。它支持Json4s和Jackson的配置。

##### Scala

你仅需要在Scala代码中引入`XLangJsonSupport._`，以便将它的隐式成员暴露在编组/解组的作用域内：

```scala
import org.squbs.marshallers.json.XLangJsonSupport._
```

自动和手工编组工具都将隐式使用此包提供的编组工具。下面的代码提供了`XLangJsonSupport`配置案例：

```scala
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.module.paramnames.ParameterNamesModule
import org.json4s.{DefaultFormats, jackson, native}
import org.squbs.marshallers.json.XLangJsonSupport

/* The following configures the default settings
 * for 'XLangJsonSupport'
 */

// Adds ParameterNamesModule to Jackson
XLangJsonSupport.setDefaultMapper(
  new ObjectMapper().registerModule(new ParameterNamesModule())
  
// Tells Json4s to use native serialization
XLangJsonSupport.setDefaultSerialization(native.Serialization)

// Adds MySerializer to the serializers used by Json4s
XLangJsonSupport.setDefaultFormats(DefaultFormats + MySerializer)


/* The following configures XLangJsonSupport for specific class.
 * Namely, it configures for 'MyClass' and 'MyOtherClass'.
 */

// Use ParameterNamesModule for mashal/unmarshal MyClass
XLangJsonSupport.register[MyClass](new ParameterNamesModule()))

// Use Json4s Jackson serialization for MyOtherClass
XLangJsonSupport.register[MyOtherClass](jackson.Serialization)

// Use MySerializer Json4s serializer for MyOtherClass
XLangJsonSupport.register[MyOtherClass](DefaultFormats + MySerializer)
```

##### Java

编组器和解组器可以从`XLangJsonSupport`中的`marshaller`和`unmarshaller`方法中获得，将类型的类实例传递给`marshal`/`unmarshal`，如下所示：

```java
import akka.http.javadsl.marshalling.Marshaller;
import akka.http.javadsl.model.HttpEntity;
import akka.http.javadsl.model.RequestEntity;
import akka.http.javadsl.unmarshalling.Unmarshaller;

import static org.squbs.marshallers.json.XLangJsonSupport.*;

Marshaller<MyClass, RequestEntity> myMarshaller =
    marshaller(MyClass.class);

Unmarshaller<HttpEntity, MyClass> myUnmarshaller =
    unmarshaller(MyClass.class);
```

这些编组器和解组器被使用，作为[Akka HTTP Routing DSL](http://doc.akka.io/docs/akka-http/current/java/http/routing-dsl/overview.html)的一部分或[invoking marshalling/unmarshalling](#invoking-marshallingunmarshalling)的一部分，后续将在本文件中讨论。

下面提供了配置`XLangJsonSupport`的示例:

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.module.paramnames.ParameterNamesModule;
import org.squbs.marshallers.json.XLangJsonSupport;

/* Global XLangJsonSupport Configuration.
 */

// Adds ParameterNamesModule to Jackson
XLangJsonSupport.setDefaultMapper(
  new ObjectMapper().registerModule(new ParameterNamesModule());
  
// Tells Json4s to use native serialization
XLangJsonSupport.setDefaultSerialization(XLangJsonSupport.nativeSerialization());

// Adds MySerializer and MyOtherSerializer (varargs) to the serializers used by Json4s
XLangJsonSupport.addDefaultSerializers(new MySerializer(), new MyOtherSerializer());


/* Per-class configuration of 'XLangJsonSupport'.
 * In this case we show configuring 'MyClass' and 'MyOtherClass'
 */

// Use ParameterNamesModule for mashal/unmarshal MyClass
XLangJsonSupport.register(MyClass.class, new ParameterNamesModule()));

// Use Json4s Jackson serialization for MyOtherClass
XLangJsonSupport.register(MyOtherClass.class, XLangJsonSupport.jacksonSerialization());

// Adds MySerializer and MyOtherSerializer (varargs) to the serializers used by Json4s for MyOtherClass
XLangJsonSupport.addSerializers(MyOtherClass.class, new MySerializer(), new MyOtherSerializer());
```

#### 调用编组/解组

除了使用编组和解组作为Akka HTTP路由DSL的一部分之外，还经常需要手动调用编组和解组，以便在服务器端和客户端`Flow`以及测试中使用。

##### Scala

Akka提供了一个强大的[Scala DSL for marshalling and unmarshalling](http://doc.akka.io/docs/akka-http/current/scala/http/common/marshalling.html#using-marshallers)。它的使用可以在下面的例子中看到：

```scala
import akka.actor.ActorSystem
import akka.http.scaladsl.marshalling.Marshal
import akka.http.scaladsl.model.MessageEntity
import akka.http.scaladsl.unmarshalling.Unmarshal
import akka.stream.ActorMaterializer

// We need the ActorSystem and Materializer to marshal/unmarshal
implicit val system = ActorSystem()
implicit val mat = ActorMaterializer()

// Also need the implicit marshallers provided by this import
import org.squbs.marshallers.json.XLangJsonSupport._

// Just call Marshal or Unmarshal as follows:
Marshal(myObject).to[MessageEntity]
Unmarshal(entity).to[MyType]
```

##### Java

`MarshalUnmarshal`实用程序类用于手动编组和解组对象，使用Akka HTTP的JavaDSL中定义的任何`Marshaller`和`Unmarshaller`。它的用法可以在下面的例子中看到：

```java
import akka.actor.ActorSystem;
import akka.http.javadsl.model.RequestEntity;
import akka.stream.ActorMaterializer;
import akka.stream.Materializer;

import org.squbs.marshallers.MarshalUnmarshal;

// We're using JacksonMapperSupport here.
// But XLangJsonSupport works the same.
import static org.squbs.marshallers.json.JacksonMapperSupport.*;

// Base infrastructure, and the 'mu' MarshalUnmarshal. 
private final ActorSystem system = ActorSystem.create();
private final Materializer mat = ActorMaterializer.create(system);
private final MarshalUnmarshal mu = new MarshalUnmarshal(system.dispatcher(), mat);

// Call 'apply' passing marshaller or unmarshaller as follows, using marshaller
// and unmarshaller methods from 'import static JacksonMapperSupport.*;':
CompletionStage<RequestEntity> mf = mu.apply(marshaller(MyClass.class), myObject);
CompletionStage<MyClass> uf = mu.apply(unmarshaller(MyClass.class), entity);
```