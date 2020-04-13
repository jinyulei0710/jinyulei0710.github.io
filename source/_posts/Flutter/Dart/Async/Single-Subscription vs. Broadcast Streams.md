---
layout: post
title:  "(译)单订阅流 vs. 广播流"
date:   2019-04-29 16:45:17 +0800
categories: 
   - Flutter 开发
tags:
   - Dart
   - Async
---

[原文链接](https://www.dartlang.org/articles/libraries/broadcast-streams)

由Florian Loitsch撰写

写于2014年一月

Dart具有两种不同风格的 Streams ：单订阅流和广播流。本文讨论了两者之间的差异，并提供了何时使用哪个的建议。

如果你还不熟悉Dart流，可以从教程[异步编程：Streams](/2019/04/28/Dart/Async/AsynchronousProgrammingStreams/)中学习基础知识。

<!--more-->


## 介绍

单订阅和广播流旨在用于不同的上下文并具有不同的要求。在许多方面，它们类似于TCP和UDP：单订阅流是稳定的，具有保证的属性（如TCP），而广播流可能丢失事件，而监听器与源没有紧密的连接（如UDP）。

以下列表总结了两者之间的主要区别：

||	Single subscription|Broadcast|
|---|---|---|
|监听者的数量|1| ∞|
|是否可以丢失事件|否|是|
|良好定义的生命周期|是|否|
|易用性|难|简单|

让我们详细看看每个差异。

**监听者数量**

单订阅流只允许监听流一次。这包括内部监听调用（例如对isNotEmpty的调用)。广播流没有此限制。

**可以丢失事件**

单订阅流不会丢失事件。如有必要，他们会缓冲数据直到监听者出现。如果没有任何监听器，广播流可以自由地丢弃事件。

**定义明确的生命周期**

单订阅流具有明确定义的生命周期：它们在用户开始监听时启动，并在用户取消或流发送关闭事件时结束。广播流通常不会给监听者由监听者结束影响其生命周期的方法。

正确获取流的生命周期非常重要：在 dart 中：io 流的生命周期决定了Dart程序对其用于传递数据的系统资源的持有时间。流类型的错误选择可能导致资源泄漏。

**易用性**

在内部，许多Stream成员（例如第一个）开始监听流。由于单订阅流只允许一个监听器，因此在它们上调用getter和方法可以自动使用该流。因此，单一订阅流不如广播流使用方便，广播流没有这种限制。

然而，广播流有其自身的陷阱。例如，isEmpty仅在使用元素后返回false。然后，用户可以继续收听流，但第一个事件将丢失。

在详细讨论流类型之前，让我们仔细看看订阅，这对于理解两种流的工作方式至关重要。

## 订阅(Subscriptions)

从流（单订阅和广播）接收事件的最简单方法是调用其listen方法：stream.listen（onDataCallback）。实际上，listen实际上是不同流的实现方式不同的唯一方法。它将流的侦听器连接到其事件源。所有其他方法都可以（通常是）在listen方法之上实现。

listen方法创建新的流订阅，将其连接到流的事件源，并在订阅中安装回调（如果给定）。然后从listen调用返回该流订阅。回调的安装只是为了方便。订阅的处理程序可以在以后更改。通常只是为了接收流订阅而看到stream.listen（null）并不罕见。

一旦流创建了流订阅，它就会将事件生成和传播移交给订阅。大多数Stream的实现甚至都没有跟踪他们的听众。一旦流创建订阅，流就不再需要知道有关侦听器的任何信息。

下图解释了listen调用：

![](listen.png)

请注意，一旦调用了listen方法，单个订阅流就不需要保留对事件源的引用。这就是为什么从Stream到事件源的箭头在“after”图片中被破坏的原因。另请注意，事件源永远不会看到（或需要查看）流。事件源仅需要对其订阅的引用。作为回报，流订阅需要指向其事件源的指针，以便它可以取消其订阅，从而关闭事件源。

此设置的另一个结果是 Stream 实例不知道事件源的状态。特别是，他们不知道事件源是否暂停。 StreamSubscription 实例接管此任务。实际上，流订阅可确保遵循暂停请求。如果无法暂停实际事件源（无论出于何种原因），则流订阅必须缓冲事件。通常，如果创建的订阅能够暂停其事件源，则流应通知其用户。不知道的话很容易导致过多的内存使用。

## 单订阅流

