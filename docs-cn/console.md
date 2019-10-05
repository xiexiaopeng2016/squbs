# 管理控制台

squbs管理控制台为squbs和JVM的系统状态和统计数据提供了一个web/JSON接口。squbs和JVM的所有状态和统计信息都以JMX MXBean的形式提供。Cube、应用或者组件可以随意注册它们的监控器。

**注意**: squbs管理控制台不允许JMX方法的执行或者修改任何设置。

## 依赖

要使用squbs管理控制台，只需添加以下依赖项到`build.sbt`文件或Scala构建脚本:

```
  "org.squbs" %% "squbs-admin" % squbsVersion
```

squbs-admin将被自动检测。不需要API。

## 访问Bean信息

squbs-admin使用上下文`/adm`。仅需命中`/adm`就会列出JMX MXBean目录，以及访问bean的相关URL。如下所示：

```json
{
  "JMImplementation:type=MBeanServerDelegate" : "http://localhost:8080/adm/bean/JMImplementation:type~MBeanServerDelegate",
  "com.sun.management:type=DiagnosticCommand" : "http://localhost:8080/adm/bean/com.sun.management:type~DiagnosticCommand",
  "com.sun.management:type=HotSpotDiagnostic" : "http://localhost:8080/adm/bean/com.sun.management:type~HotSpotDiagnostic",
  "java.lang:type=ClassLoading" : "http://localhost:8080/adm/bean/java.lang:type~ClassLoading",
  "java.lang:type=Compilation" : "http://localhost:8080/adm/bean/java.lang:type~Compilation",
  "java.lang:type=GarbageCollector,name=PS MarkSweep" : "http://localhost:8080/adm/bean/java.lang:type~GarbageCollector,name~PS%20MarkSweep",
  "java.lang:type=GarbageCollector,name=PS Scavenge" : "http://localhost:8080/adm/bean/java.lang:type~GarbageCollector,name~PS%20Scavenge",
  "java.lang:type=Memory" : "http://localhost:8080/adm/bean/java.lang:type~Memory",
  "java.lang:type=MemoryManager,name=CodeCacheManager" : "http://localhost:8080/adm/bean/java.lang:type~MemoryManager,name~CodeCacheManager",
  "java.lang:type=MemoryManager,name=Metaspace Manager" : "http://localhost:8080/adm/bean/java.lang:type~MemoryManager,name~Metaspace%20Manager",
  "java.lang:type=MemoryPool,name=Code Cache" : "http://localhost:8080/adm/bean/java.lang:type~MemoryPool,name~Code%20Cache",
  "java.lang:type=MemoryPool,name=Compressed Class Space" : "http://localhost:8080/adm/bean/java.lang:type~MemoryPool,name~Compressed%20Class%20Space",
  "java.lang:type=MemoryPool,name=Metaspace" : "http://localhost:8080/adm/bean/java.lang:type~MemoryPool,name~Metaspace",
  "java.lang:type=MemoryPool,name=PS Eden Space" : "http://localhost:8080/adm/bean/java.lang:type~MemoryPool,name~PS%20Eden%20Space",
  "java.lang:type=MemoryPool,name=PS Old Gen" : "http://localhost:8080/adm/bean/java.lang:type~MemoryPool,name~PS%20Old%20Gen",
  "java.lang:type=MemoryPool,name=PS Survivor Space" : "http://localhost:8080/adm/bean/java.lang:type~MemoryPool,name~PS%20Survivor%20Space",
  "java.lang:type=OperatingSystem" : "http://localhost:8080/adm/bean/java.lang:type~OperatingSystem",
  "java.lang:type=Runtime" : "http://localhost:8080/adm/bean/java.lang:type~Runtime",
  "java.lang:type=Threading" : "http://localhost:8080/adm/bean/java.lang:type~Threading",
  "java.nio:type=BufferPool,name=direct" : "http://localhost:8080/adm/bean/java.nio:type~BufferPool,name~direct",
  "java.nio:type=BufferPool,name=mapped" : "http://localhost:8080/adm/bean/java.nio:type~BufferPool,name~mapped",
  "java.util.logging:type=Logging" : "http://localhost:8080/adm/bean/java.util.logging:type~Logging",
  "org.squbs.unicomplex:type=CubeState,name=admin" : "http://localhost:8080/adm/bean/org.squbs.unicomplex:type~CubeState,name~admin",
  "org.squbs.unicomplex:type=Cubes" : "http://localhost:8080/adm/bean/org.squbs.unicomplex:type~Cubes",
  "org.squbs.unicomplex:type=Extensions" : "http://localhost:8080/adm/bean/org.squbs.unicomplex:type~Extensions",
  "org.squbs.unicomplex:type=Listeners" : "http://localhost:8080/adm/bean/org.squbs.unicomplex:type~Listeners",
  "org.squbs.unicomplex:type=ServerStats,listener=default-listener" : "http://localhost:8080/adm/bean/org.squbs.unicomplex:type~ServerStats,listener~default-listener",
  "org.squbs.unicomplex:type=SystemSetting" : "http://localhost:8080/adm/bean/org.squbs.unicomplex:type~SystemSetting",
  "org.squbs.unicomplex:type=SystemState" : "http://localhost:8080/adm/bean/org.squbs.unicomplex:type~SystemState"
}
```

