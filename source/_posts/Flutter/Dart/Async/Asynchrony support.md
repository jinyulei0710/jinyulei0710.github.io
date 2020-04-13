---
layout: post
title:  "(译)异步支持"
date:   2019-05-04 07:45:17 +0800
categories: 
   - Flutter 开发
tags:
   - Dart
   - Async
---

[原文链接](https://www.dartlang.org/guides/language/language-tour#asynchrony-support)

Dart 库充满了返回 [Future](https://api.dartlang.org/stable/dart-async/Future-class.html) 或 [Stream](https://api.dartlang.org/stable/dart-async/Stream-class.html) 对象 的 函数。这些函数是异步的：它们在设置可能耗时的操作（例如 I/O )后返回，而不等待该操作完成。

`async` 和 `await` 关键字支持异步编程，允许你编写看起来类似于同步代码的异步代码。

<!--more-->


## 处理 Futures

当你需要完成的 Future 的结果时，你有两个选择：

* 使用 `async` 和 `await`
* 使用 Future API , 如[库导览](https://www.dartlang.org/guides/libraries/library-tour#future)中所述

使用 `async` 和 `await` 的代码是异步的，但它看起来很像同步代码。例如，这里有一些代码使用 `await` 来等待异步函数的结果：

```dart
await lookUpVersion();
```
要使用 `await` ，代码必须位于异步函数中 - 标记为 async 的函数：

```dart
Future checkVersion() async {
  var version = await lookUpVersion();
  // Do something with version
}
```

>注意：虽然异步函数可能会执行耗时的操作，但它不会等待这些操作。相反，异步函数只在遇到第一个 `await` 表达式（[详细信息](https://github.com/dart-lang/sdk/blob/master/docs/newsletter/20170915.md#synchronous-async-start)）时才会执行。然后它返回一个 Future 对象，仅在 `await` 表达式完成后才恢复执行。


使用try，catch，最后在使用await的代码中处理错误和清理：

```dart
try {
  version = await lookUpVersion();
} catch (e) {
  // React to inability to look up the version
}
```

你可以在异步函数中多次使用 `await`。例如，以下代码等待了函数的结果三次：

```dart
var entrypoint = await findEntrypoint();
var exitCode = await runExecutable(entrypoint,args);
await flushThenExit(exitCode)'

```

在 `await` 表达式中，表达式的值通常是 Future ;如果不是，那么该值将自动包装在Future中。此 Future 对象表示的是返回一个对象的承诺。 await 表达式的值是返回的对象。 await 表达式使执行暂停，直到该对象可用。

如果在使用 `await` 时遇到编译时错误，请确保 `await` 在异步函数中。例如，要在应用程序的 `main()` 函数中使用 `await` ，`main()` 的主体必须标记为 `async`：

```dart
Future main() async {
  checkVersion();
  print('In main: version is ${await lookUpVersion()}');
}
```

## 声明异步函数

异步函数是一个函数，其主体用async修饰符标记。

将 `async` 关键字添加到函数使其返回一个 Future 。例如，考虑这个同步函数，它返回的是一个 String：

```dart
String lookUpVersion() => '1.0.0';
```

如果将其更改为异步函数 - 例如，因为将来的实现将非常耗时 - 返回的值是一个 Future：

```dart
Future<String> lookUpVersion() async => '1.0.0';
```

请注意，函数的主体不需要使用Future API。如有必要，Dart会创建 `Future` 对象。

如果您的函数没有返回有用的值，请使其返回类型为 `Future<void>`。

## 处理 Streams


当你需要从 Stream 获取值时，你有两个选择：

* 使用 `async` 和一个异步for循环(await for)。
* 使用Stream API，如[库导览](https://www.dartlang.org/guides/libraries/library-tour#stream)中所述。


> 注意：在使用 `await for` 之前，请确保它使代码更清晰了，并且你确实希望等待所有流的结果。例如，你通常不应该针对UI事件监听器使用 `await for` ，因为UI框架会发送无穷无尽的事件流。

异步for循环具有以下形式：

```dart
await for (varOrType identifier in expression) {
  // Executes each time the stream emits a value.
}
```

表达式的值必须具有类型化的Stream。执行过程如下：

1. 等到流发出一个值。
2. 执行for循环的主体，将变量设置为该发出的值。
3. 重复1和2，直到关闭流。


要停止监听流，可以使用 `break` 或 `return` 语句，该语句会中断 for 循环并从流中取消取消。

如果在实现异步for循环时遇到编译时错误，请确保 `await for` 是异步函数。例如，要在 app 的`main()` 函数中使用异步for循环，`main()`的主体必须标记为 `async`：

```dart
Future main() async {
  // ...
  await for (var request in requestServer) {
    handleRequest(request);
  }
  // ...
}
```
有关异步编程的更多信息，请参阅库导览的[dart:async](/dart/async/2019/05/04/dart-async-asynchronous-programming/)部分。另请参阅文章[Dart语言异步支持：阶段一](/dart/async/2019/05/04/Dart-Language-Asynchrony-Support-Phase-1/)和[Dart语言异步支持：阶段二](/dart/async/2019/05/04/Dart-Language-Asynchrony-SupportPhase2/)和[Dart语言规范](https://www.dartlang.org/guides/language/spec)。