单订阅流只允许一个订阅者。当消费者可以触发事件的生成（“生产”）并且丢失事件将是错误时，使用单订阅流。这种流的最佳示例是 File.openRead()：用户通过监听来启动流，并且用户通常不希望丢失事件。

单一订阅流只能被听一次这一事实具有一些重要意义。首先，最重要的是，对于单一订阅流，当流应该生成事件时，以及何时应该停止这样做时，这是非常明显的。其次，更令人讨厌的是，许多 Stream getter 和方法对于单订阅流不太有用。例如，只需使用 isNotEmpty getter在内部监听流并在获取数据后取消其订阅。此时，单订阅流已完成：流已经有一个监听器（isNotEmpty getter），并且不允许其他监听。同样，第一个getter只能被调用一次。

### 创建单订阅流

创建流的最简单方法是使用 StreamController ，其中默认构造函数使用单订阅流实例化控制器。

控制器使用add，addError 和 close 等方法实现 StreamSink 接口。事件源（例如本机I / O扩展）只在具有新数据时调用相应的函数。 StreamControllers为流的实现者提供了一个简单的抽象，但从根本上说，订阅部分中解释的概念仍然有效，如下图所示。

![](listen-with-controller.png)

由于控制器在用户侦听其流之前存在，因此事件源可以在用户开始监听之前将数据添加到控制器。为避免数据丢失，控制器会缓冲数据，直到用户开始收听。

此安全措施是流控制器中滥用最多的功能之一。在实例化控制器时，您可以注册onListen回调以在侦听器存在时得到通知。只有在调用此回调之后，才应将事件源添加到控制器。同样，您应该注册一个关闭流的onCancel回调。当用户取消订阅时，或者使用close（）关闭控制器时，将调用onCancel回调。

请注意，仅在流具有侦听器时生成事件不是严格的规则，而是一个很好的准则。存在一些完全有效的用例，用于在任何订阅者收听之前向控制器添加数据。但请注意，StreamSubscription的缓冲方法仅针对少数事件进行了优化，并且不会滥用它。

通常，避免数据生成的一种好方法是延迟计算直到监听器存在。例如，在侦听器存在之前，File.openRead（）不会触及文件系统。这个事实的一个奇怪的含义是File.openRead（）如果在不存在的文件上调用它就不会抛出。仅当用户开始侦听文件内容时，流才会打开硬盘驱动器上的文件。如果在侦听器订阅之前的时间（通过其他方式）创建文件，则流正常工作。直接的推论是调用File.openRead（）不会将文件锁定在硬盘驱动器上。

作为一个很好的指导原则，假设从未收听过流。此规则的例外是小型本地代码片段，例如测试，其中保证订阅发生。如果必须侦听流（以避免内存泄漏），请在创建流的函数的注释中明确说明此事实。

## 广播流
作为单订阅流的对应物，Dart还带有广播流。它们的预期用途是用于独立于听众产生输出的事件源，并且缺少某些事件不是问题。通常，所有DOM事件源都是广播流。

我们可以在多次监听呼叫后扩展我们的广播流订阅图：
![](broadcast-listen.png)

StreamSubscription实例再次负责确保暂停请求得到遵守。由于流订阅通常彼此不了解，因此最简单的解决方案是在订阅级别缓冲所有传入事件。如果订阅未恢复或取消，这可能导致内存泄漏。

能够多次收听流有一些很好的含义。其中一个是第一个，isEmpty，等等不要使流无法使用。作为回报，确定流的生命周期更加困难。流应该关闭吗？如果是，它应该在它丢失所有订阅之后关闭（只有在拥有至少一个订阅者的情况下才使流多订阅），或者只有在特定的关键数据事件没有时才允许订阅者被排出？所有这些提议都是有效的，因此广播流应记录其结束行为。通常我们假设没有文档意味着流具有无限的生命周期。

### 创建广播流
StreamController 类附带了一个专为广播流设计的工厂构造函数StreamController.broadcast。与单订阅流相反，它不会缓冲事件。它或者直接向订阅者发送事件，或者如果没有，则丢弃事件。


>注意：没有缓冲，广播控制器似乎更简单，因此实现更快。但是，他们需要处理在上一个事件尚未分发时添加到控制器的事件。因此，当前的广播实现比单订阅流的实现更复杂。