浏览器的JSON插件让你可以检测并轻松地点击这些链接。按照提供的链接，例如，`org.squbs.unicomplex:type=Cubes`将展示所有bean的细节，如下所示：

```json
{
  "Cubes" : [
    {
      "fullName" : "org.squbs.admin",
      "name" : "admin",
      "supervisor" : "Actor[akka://squbs/user/admin#104594558]",
      "version" : "0.7.1"
    }
  ]
}
```

## Bean URL & 编码

所有的bean都在`/adm/bean/`路径下，后跟完整的bean对象名称。bean对象名称得到如下转换，以使 URL访问变得容易：

1. bean名称中的=换为~。
2. 所有其他字符都按照标准URL编码进行编码。例如，名称中的空格被编码为`%20`。

例如，使用对象名`java.lang:type=GarbageCollector,name=PS Scavenge`访问bean，URL将是`/adm/bean/java.lang:type~GarbageCollector,name~PS%20Scavenge`。

## 监听器

squbs-admin绑定在`admin-listener`。默认，这是`default-listener`的别名。服务可以覆盖Unicomplex配置，并重新分配`admin-listener`别名到其它已定义的监听器。如下所示如何在`application.conf`重新分配别名：

在`Unicomplex`中定义别名

```
default-listener {
  aliases = [admin-listener]
}
```

在`application.conf`中重写的例子

```
default-listener {
  aliases = []
}

my-listener {
  aliases = [admin-listener]
}
```

## 排除

在许多情况下，MXBean可能包含被视为敏感的信息，不应在管理控制台中显示。当然，最好的解决方法是，首先不要将这些信息包含在JMX bean中。然而，有时这些bean不是由您的组件公开的，而是由第三方组件或库公开的，而且除了修改第三方组件外，无法隐藏这些信息。排除是管理控制台提供的一项功能，用于隐藏此类信息在JSON中的显示。

### 配置排除

在application.conf中排除是在squbs.admin下配置，如下所示
排除是在squbs.admin下配置的，在你的标准配置`application.conf`中，可以在下面的例子中看到：

```
squbs.admin {
  exclusions = [
    "java.lang:type=Memory",
    "java.lang:type=GarbageCollector,name=PS MarkSweep::init",
    "java.lang:type=GarbageCollector,name=PS MarkSweep::startTime"
  ]
}
```

* 要从控制台的视图中排除整个MXBean，请在排除项中列出bean对象名称。 
* 若要排除MXBean中的字段或属性，请在排除项中列出bean对象名称和字段名称。
* bean对象名和字段名之间用`::`分隔。
* 字段名充当MXBean中任意深度的任意属性或字段的过滤器。如果字段名与提供的名称匹配，则将其排除。
* 可以通过列出`beanName::field1`，`beanName::field2`等来实现同一个bean上的多个字段排除。

## 错误响应

当请求获得无效bean对象名称，或者一个被列入黑名单bean时，会得到一个404(NotFound)响应。由于安全原因，这两种情况的结果无法区分。
