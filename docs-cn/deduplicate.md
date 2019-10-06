# 重复数据删除阶段

### 概述

`Deduplicate`是一个Akka流`GraphStage`用于删除一个流中相同(连续或非连续)的元素。

### 依赖

将以下依赖项添加到您的`build.sbt`或scala构建文件：

```
"org.squbs" %% "squbs-ext" % squbsVersion
```

### 用法

用法与标准Akka Stream阶段非常相似：

```scala
val result = Source("a" :: "b" :: "b" :: "c" :: "a" :: "a" :: "a" :: "c" :: Nil).
  via(Deduplicate()).
  runWith(Sink.seq)
  
// Output: ("a" :: "b" :: "c" :: Nil)
```

`Deduplicate`保存一个已经见过的元素的注册表。为了防止注册表无限制地增长，它允许为每个消息指定重复的数目。一旦到达`duplicateCount`，该元素将从注册表中删除。在下面的例子中，`duplicateCount`被指定为`2`，因此，`"a"`在第三次出现时不会被删除：

```scala
val result = Source("a" :: "b" :: "b" :: "c" :: "a" :: "a" :: "a" :: "c" :: Nil).
  via(Deduplicate(2)).
  runWith(Sink.seq)
  
// Output: ("a" :: "b" :: "c" :: "a" :: Nil)
```

请注意，`duplicateCount`可以防止注册表在重复的数量已知的情况下不断增长。但是，仍然存在内存泄漏的可能性。例如，如果`duplicateCount`被设置为`2`，那么一个元素将一直保存在注册表中，直到看到重复为止; 然而，在某些情况下，可能永远不会出现重复，例如，使用了`filter`或`drop`。因此，请注意在您的用例中`Duplicate`的后果。

你也可以提供一个不同的注册表实现`Deduplicate`，定期清理自己。但是，只有在您确定在给定的时间范围内不会看到副本时，才应该这样做; 否则，副本逻辑可能被破坏。

### 配置注册表项和注册表实现

`Deduplicate`默认使用元素本身作为注册表的键。但是，它也接受一个映射元素到键的函数。例如，如果流中的元素是`(Int, String)`类型的元组并且你想要识别重复的只是基于元组的第一个字段，你可以传递一个函数如下:

```scala
val deduplicate = Deduplicate((element: (Int, String)) => element._1, 2) // 2是默认值
```  

`Deduplicate`还允许将注册表替换为另一个类型`java.util.Map[Key, MutableLong]`的实现。

```scala
val deduplicate = Deduplicate(2, new util.TreeMap[String, MutableLong]())
```