StreamController.broadcast构造函数还有两个回调：onListen和onCancel。只要控制器分别从未订阅切换到订阅，从订阅切换到取消订阅，都会调用它们（包括当流从事件源端关闭时）。

创建流控制器的一种更危险的方法是通过 asBroadcastStream() 查看单订阅控制器。调用asBroadcastStream 基本上告诉用户想要接管流的生命周期管理的单订阅流。与cancelOnError 订阅者一起使用时，这很容易导致从未关闭的单流订阅，从而泄漏内存或资源。

以下示例演示了这种情况：

```dart
import 'dart:io';
import 'dart:async';

main() {
  ServerSocket.bind("localhost", 4999).then((socket) {
    socket.asBroadcastStream()  // <== asBroadcastStream.
       .map((x) { throw "oops, my mistake"; })
       .listen(print)
       .asFuture()  // Automatically cancels on error.
       .catchError((_) { print("caught error"); });
  });
}
```


在此示例中，单订阅流套接字被视为广播流。 asBroadcastStream在其唯一的侦听器取消后不会关闭，并且套接字保持打开状态。这是资源泄漏的典型情况。如果省略了asBroadcastStream()调用，则会通知单订阅流其订户已​​消失并将关闭套接字。

请记住：asBroadcastStream 很危险。仅在极少数情况下使用它。我们已经看到asBroadcastStream 在监听器更改和 StreamIterators 是更好的选择的情况下使用。有关示例，请参阅下一节。

或者，使用可以给 asBroadcastStream 的回调。它们允许管理订阅更改（类似于StreamController.broadcast构造函数），并提供取消订阅单订阅流的方法。


### asBroadcastStream的替代品
通常asBroadcastStream是（ab）使用的，因此可以多次调用 Stream getter 和方法，例如first和take。如上所示，这是一个危险的权衡。本节介绍一些更安全的替代方案。

#### 监听者交换
流订阅允许交换他们的听众。您可以经常只在订阅上交换监听器回调，而不是在流上多次调用。例如：


```dart
var bstream = stream.asBroadcastStream();
bstream.first.then((x) {
  handleFirstMessage();
  return bstream.first;
}).then((x) {
  handleSecondMessage();
  bstream.listen(handleAllOtherMessages);
});
```

可以写成：

```
StreamSubscription subscription = stream.listen(null);
subscription.onData((x) {
  handleFirstMessage();
  subscription.onData((x) {
    handleSecondMessage();
    subscription.onData(handleAllOtherMessages);
  });
});
```
这引入了一些嵌套，但是一些抽象（或使用方法而不是匿名闭包）可以轻松地摆脱它们。一种流行的抽象是，例如，状态机。另一个是StreamIterator类。

#### StreamIterator

通常，异步流的不同事件将发往系统的不同部分。在这种情况下，StreamIterator通常很方便。 StreamIterators的事件被逐一拉出，每个部分都可以单独拉出它需要的事件。但是，该操作仍然是异步的：与同步迭代器相反，移动到下一个事件可能需要一些时间，因此moveNext函数返回一个未来。

```dart
Future moveNextAssert(iterator) {
  var future = iterator.moveNext();
  return future.then((hasNext) {
    if (!hasNext) throw new StateError("missing element");
    return iterator.current;
  });
}

var lines = new File(...).openRead()
    .transform(utf8.decoder)
    .transform(new LineSplitter());
var iterator = new StreamIterator(lines);
moveNextAssert(iterator)
  .then((line) {
    print("First line: $line");
    return moveNextAssert(iterator);
  })
  .then((fileName) {  // Assume second line is a file.
    return handleFile(fileName)  // Wait for it to finish.
      .then((_) => moveNextAssert(iterator));
  })
  .then((line) {
    print("Last line: $line");
    return iterator.moveNext();
  })
  .then((hasNext) {
    if (hasNext) throw "More lines than expected";
  });

```
## 结论
尽管它们有相似之处，但单一订阅和广播流有重要区别。您需要了解这些差异，这样可以避免资源泄漏和不必要的内存消耗。

单订阅流是为不必丢失事件和/或流必须具有明确定义的生命周期的用例而设计的。广播流针对不一定由监听器控制的事件源以及某些事件可能丢失或被忽略的事件进行调整。

在单订阅流上使用 asBroadcastStream 可能导致资源泄漏。考虑更安全的替代方案，例如监听器交换和 StreamIterators。
