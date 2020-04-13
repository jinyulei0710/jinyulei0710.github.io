---
layout: post
title:  "(译)异步编程: futures 和 async-await "
date:   2019-04-28 10:00:17 +0800
categories: 
   - Flutter 开发
tags:
   - Dart
   - Async
---
[原文链接](https://dart.dev/tutorials/language/futures)

> 关键点是什么？

>* Dart 代码在单线程的执行区块上运行
* 阻塞线程的代码能够让你的程序卡死。
* `Future` 对象代表的是异步操作的结果-稍后要完成的处理或者 I/O 。
* 要实现暂停执行直到一个 future 完成了，在 async 函数中使用 `await` 
* 要捕捉异常，在异步函数中使用 try-catch 表达式
* 要并发执行代码，就创建一个 *isolate* 。(或者对于 web 应用来说，创建一个工作线程)

<!--more-->

Dart 代码在单线程的执行区块上运行。如果 Dart 代码阻塞了，例如，执行一个耗时的计算或者等待 I/O ，整个程序就会卡死。

异步操作让你的代码完成其它工作的同时等待一个操作完成。Dart 使用 `Future` 来表示异步操作的结果。要使用 future ，你可以使用 `async` 和 `await` ，或者 `Future` API。

> 注意：所有 Dart 代码都在一个 *isolate* 的上下文中运行，该 *isolate* 拥有Dart代码使用的所有内存。当 Dart 代码正在执行时，同一个 isolate 中的其他代码都无法运行。


> 如果你希望多个部分的 Dart 代码并发运行，你可以在各自的 isolate 中运行它们。 （ Web 应用程序使用工作线程而不是 isolate。）多个 isolate 同时运行的时候，通常每个isolate都在其自己的CPU核心上运行。*isolate* 之间不共享内存，它们可以进行交互的唯一函数是通过相互发送消息。有关更多信息，请参阅 [isolates](https://api.dartlang.org/stable/2.3.1/dart-isolate/dart-isolate-library.html) 或 [ Web 工作线程](https://api.dartlang.org/stable/2.3.1/dart-html/Worker-class.html) 的文档。

## 介绍

让我们看一些可能导致程序卡死的代码：

```dart
// 同步代码
void printDailyNewsDigest() {
  var newsDigest = gatherNewsReports(); // 可能需要一段时间。
  print(newsDigest);
}

main() {
  printDailyNewsDigest();
  printWinningLotteryNumbers();
  printWeatherForecast();
  printBaseballScore();
}
```

代码收集了今天的新闻，并将之打印出来，然后打印其他一堆用户感兴趣的条目：

```
<gathered news goes here>
Winning lotto numbers: [23, 63, 87, 26, 2]
Tomorrow's forecast: 70F, sunny.
Baseball score: Red Sox 10, Yankees 0
```

我们的代码是有问题的：因为 `gatherNewsReports()` 阻塞了线程，剩余的代码只有在`gatherNewsReports()` 返回文件内容之后才能运行，不管这个过程需要多长时间。如果读取文件花费了很长时间，用户就需要等待，然后就开始胡思乱想。

为了帮助应用能及时响应，Dart 库的作者在定义函数的时候使用了一个异步模型来做潜在的繁重工作。譬如使用 future 返回它们的值的函数。

## 什么是 future ？

future 是一个 [Future\<T>](https://api.dartlang.org/stable/dart-async/Future-class.html) 对象，用来表示异步操作产生的结果 `T`。如果结果不是一个可用的值，future的类型就是 `Future<void>` 。当函数返回的 future 被调用了，有两件事发生了：

1. 函数把需要做的工作放入队列然后返回一个未完成的 `Future` 对象。
2. 稍后，当操作完成了，`Future` 对象就以一个值或一个错误完成了。

当写依赖于 future 的代码的时候，你有两个选择

* 使用 `async` 和 `await`
* 使用 `Future` API

## Async 和 await

`async` 和 `await` 关键字是 Dart 语言[异步支持](https://dart.dev/guides/language/language-tour#asynchrony-support)的一部分。它们允许你像写同步代码一样写异步代码，并且不用使用 `Future` API。async 函数的 `async` 关键字是在它的 body 之前的。`await` 关键字只有在异步函数中才能使用。

>版本说明：在 Dart 1.x 中，异步函数立即暂停执行。在 Dart 2中，异步函数不是立即挂起，而是同步执行，直到第一个 `await` 或 `return`出现。

以下的 app 通过使用 `async` 和 `await` 去读取这个网站上的文件内容来模拟读取新闻。

```dart
import 'dart:async';

Future<void> printDailyNewsDigest() async {
  var newsDigest = await gatherNewsReports();
  print(newsDigest);
}

main() {
  printDailyNewsDigest();
  printWinningLotteryNumbers();
  printWeatherForecast();
  printBaseballScore();
}
printWinningLotteryNumbers() {
  print('Winning lotto numbers: [23, 63, 87, 26, 2]');
}

printWeatherForecast() {
  print("Tomorrow's forecast: 70F, sunny.");
}

printBaseballScore() {
  print('Baseball score: Red Sox 10, Yankees 0');
}

const news = '<gathered news goes here>';
const oneSecond = Duration(seconds: 1);

// 想象这个函数是更为复杂和慢的. :)
Future<String> gatherNewsReports() =>
    Future.delayed(oneSecond, () => news);

// 作为一种选择, 你可以使用 dart:io 或 dart:html来从服务器获取新闻
// 例如：
//
// import 'dart:html';
//
// Future<String> gatherNewsReportsFromServer() => HttpRequest.getString(
//      'https://www.dartlang.org/f/dailyNewsDigest.txt',
//    );

```

```
Winning lotto numbers: [23, 63, 87, 26, 2]
Tomorrow's forecast: 70F, sunny.
Baseball score: Red Sox 10, Yankees 0
<gathered news goes here>
```

注意下 `printDetailDigest()` 是首个被调用的函数，但是新闻却是最后被打印出来的，即使这个文件只包含了单行的信息。这是因为读取和打印文件的的代码是异步运行的。

在此例中，`printDetailNewsDigest()` 函数调用了`gatherNewReports()`，它是非阻塞的。调用`  gatherNewsReports()` 把要做的工作放入队列但是不阻止剩余的代码执行。程序打印出了，乐透号码、天气预报、以及棒球的比分； `当gatherNewsReports()` 结束收集新闻的时候，程序把它打印出来。如果读取 `gatherNewsReports()` 花费了一小会的时间来完成它的工作，并不会造成什么伤害，在每天新闻打印之前用户可以看其它的东西。

注意下返回类型，`gatherNewsReports()` 的返回类型是 `Future<String>` ,这意味着它是以 string 类型值完成的 future 。`printDailyNewsDigest()` 函数，不会返回一个值，它的返回类型是 `Future<Void>`。

以下的示意图展示了代码中的执行流程。每个编码对应一个步骤。

![img](https://www.dartlang.org/tutorials/images/async-await.png)

1. 应用开始执行
2. `main()` 函数调用 async 函数 `printDailyNewsDigest()` ,它就开始同步执行。
3. `printDailyNewsDigest()` 使用 await 来调用函数 `gatherNewsReports()` ,`gatherNewsReports()` 这个函数就开始执行。
4. `gatherNewsReports()` 函数返回一个未完成的 future（一个 Future<String> 的实例）。
5. 因为 `printDailyNewsDigest()` 是一个异步的函数并且在等待一个值，它暂停了它的执行并返回一个未完成的 future .(在此例中，一个 `Future<void>` 被返回到了它的调用者 `main()`)。
6. 剩余的打印函数执行。因为它们是同步的，每个函数在移动到下一个打印函数之前已经执行完全了。例如，中奖的乐透号码是在天气预报打印之前的。
7. 当 `main()` 函数执行完成，异步函数就能恢复执行了。首先，由 `gatherNewReports()` 返回的 future 完成了。然后 `printDailyNewsDigest()` 继续执行，打印出新闻。
8. 当 `printDailyNewsDigest()` 函数体完成了执行，这个最初返回的 future 完成了，然后应用就退出了。


注意下，async 函数开始是直接执行的(同步的)。但它遇到下面任何一种情形的时候，函数暂停了执行并返回了一个未完成的 future：

* 函数的首个 await 表达式（在表达式中获取未完成的 future 之后）。
* 函数中任意的 return 声明
* 函数体的结束

### 错误处理

如果返回类型为 Future 的函数以错误结尾的时候，你可能想要捕捉这个错误。异步函数可以使用 try-catch 处理错误：

```dart
Future<void> printDailyNewsDigest() async {
  try {
    var newsDigest = await gatherNewsReports();
    print(newsDigest);
  } catch (e) {
    // Handle error...
  }
}
```

try-catch 代码与异步代码的行为方式与同步代码的行为方式相同：如果 `try` 块中的代码抛出异常，则 `catch` 子句中的代码将执行。

### 顺序处理

你可以使用多个 `await` 表达式来确保每个语句在执行下一个语句之前完成：

```dart
// 使用 async 和 await 的线性处理.
main() async {
  await expensiveA();
  await expensiveB();
  doSomethingWith(await expensiveC());
}
```

在 `expensiveA()` 完成之前，`expensiveB()` 函数不会执行，依此类推。

## 其他资源

阅读以下文档，了解有关在 Dart 中使用 future 和异步编程的更多详细信息：

* [异步支持](/2019/05/04/Dart/Async/Asynchrony%20support/)，[语言导览](https://www.dartlang.org/guides/language/language-tour)中的一个部分。
* [futures](https://api.dartlang.org/stable/dart-async/Future-class.html)，[isolates](https://api.dartlang.org/stable/dart-isolate/dart-isolate-library.html) 和 [web 工作线程](https://api.dartlang.org/stable/dart-html/Worker-class.html) 的 API 参考文档。

## 接下来是什么？
下一个教程，[异步编程：Streams](/2019/04/28/Dart/Async/AsynchronousProgrammingStreams/)，向你展示如何使用事件流。