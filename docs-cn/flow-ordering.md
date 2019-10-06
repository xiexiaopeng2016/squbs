# 有界排序

### 概述

某些流拓扑，如调用HTTP客户端或`mapAsyncUnordered`的流拓扑，允许流元素失去次序。有界排序阶段允许流在需要时按顺序返回。这种排序是使用滑动窗口算法的最佳结果。如果流在给定的有界滑动窗口大小内无序，则可以实现总的排序。但是，如果元素的顺序超出了滑动窗口的大小，则流不能也不会等待。无序的元素将被跳过，或者如果它到达窗口之外的时间较晚，将无序地向下传递。这个有界的滑动窗口限制是为了防止流饥饿和内存泄漏从缓冲区建立。

### 依赖

将以下依赖项添加到您的`build.sbt`或scala构建文件：

```
"org.squbs" %% "squbs-ext" % squbsVersion
```

### 用法

有界排序功能由`BoundedOrdering`组件提供，该组件暴露为`Flow`组件，可以使用`via`操作符连接到一个流`Source`或另一个`Flow`组件。通过`BoundedOrdering`的元素期望具有可预测的、可排序的和连续的id，我们可以从该元素派生该id。这个id通常是`Long`类型，但也可以是任何其他类型，只要它的顺序是可预测的。

`BoundedOrdering`的创建需要一些输入：

1. `maxBounded`参数定义了滑动窗口的大小，`BoundedOrdering`将等待无序元素。
2. `initialId`参数是第一个元素的初始id。通过这种方式，流将知道第一个元素是否丢失或无序。
3. `nextId`参数指定从当前元素的id派生后续元素的id的函数。
4. `getId`参数指定用于从元素中提取id的函数。
5. 可选地，如果系统不知道如何比较和排序id类型，可以传递一个`Comparator`。

初始化和使用可以在以下例子中看到：

##### Scala

```scala
// This is the sample message type.
case class Element(id: Long, content: String)

// Create the BoundedOrdering component.
val boundedOrdering = BoundedOrdering[Element, Long](maxBounded = 5, 1L, _ + 1L, _.id)

// Then just use it in the stream.
Source(input).via(boundedOrdering).to(Sink.ignore).run()
```

##### Java

```java
// This is the sample message type.
static class Element {
    final Long id;
    final String content;

    // Omitting constructor, hashCode, equals, and toString for brevity.
    // Do not forget to implement.
}

// Create the BoundeOrdering component.
Flow<Element, Element, NotUsed> boundedOrdering = BoundedOrdering.create(5, 1L, i -> i + 1L, e -> e.id);

// Then just use it in the stream.
Source.from(input).via(boundedOrdering).to(Sink.ignore()).run(mat);
```

#### 注意

这一阶段可能在一定程度上暂时分离上下游需求。这将发生在元素到达顺序错误时。无序的元素将被缓冲，直到它的顺序到达，或者边界到达。当这种情况发生时，需求从该组件的上游产生，而不排放下游的元素，直到达到给定的条件。
