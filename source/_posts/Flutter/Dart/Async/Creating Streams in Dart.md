---
layout: post
title:  "(译)在Dart中创建流"
date:   2019-04-30 16:45:17 +0800
categories: 
   - Flutter 开发
tags:
   - Dart
   - Async
---

[原文链接](https://www.dartlang.org/articles/libraries/creating-streams)

由 Lasse Nielsen 撰写

2013年4月(2018年10月更新)

dart:async 库包含两种对许多 Dart API 很重要的类型： [Stream](https://api.dartlang.org/stable/dart-async/Stream-class.html) 和 [Future](https://api.dartlang.org/stable/dart-async/Future-class.html) 。如果Future表示单次计算的结果，则流是一系列结果。你在流上监听以获得结果（数据和错误）以及流关闭的通知。你也可以在监听时暂停或在流完成之前停止监听流。

<!--more-->


但是这篇文章不是关于使用流。这是关于创建自己的流。你可以通过以下几种方式创建流：

* 转换现有流。
* 使用 `async *` 函数从头开始创建流。
* 使用 `StreamController` 创建流。

本文介绍了每种方法的代码，并提供了帮助你正确实现流的提示。

有关使用流的帮助，请参阅[异步编程:Streams](/dart/async/2019/04/28/AsynchronousProgrammingStreams/)。


## 转换现有流
创建流的常见情况是你已经拥有流，并且你希望基于原始流的事件创建新流。例如，你可能有一个字节流，你希望通过 UTF-8 解码输入来转换为字符串流。最通用的方法是创建一个等待原始流上的事件然后输出新事件的新流。例：

```dart
/// Splits a stream of consecutive strings into lines.
///
/// The input string is provided in smaller chunks through
/// the `source` stream.
Stream<String> lines(Stream<String> source) async* {
  // Stores any partial line from the previous chunk.
  var partial = '';
  // Wait until a new chunk is available, then process it.
  await for (var chunk in source) {
    var lines = chunk.split('\n');
    lines[0] = partial + lines[0]; // Prepend partial line.
    partial = lines.removeLast(); // Remove new partial line.
    for (var line in lines) {
      yield line; // Add lines to output stream.
    }
  }
  // Add final partial line to output stream, if any.
  if (partial.isNotEmpty) yield partial;
}

```

对于许多常见的转换，您可以使用 Stream 提供的转换方法，例如 map() ，where() ，expand() 和 take() 。

例如，假设你有一个stream，counterStream 流，它每秒发出一个递增计数器。以下是它的实现方式：

```dart
var counterStream =
    Stream<int>.periodic(Duration(seconds: 1), (x) => x).take(15);
```

要快速查看事件，你可以使用以下代码：

```dart
counterStream.forEach(print); // Print an integer every second, 15 times.
```
要转换流事件，可以在监听流之前调用流上的转换方法(如map()。该方法返回一个新流。

```dart
// Double the integer in each event.
var doubleCounterStream = counterStream.map((int x) => x * 2);
doubleCounterStream.forEach(print);

```
你可以使用任何其他转换方法，而不是map()，例如：

```dart
.where((int x) => x.isEven) // Retain only even integer events.
.expand((var x) => [x, x]) // Duplicate each event.
.take(5) // Stop after the first five events.
```
通常，你只需要一种转换方法。但是，如果你需要更多地控制转换，可以使用 Stream 的transform() 方法指定 StreamTransformer 。平台库为许多常见任务提供流变换器。例如，以下代码使用 dart:convert 库提供的 utf8.decoder 和 LineSplitter 转换器。

```dart
Stream<List<int>> content = File('someFile.txt').openRead();
List<String> lines =
    await content.transform(utf8.decoder).transform(LineSplitter()).toList();
```

## 从头创建流

创建新流的一种方法是使用异步生成器(async \*）函数。调用函数时会创建流，并且在监听流时函数的主体开始运行。函数返回时，流关闭。在函数返回之前，它可以使用 yield 或 yield \* 语句在流上发出事件。

这是一个以固定间隔发出数字的原始示例：

```dart
Stream<int> timedCounter(Duration interval, [int maxCount]) async* {
  int i = 0;
  while (true) {
    await Future.delayed(interval);
    yield i++;
    if (i == maxCount) break;
  }
}

```
[PENDING：显示使用它的代码，所以我们有一些提及 StreamSubscription 的上下文？]

此函数返回一个 Stream 。当监听该流时，正文开始运行。它反复延迟请求的间隔，然后产生下一个数字。如果省略count参数，则循环上没有停止条件，因此流将永远输出越来越大的数字 - 或者直到监听器器取消其订阅。

当监听器取消（通过对listen（）方法返回的 StreamSubscription 对象调用cancel()）时，下一次正文到达 yield 语句时，yield 将作为 return 语句。执行任何封闭的 finally 块，并退出该函数。如果函数在退出之前尝试生成一个值，那么它将失败并充当返回值。

当函数最终退出时，cancel() 方法返回的 future 将完成。如果函数以错误退出，则将来会以该错误完成;否则,它以 null 结束。

另一个更有用的示例是将 future 序列转换为流的函数：

```dart
Stream<T> streamFromFutures<T>(Iterable<Future<T>> futures) async* {
  for (var future in futures) {
    var result = await future;
    yield result;
  }
}

```
此函数要求future可迭代新的future，等待该future，发出结果值，然后循环。如果将来因错误而完成，则流将以该错误完成。

很少有异步\*函数从无到有构建流。它需要从某个地方获取数据，而且通常在某个地方是另一个流。在某些情况下，与上面的future序列一样，数据来自其他异步事件源。但是，在许多情况下，async\*函数过于简单，无法轻松处理多个数据源。这就是 StreamController 类的用武之地。

## 使用StreamController

如果你的流的事件来自程序的不同部分，而不仅仅是来自异步函数可以遍历的流或future，那么使用 StreamController 来创建和填充流。

StreamController 为你提供了一个新的流，以及一种在任何地方和任何地方向流添加事件的方法。该流具有处理监听器和暂停所需的所有逻辑。您返回流并将控制器保留给自己。

以下示例（来自 stream_controller_bad.dart ）展示了 StreamController 的一个基本但有缺陷的用法，用于实现前面示例中的 timedCounter() 函数。此代码创建一个要返回的流，然后根据计时器事件将数据提供给它，这些事件既不是 future 也不是流事件。

```dart
// NOTE: This implementation is FLAWED!
// It starts before it has subscribers, and it doesn't implement pause.
Stream<int> timedCounter(Duration interval, [int maxCount]) {
  var controller = StreamController<int>();
  int counter = 0;
  void tick(Timer timer) {
    counter++;
    controller.add(counter); // Ask stream to send counter values as event.
    if (maxCount != null && counter >= maxCount) {
      timer.cancel();
      controller.close(); // Ask stream to shut down and tell listeners.
    }
  }

  Timer.periodic(interval, tick); // BAD: Starts before it has subscribers.
  return controller.stream;
}
```

和以前一样，你可以使用 timedCounter() 返回的流，如下所示：[PENDING：我们之前是否显示过这个？]

```dart
var counterStream = timedCounter(const Duration(seconds: 1), 15);
counterStream.listen(print); // Print an integer every second, 15 times.

```
timedCounter() 的这个实现有几个问题：

* 它在订阅者之前就开始制作活动。
* 即使订阅者请求暂停，它也会不断产生事件。

如下一节所示，您可以通过在创建StreamController时指定回调（如 onListen 和onPause ）来解决这两个问题。


### 等待订阅
通常，流应该在开始工作之前等待订阅者。 async * 函数会自动执行此操作，但在使用StreamController 时，您可以完全控制并且即使不应该也可以添加事件。当流没有订阅者时，其 StreamController 会缓冲事件，如果流永远不会获得订阅者，则会导致内存泄漏。

尝试将使用该流的代码更改为以下内容：

```dart
void listenAfterDelay() async {
  var counterStream = timedCounter(const Duration(seconds: 1), 15);
  await Future.delayed(const Duration(seconds: 5));

  // After 5 seconds, add a listener.
  await for (int n in counterStream) {
    print(n); // Print an integer every second, 15 times.
  }
}
```

当此代码运行时，前5秒没有打印任何内容，尽管流正在运行。然后添加侦听器，并且一次打印前5个左右的事件，因为它们由 StreamController 缓冲。

要获得订阅通知，请在创建 StreamController 时指定 onListen 参数。当流获得其第一个订阅者时，将调用 onListen 回调。如果指定 onCancel 回调，则在控制器丢失其最后一个订阅者时调用它。在前面的示例中，Timer.periodic() 应该移动到onListen处理程序，如下一节所示。

### 尊重暂停状态

当监听器器请求暂停时，避免产生事件。当流订阅暂停时，async\*函数会在yield语句中自动暂停。另一方面，StreamController 在暂停期间缓冲事件。如果提供事件的代码不遵循暂停，则缓冲区的大小可以无限增长。此外，如果监听器在暂停后立即停止监听，则浪费了创建缓冲区所花费的工作。

要查看没有暂停支持会发生什么，请尝试将使用该流的代码更改为以下内容：

```dart
void listenWithPause() {
  var counterStream = timedCounter(const Duration(seconds: 1), 15);
  StreamSubscription<int> subscription;

  subscription = counterStream.listen((int counter) {
    print(counter); // Print an integer every second.
    if (counter == 5) {
      // After 5 ticks, pause for five seconds, then resume.
      subscription.pause(Future.delayed(const Duration(seconds: 5)));
    }
  });
}

```


当五秒钟暂停时，在此期间触发的事件都会立即收到。发生这种情况是因为流的源不支持暂停并且不断向流添加事件。因此，流缓冲事件，然后在流取消暂停时清空其缓冲区。

以下版本的 timedCounter()（来自stream_controller.dart）通过使用StreamController 上的 onListen，onPause ，onResume 和 onCancel 回调来实现暂停。


```dart
Stream<int> timedCounter(Duration interval, [int maxCount]) {
  StreamController<int> controller;
  Timer timer;
  int counter = 0;

  void tick(_) {
    counter++;
    controller.add(counter); // Ask stream to send counter values as event.
    if (counter == maxCount) {
      timer.cancel();
      controller.close(); // Ask stream to shut down and tell listeners.
    }
  }

  void startTimer() {
    timer = Timer.periodic(interval, tick);
  }

  void stopTimer() {
    if (timer != null) {
      timer.cancel();
      timer = null;
    }
  }

  controller = StreamController<int>(
      onListen: startTimer,
      onPause: stopTimer,
      onResume: startTimer,
      onCancel: stopTimer);

  return controller.stream;
}

```


使用上面的 listenWithPause() 函数运行此代码。你会看到它在暂停时停止计数，然后它恢复得很好。

必须使用所有监听器- onListen，onCancel，onPause 和 onResume-来通知暂停状态的更改。原因是如果订阅和暂停状态同时发生变化，则仅调用 onListen 或 onCancel 回调。


## 最后的提示
在不使用async \*函数的情况下创建流时，请记住以下提示：

* 使用同步控制器时要小心 - 例如，使用 StreamController 创建的同步控制器（sync：true）。在未调用的同步控制器上发送事件时（例如，使用由 EventSink 定义的 add()，addError()或close()方法），事件将立即发送到流上的所有监听器。在添加监听器的代码完全返回之前，永远不能调用流监听器，并且在错误的时间使用同步控制器可能会破坏此承诺并导致良好的代码失败。避免使用同步控制器。

* 如果使用 StreamController ，则在 listen 调用返回 StreamSubscription 之前调用 onListen 回调。不要让 onListen 回调依赖于已存在的订阅。例如，在以下代码中，在订阅变量具有有效值之前触发 onListen 事件（并调用处理程序）。

```dart
subscription = stream.listen（handler）;
```

* StreamController 定义的 onListen，onPause ，onResume 和 onCancel 回调在流的监听器状态更改时由流调用，但从不在事件触发期间或在调用另一个状态更改处理程序期间调用。在这些情况下，状态更改回调会延迟，直到上一次回调完成。

* 不要尝试自己实现 Stream 接口。很容易获得事件，回调之间的交互，以及添加和删除侦监听的巧妙错误。始终使用可能来自 StreamController 的现有流来实现新流的监听调用。

* 虽然可以通过扩展 Stream 类并在顶部实现 listen 方法和额外功能来创建扩展 Stream功能和更多功能的类，但通常不建议这样做，因为它引入了用户必须考虑的新类型。相反，你通常可以创建一个具有 Stream（和更多）的类 - 而不是一个Stream（以及更多）。