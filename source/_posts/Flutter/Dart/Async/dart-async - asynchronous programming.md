---
layout: post
title:  "(译)dart:async - 异步编程"
date:   2019-05-04 08:30:17 +0800
categories: 
   - Flutter 开发
tags:
   - Dart
   - Async
---

[原文链接](https://www.dartlang.org/guides/libraries/library-tour#dartasync---asynchronous-programming)

异步编程通常使用回调函数，但 Dart 提供了备选：[Future](https://api.dartlang.org/stable/dart-async/Future-class.html) 和 [Stream](https://api.dartlang.org/stable/dart-async/Stream-class.html) 对象。Future 就像是对将来某个时候提供结果的承诺。Stream 是一种获取值序列的方法，例如事件。 Future，Stream 等都在dart:async库([API参考](https://api.dartlang.org/stable/dart-async/dart-async-library.html)）中。

<!--more-->


>注意：你并不总是需要直接使用 Future 或 Stream API。 Dart 语言支持使用 `async` 和 `await` 等关键字进行异步编码。有关详细信息，请参阅语言导览中的[异步支持](/dart/async/2019/05/04/Asynchrony-support/)。

dart:async 库适用于 Web 应用程序和命令行应用程序。要使用它，请导入 dart:async ：

```dart
import 'dart:async';
```

>版本说明：从 Dart 2.1 开始，你无需导入 dart:async 即可使用 Future 和 Stream API，因为 dart:core 会导出这些类。


## Future

Future 对象出现在 Dart 库中，通常作为异步方法返回的对象。当 future 完成后，其值即可使用。

### 使用 await

在直接使用Future API之前，请考虑使用 `await` 。使用 `await` 表达式的代码比使用 Future API的代码更容易理解。

考虑以下功能。它使用 Future 的 then() 方法连续执行三个异步函数，在执行下一个函数之前要等待每个函数。

```dart
runUsingFuture() {
  // ...
  findEntryPoint().then((entryPoint) {
    return runExecutable(entryPoint, args);
  }).then(flushThenExit);
}
```

具有await表达式的等效代码看起来更像是同步代码：

```dart
runUsingAsyncAwait() async {
  // ...
  var entryPoint = await findEntryPoint();
  var exitCode = await runExecutable(entryPoint, args);
  await flushThenExit(exitCode);
}
```


异步函数可以捕获Futures的异常。例如：

```dart
var entryPoint = await findEntryPoint();
try {
  var exitCode = await runExecutable(entryPoint, args);
  await flushThenExit(exitCode);
} catch (e) {
  // Handle the error...
}
```

> 重要提示：异步函数返回 Futures。如果你不希望函数返回 Future，请使用其他解决方案。例如，你可以从函数中调用异步函数。

有关使用`await`和相关Dart的语言功能的更多信息，请参阅[异步支持](/dart/async/2019/05/04/Asynchrony-support/)。

### 基本使用

你可以使用 `then()` 来安排将来完成时运行的代码。例如，`HttpRequest.getString()` 返回一个 Future，因为HTTP请求可能需要一段时间。使用 `then()` 可以在 Future 完成并且承诺的字符串值可用时运行一些代码：

```dart
HttpRequest.getString(url).then((String result) {
  print(result);
});
```
使用`catchError()`来处理Future对象可能抛出的任何错误或异常。

```dart
HttpRequest.getString(url).then((String result) {
  print(result);
}).catchError((e) {
  // Handle or ignore the error.
});
```

then().catchError() 模式是 try-catch 的异步版本。

重要说明：确保对 `then()` 的结果调用 `catchError()` - 而不是原始`Future`的结果。否则，`catchError()` 只能处理原始 Future 的计算中的错误，而不能处理`then()`注册的处理程序中的错误。

### 链接多个异步函数

then() 方法返回一个 Future，提供了一种以特定顺序运行多个异步函数的有用方法。如果使用then() 注册的回调返回 Future，then() 返回等效的 Future。如果回调返回任何其他类型的值，then() 创建一个使用该值完成的新Future。

```dart
Future result = costlyQuery(url);
result
    .then((value) => expensiveWork(value))
    .then((_) => lengthyComputation())
    .then((_) => print('Done!'))
    .catchError((exception) {
  /* Handle exception... */
});
```

在前面的示例中，方法按以下顺序运行：

1. costlyQuery()
2. expensiveWork()
3. lengthyComputation()

这是使用await编写的相同代码：

```dart
try {
  final value = await costlyQuery(url);
  await expensiveWork(value);
  await lengthyComputation();
  print('Done!');
} catch (e) {
  /* Handle exception... */
}
```
### 等待多个feature

有时您的算法需要调用许多异步函数并等待它们全部完成才能继续。使用[Future.wait()](https://api.dartlang.org/stable/dart-async/Future/wait.html)静态方法管理多个Futures并等待它们完成：

```dart
Future deleteLotsOfFiles() async =>  ...
Future copyLotsOfFiles() async =>  ...
Future checksumLotsOfOtherFiles() async =>  ...

await Future.wait([
  deleteLotsOfFiles(),
  copyLotsOfFiles(),
  checksumLotsOfOtherFiles(),
]);
print('Done with all the long steps!');
```

## Stream 

流对象出现在整个 Dart API 中，表示数据序列。例如，按钮点击等HTML事件是使用流传递的。你还可以将文件作为流读取。

### 使用异步for循环
有时你可以使用异步for循环(`await for`)而不是使用 Stream API。

考虑以下功能。它使用Stream的 `listen()` 方法订阅文件列表，传入一个搜索每个文件或目录的函数文字。

```dart
void main(List<String> arguments) {
  // ...
  FileSystemEntity.isDirectory(searchPath).then((isDir) {
    if (isDir) {
      final startingDir = Directory(searchPath);
      startingDir
          .list(
              recursive: argResults[recursive],
              followLinks: argResults[followLinks])
          .listen((entity) {
        if (entity is File) {
          searchFile(entity, searchTerms);
        }
      });
    } else {
      searchFile(File(searchPath), searchTerms);
    }
  });
}
```

带有await表达式的等效代码，包括异步for循环（`await for`），看起来更像是同步代码：

```dart
Future main(List<String> arguments) async {
  // ...
  if (await FileSystemEntity.isDirectory(searchPath)) {
    final startingDir = Directory(searchPath);
    await for (var entity in startingDir.list(
        recursive: argResults[recursive],
        followLinks: argResults[followLinks])) {
      if (entity is File) {
        searchFile(entity, searchTerms);
      }
    }
  } else {
    searchFile(File(searchPath), searchTerms);
  }
}
```

>重要提示：在使用 `await for` 之前，请确保它使代码更清晰，并且你确实希望等待所有流的结果。例如，你通常不应该使用等待DOM事件监听器，因为DOM会发送无穷无尽的事件流。如果你使用 `await for` 来连续注册两个DOM事件监听器，则永远不会处理第二种事件。


有关使用`await`和相关Dart语言功能的更多信息，请参阅[异步支持](/dart/async/2019/05/04/Asynchrony-support/)。

## 监听流数据

要在到达时获取每个值，请使用`await for`或使用`listen()`方法订阅流：

```dart
// Find a button by ID and add an event handler.
querySelector('#submitInfo').onClick.listen((e) {
  // When the button is clicked, it runs this code.
  submitData();
});
```

在此示例中，`onClick`属性是“submitInfo”按钮提供的Stream对象。

如果你只关心一个事件，则可以使用诸如 `first` ，`last` 或 `single` 之类的属性来获取它。要在处理事件之前测试事件，请使用 `firstWhere()` ，`lastWhere()` 或 `singleWhere()` 等方法。

如果你关心事件的子集，则可以使用诸如 `skip()` ，`skipWhile()` ，`take()`，`takeWhile()` 和 `where()` 之类的方法。


### 转换流数据

通常，您需要先更改流数据的格式，然后才能使用它。使用 `transform()` 方法生成具有不同类型数据的流：

```dart
var lines = inputStream
    .transform（utf8.decoder）
    .transform（LineSplitter（））;
```

此示例使用两个变换器。首先，它使用 utf8.decoder 将整数流转换为字符串流。然后，它使用一个 LineSplitter将字符串流转换为单独的行流。这些变换器来自dart:convert库（参见[dart:convert部分](https://www.dartlang.org/guides/libraries/library-tour#dartconvert---decoding-and-encoding-json-utf-8-and-more)）。

### 处理错误和完成

如何指定错误和完成处理代码取决于你是使用异步 for 循环(`await for`)还是Stream API。

如果使用异步 for 循环，则使用 try-catch 来处理错误。关闭流后执行的代码将在异步for循环之后执行。

```dart
Future readFileAwaitFor() async {
  var config = File('config.txt');
  Stream<List<int>> inputStream = config.openRead();

  var lines = inputStream
      .transform(utf8.decoder)
      .transform(LineSplitter());
  try {
    await for (var line in lines) {
      print('Got ${line.length} characters from stream');
    }
    print('file is now closed');
  } catch (e) {
    print(e);
  }
}
```

如果使用Stream API，则通过注册 `onError` 监听器来处理错误。通过注册 `onDone` 监听器关闭流后运行代码。

```dart
var config = File('config.txt');
Stream<List<int>> inputStream = config.openRead();

inputStream
    .transform(utf8.decoder)
    .transform(LineSplitter())
    .listen((String line) {
  print('Got ${line.length} characters from stream');
}, onDone: () {
  print('file is now closed');
}, onError: (e) {
  print(e);
});
```

更多信息
有关在命令行应用程序中使用 Future 和 Stream 的一些示例，请参阅[dart:io导览](https://www.dartlang.org/server/io-library-tour)。另请参阅这些文章和教程：

* [异步编程: Futures](/dart/async/2019/04/28/AsynchronousProgrammingFutures/)
* [Futures 和错误处理](/dart/async/2019/04/29/Futures-and-Error-Handling/)
* [事件循环和 Dart](/dart/async/2019/04/29/The-Event-Loop-and-Dart/)
* [异步编程：Streams](/dart/async/2019/04/28/AsynchronousProgrammingStreams/)
* [在 Dart 中创建流](/dart/async/2019/04/30/Creating-Streams-in-Dart/)