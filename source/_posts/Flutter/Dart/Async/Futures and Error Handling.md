---
layout: post
title:  "(译) Futures 和错误处理"
date:   2019-04-29 09:45:17 +0800
categories: 
   - Flutter 开发
tags:
   - Dart
   - Async
---

[原文链接](https://www.dartlang.org/guides/libraries/futures-error-handling)

本指南介绍了在编写异步代码时如何处理错误。你可以通过两种方式编写异步代码：

* 使用 `async` 函数和 `await` 表达式，其背后使用的是 Futures。
* 直接使用 [Future API](https://api.dartlang.org/stable/dart-async/dart-async-library.html)。

<!--more-->

注意：如果你不熟悉 Futures 背后的一般概念，请先阅读[异步编程：Futures](/2019/05/04/Dart/Async/Asynchrony%20support/)。

为什么要使用 Future API 而不是使用 `async` 和` await` ？当你需要比从 `async` 和 `await` 获得的还要有更多控制时。例如，假设你希望在 future 完成时发生某些事情，但你也希望立即继续你正在做的事情。如果使用 `await` ，则活动将暂停，直到 Future 完成。使用 Future API ，你可以注册回调并继续。


## Async, await 和异常

使用异常处理 -  try ，catch 和 finally - 来处理使用 async 函数和 await 表达式的代码中的错误。例如：

```dart
Future main() async {
  var dir = new Directory('/tmp');

  try {
    var dirList = dir.list();
    await for (FileSystemEntity f in dirList) {
      if (f is File) {
        print('Found file ${f.path}');
      } else if (f is Directory) {
        print('Found dir ${f.path}');
      }
    }
  } catch (e) {
    print(e.toString());
  }
}

```

本指南的其余部分解决了使用 Future API 时的处理错误。

## Future API 以及 回调

使用 Future API 的函数注册回调，用于处理完成 Future 的值(或错误)。例如：

```dart
myFunc().then(processValue)
        .catchError(handleError);
```

注册的回调基于以下规则触发：如果在以值结束的 Future 上调用 `then()` 的回调，则触发该回调;如果在一个以错误结束的 Future 上调用了 `catchError()` 的回调，则会触发它。

在上面的例子中，如果 `myFunc()` 的Future以值结束，那么()的回调将触发。如果`then()` 中没有产生新错误，则不会触发 `catchError()` 的回调。另一方面，如果`myFunc()` 以错误结束，`then()` 的回调不会触发，而 `catchError()` 的回调就会触发。

## 将 then() 和 catchError() 一起使用的例子


链接 `then()` 和 `catchError()` 调用是处理 Futures 时的常见模式，可以被认为是 try-catch 块的粗略等价物。

接下来的几节给出了这种模式的例子。

### catchError() 作为一个完整的错误处理器

以下示例处理从 `then()` 的回调中抛出异常，并演示 `catchError()` 作为错误处理程序的多功能性：

```dart
myFunc()
  .then((value) {
    doSomethingWith(value);
    ...
    throw("some arbitrary error");
  })
  .catchError(handleError);

```


如果 `myFunc()` 的Future以值结束，`then()` 的回调会触发。如果then()的回调中的代码抛出（就像在上面的例子中那样），then()的Future以错误结束。该错误由 `catchError()` 处理。

如果 `myFunc()` 的Future以错误结束，`then()` 的Future完成该错误。该错误也由 `catchError()` 处理。

无论错误是在 `myFunc()` 内还是在`then()`内，`catchError()` 都会成功处理它。

### than() 中的错误处理

对于更细粒度的错误处理，你可以在 `then()` 中注册第二个（`onError`）回调来处理已结束但有错误的 Futures。这是是 `then()` 的签名：

```dart
abstract Future then(onValue(T value), {onError(AsyncError asyncError)})
```
仅当你要区分转发到 then() 的错误和 then() 中生成的错误时，才注册可选的 onError 回调：

```dart
funcThatThrows()
  .then(successCallback, onError: (e) {
    handleError(e);          // Original error.
    anotherFuncThatThrows(); // Oops, new error.
  })
  .catchError(handleError);  // Error from within then() handled.

```

在上面的例子中，使用 `onError` 回调处理 `funcThatThrows()` 的Future错误; `anotherFuncThatThrows()` 导致then() 的Future以错误结尾;这个错误由​​`catchError()` 处理。

通常，不建议实现两种不同的错误处理策略：仅当有令人信服的理由在 then() 中捕获错误时才注册第二个回调。

### 一个长链中间的错误

通常会有一系列then()调用，并使用catchError()捕获从链的任何部分生成的错误：

```dart

Future<String> one()   => new Future.value("from one");
Future<String> two()   => new Future.error("error from two");
Future<String> three() => new Future.value("from three");
Future<String> four()  => new Future.value("from four");

void main() {
  one()                                   // Future completes with "from one".
    .then((_) => two())                   // Future completes with two()'s error.
    .then((_) => three())                 // Future completes with two()'s error.
    .then((_) => four())                  // Future completes with two()'s error.
    .then((value) => processValue(value)) // Future completes with two()'s error.
    .catchError((e) {
      print("Got error: ${e.error}");     // Finally, callback fires.
      return 42;                          // Future completes with 42.
    })
    .then((value) {
      print("The value is $value");
    });
}

// Output of this program:
//   Got error: error from two
//   The value is 42

```


在上面的代码中，one() 的Future以值结束，但是 two() 的 Future 以错误结束。当在以错误结束的 Future 上调用 then() 时，则不会触发 then() 的回调。相反，then() 的Future 以它的接收器的错误结尾。在我们的示例中，这意味着在调用 two() 之后，每个后续then() 返回的 Future 都会以 two 的错误结束。该错误最终在 catchError() 中处理。


### 处理特有的错误


如果我们想捕获特定错误怎么办？或者捕获多个错误？

catchError() 接受一个可选的命名参数test，它允许我们查询抛出的错误类型。

```dart
abstract Future catchError(onError(AsyncError asyncError), {bool test(Object error)})
```

考虑 handleAuthResponse(params），这是一个根据提供的参数对用户进行身份验证的函数，并将用户重定向到适当的URL。鉴于复杂的工作流程，handleAuthResponse()可能会生成各种错误和异常，你应该以不同的方式处理它们。以下是如何使用test来执行此操作：

```dart
void main() {
  handleAuthResponse({'username': 'johncage', 'age': 92})
    .then((_) => ...)
    .catchError(handleFormatException,
                test: (e) => e is FormatException)
    .catchError(handleAuthorizationException,
                test: (e) => e is AuthorizationException);
}
```

## 使用 whenComplete() 的 Async 的 try-catch-finally


如果 then().catchError() 对应 try-catch ，whenComplete() 相当于 'finally'。在 whenComplete()中注册的回调，是在 whenComplete() 的接收器结束时被调用的，不管它是以值还是错误结尾：

```dart
var server = connectToServer();
server.post(myUrl, fields: {"name": "john", "profession": "juggler"})
      .then(handleResponse)
      .catchError(handleError)
      .whenComplete(server.close);
```

无论 server.post() 是否产生有效响应或错误，我们都想调用 server.close 。我们通过将它放在 whenComplete() 中来确保这一点。

### 结束由 whenComplete() 返回的Future

如果没有错误从 `whenComplete()` 发出，则其 Future 将以与调用 `whenComplete() ` 的Future相同的方式完成。通过示例最容易理解。

在下面的代码中，then() 的Future以错误结束，所以 whenComplete() 的 Future 也以错误结束。

```dart
void main() {
  funcThatThrows()
    .then((_) => print("won't reach here"))    // Future completes with an error.
    .whenComplete(() => print('reaches here')) // Future completes with the same error.
    .then((_) => print("won't reach here"))    // Future completes with the same error.
    .catchError(handleError);                  // Error is handled here.
}
```

在下面的代码中，then() 的 Future 完成了一个错误，现在由 catchError() 处理。因为catchError() 的Future以 someObject 结束，所以 whenComplete() 的Future 以同一个对象结束。

```dart

void main() {
  funcThatThrows()
    .then((_) => ...)         // Future completes with an error.
    .catchError((e) {
      handleError(e);
      printErrorMessage();
      return someObject;
    })                                   // Future completes with someObject.
    .whenComplete(() => print("Done!")); // Future completes with someObject.
}
```


### 来自 whenComplete() 的错误
如果 whenComplete() 的回调引发错误，那么 whenComplete() 的Future以该错误结束：

```dart
void main() {
  funcThatThrows()
    .catchError(handleError)               // Future completes with a value.
    .whenComplete(() => throw "new error") // Future completes with an error.
    .catchError(handleError);              // Error is handled.
}
```

## 潜在问题：未能及早注册错误处理程序
在 Future 完成之前安装错误处理程序至关重要：这可以避免 Future 完成错误，错误处理程序尚未附加以及错误意外传播的情况。考虑以下代码：

```dart
void main() {
  Future future = funcThatThrows();

  // BAD. Too late to handle funcThatThrows() exception.
  new Future.delayed(const Duration(milliseconds: 500), () {
    future.then(...)
          .catchError(...);
  });
}
```


在上面的代码中，catchError() 在调用 funcThatThrows() 之后直到半秒才注册，并且错误未处理。

如果在 Future.delayed() 回调中调用 funcThatThrows() ，问题就会消失：

```dart
void main() {
  new Future.delayed(const Duration(milliseconds: 500), () {
    funcThatThrows().then(processValue)
                    .catchError(handleError)); // We get here.
  });
}
```
### 潜在问题：意外混合同步和异步错误
返回 future 的函数几乎总是会在将来发出错误。由于我们不希望这些函数的调用者必须实现多个错误处理方案，因此我们希望防止任何同步错误泄漏。考虑以下代码：

```dart
Future<int> parseAndRead(data) {
  var filename = obtainFileName(data);         // Could throw.
  File file = new File(filename);
  return file.readAsString().then((contents) {
    return parseFileData(contents);            // Could throw.
  });
}
```

该代码中的两个函数可能同步抛出：`obtainFileName()` 和`parseFileData()` 因为`parseFileData()` 在`then()` 回调中执行，所以它的错误不会从函数中泄漏出来。相反，`then()`的`futureparseFileData()`的错误结束，错误最终结束了parseAndRead() 的future，并且 `catchError()` 可以成功处理错误。

但是，在 then() 回调中不调用 obtainFileName() ;如果它抛出，则传播同步错误：

```dart
void main() {
  parseAndRead(data).catchError((e) {
    print("inside catchError");
    print(e.error);
  });
}

// Program Output:
//   Unhandled exception:
//   <error from obtainFileName>
//   ...
```

因为使用 catchError() 不捕获错误，所以 parseAndRead() 的客户端将为此错误实现单独的错误处理策略。

### 解决方案：使用Future.sync() 来包装代码


确保不会从函数中意外抛出同步错误的常见模式是将函数体包装在新的 Future.sync() 回调中：
```dart
Future<int> parseAndRead(data) {
  return new Future.sync(() {
    var filename = obtainFileName(data);         // Could throw.
    File file = new File(filename);
    return file.readAsString().then((contents) {
      return parseFileData(contents);            // Could throw.
    });
  });
}

```
如果回调返回非 Future 值，则 Future.sync() 的 Future 以该值结束。如果回调抛出（就像在上面的例子中那样），则 Future 会以错误结束。如果回调本身返回 Future，则Future 的值或错误将以 Future.sync() 的 Future 结束。

使用 Future.sync() 中包含的代码，catchError() 可以处理所有错误：

```dart
void main() {
  parseAndRead(data).catchError((e) {
    print("inside catchError");
    print(e.error);
  });
}

// Program Output:
//   inside catchError
//   <error from obtainFileName>
```
Future.sync() 使你的代码能够抵御未捕获的异常。如果你的函数中包含大量代码，那么你可能会在没有意识到的情况下做一些危险的事情：

```dart
Future fragileFunc() {
  return new Future.sync(() {
    var x = someFunc();     // Unexpectedly throws in some rare cases.
    var y = 10 / x;         // x should not equal 0.
    ...
  });
}
```

Future.sync() 不仅允许你处理你可能发生的错误，还可以防止错误意外泄漏你的功能。

## 更多信息
以下文档提供了有关 Future 的更多信息：

* [事件循环和Dart](/2019/04/29/Dart/Async/The%20Event%20Loop%20and%20Dart/)，一篇描述如何使用Futures安排任务的文章
* [Future的API参考](https://api.dartlang.org/stable/dart-async/Future-class.html)
