# 验证

squbs验证通过使用[Accord验证库](http://wix.github.io/accord/)为数据验证提供了一个[Akka HTTP](http://doc.akka.io/)指令。目前这是Scala独有的特性，Java版本将会添加到将来的squbs中。
  
## 依赖

将以下依赖项添加到您的`build.sbt`或scala构建文件:

```scala
"org.squbs" %% "squbs-pattern" % squbsVersion,
"com.wix" %% "accord-core" % "0.7.1"
```  
  
## 用法

考虑到隐式的`Person`验证器在作用域内，`validate`指令可以用作其他[Akka HTTP指令](http://doc.akka.io/docs/akka-http/current/scala/http/routing-dsl/directives/index.html)：
  
```scala
import ValidationDirectives._
validate(person) { 
    ...
}
```  

## 示例

这里是一个示例`Person`类和相应的验证器(请参阅[Accord验证库](http://wix.github.io/accord/)以获得更多的验证器使用示例)。

```scala
case class Person(firstName: String, lastName: String, middleName: Option[String] = None, age: Int)

object SampleValidators {

  import com.wix.accord.dsl._
  implicit val personValidator = com.wix.accord.dsl.validator[ Person ] { p =>
                p.firstName as "First Name" is notEmpty
                p.lastName as "Last Name" is notEmpty
                p.middleName.each is notEmpty // If exists, should not be empty.
                p.age should be >= 0
              }
}
```

现在你可以使用`validate`指令如下：
 
```scala
def route =
 path("person") {
   post {
     entity(as[Person]) { person =>
       import ValidationDirectives._
       // importing the person validator
       import SampleValidators._
       validate(person) {
           complete {
             person
           }
       }
     }
   }
 }
```

如果发生验证拒绝，则返回`400 Bad Request`，响应体中包含逗号分隔的导致验证拒绝的字段列表。使用上面的例子，如果请求体包含以下内容：
 
```
{
    "firstName" : "John",
    "lastName" : "",
    "age" : -1
}
```

然后，响应体将包含：
 
```
Last Name, age 
```