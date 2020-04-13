---
layout: post
title:  "(译)Dart语言异步支持：阶段一"
date:   2019-05-04 09:55:17 +0800
categories: 
   - Flutter 开发
tags:
   - Dart
   - Async
---

[原文链接](https://www.dartlang.org/articles/language/await-async)

由Gilad Bracha撰写

2014年10月

Dart即将推出支持异步编程的新语言功能。这些功能将逐渐上线。在本文中，我们将讨论最基本的补充：`await` 表达式和 `async` 方法。这些是与异步相关的最常用功能。

<!--more-->


>注意：本文假设你已熟悉Dart中的异步编程。

> 版本说明：在 Dart 1.x 中，异步函数立即暂停执行。在 Dart 2 中，异步函数不是立即挂起，而是同步执行，直到第一个 `await` 或 `return` 出现才会暂停执行。

## Async 函数

异步函数是主体标有异步修饰符的函数。

```dart
foo() async => 42;
```

当你调用异步函数时，它会立即返回一个Future;该函数的主体被安排稍后执行。当主体执行时，调用返回的Future将使用结果完成 - 无论主体是否成功运行，或引发异常。在提供的简单示例中，调用 foo()会产生 Future。Future 最终以数字42完成。

你可以在没有async修饰符的情况下编写类似的函数：

```dart
foo() => new Future(() => 42);
```

修饰符为你节省了一些样板，但真正的一点是，它允许你在函数内部使用 `await` 表达式，我们很快就会看到。稍后，我们将返回异步函数以更全面地理解它们。


## Await 表达式


Await 表达式允许你编写异步代码，就像它是同步的一样。假设你有一个引用文件的变量myFile。（有关文件的详细信息，请参阅dart：io中的File类。）你决定将其复制到新位置newPath，声明为

```dart
String newPath = '/some/where/out/there';
```

你希望以下是true的：

```dart
myFile.copy(newPath).path == newPath;
```

不幸的是，这不会奏效。由于Dart的 I/O API是异步的，因此复制操作返回 Future ，并且你无法在其上调用路径。你必须在从 copy() 返回的 Future 上安排回调，并且该回调执行与其传入参数f的比较：

```dart
myFile.copy(newPath).then((f) => f.path == newPath);
```
这有点乏味，但是你的代码涉及的越多，情况就越糟糕。你真正想要做的是等待异步文件复制操作完成，获取结果并继续执行。 await 表达式可以让你做到这一点：

```dart
(await myFile.copy(newPath)).path == newPath;
```
当 **await** 表达式运行时，调用 myFile.copy()，产生一个 Future。执行然后暂停，等待 Future 完成。在 Future 完成文件后，执行恢复。await 表达式的值是 Future 的完成 - 我们正在等待的文件。现在我们可以提取它的路径并将其与 newPath 进行比较。

通常，await表达式具有以下形式：

```dart
await e
```

其中  e是一元表达式。通常，e 是异步计算，并且期望评估为 Future。 await 表达式计算 e，然后挂起当前运行的函数，直到结果准备好 - 也就是说，直到 Future 完成。await 表达式的结果是 Future 的完成。

>注意：暂停后，执行将在事件循环的后续周期中恢复。有关Dart事件循环的说明，请参阅[事件循环和 Dart ](/dart/async/2019/04/29/The-Event-Loop-and-Dart/)。

如果 Future 使用错误而不是值来完成，则 await 表达式会在执行恢复时抛出相同的错误，这极大地简化了异步代码中异常的处理。

如果 e 没有被评估成 Future 怎么办？好吧，无论如何 **await** 会等待（技术上，它将结果包装在 Future 中并等待它在事件循环周期中完成）。这是Dart与其他语言中类似功能之间的差异之一。在 Dart 中，**await** 总是等待。这使得行为更具可预测性。特别是，如果你有一个无条件等待内部的循环，你总是可以确保你将在每次迭代时暂停。

如果 e 本身会引发异常怎么办(请注意，这与使用错误完成对  Future的求值不同。）抛出的异常包含在 Future 中，执行挂起。当我们恢复时，抛出异常。同样，暂停是可预测的。

最后一个但是很关键的点：你只能在异步函数中使用 **await** 表达式。如果你尝试在普通函数中使用 **await** ，则会出现编译错误。如果你要暂停一个普通的函数，它将不再是同步的。

## Async 函数：难懂的条文

现在我们已经了解了 **await** 表达式是如何工作的，让我们重新审视异步函数，以便我们清楚一些重要的细节。

首先，注意修饰符介于函数签名和它的主体之间。我们也可以将foo()写成

```dart
foo() async { return 42; }
```

简而言之，修饰符位于=>或打开函数体的大括号之前。

修饰符不是签名的一部分;它只是函数的实现细节。从调用者的角度来看，调用异步函数与调用传统函数没什么不同。

由于同样的原因，**async** 修饰符对函数的声明返回类型也没有影响。但是，它确实会改变实际返回的对象类型。请注意，return 语句返回一个整数，但该函数已经将 Future 返回给它的调用者！在异步函数内部，return 语句的操作方式与常规函数不同。在异步函数中，return 完成函数在调用时返回给调用者的 Future。 Future 将使用返回的表达式的值完成。

同样，如果在异步函数中抛出(或重新抛出)异常，则抛出的对象将用于完成 Future 并出现错误。

如果返回的表达式具有类型 T，则该函数应具有返回类型 Future\<T>（或其超类型）。否则，发出静态警告。我们的示例不声明返回类型，因此它们具有返回类型动态 - 因此不会给出警告。

如果 return 语句中的表达式是 Future\<T>，则函数返回类型应保留为 Future\<T> 而不是 Future\<Future<T>>。除了等待更多之外，对于已经完成另一个 Future 的 Future ，你可以做的事情并不多，因此异步库消除了 Futures 层。类型准则旨在识别这一事实。

最后，请注意 Dart 中的异步函数始终是异步的。这与其他语言中的异步函数不同，在某些情况下，函数可能完全同步。在 Dart 中，你知道异步函数的每个部分在调用它的调用返回给调用者之后执行。

## 把所有东西放在一起

这是一个结合我们迄今为止所学到的内容的例子。假设我们正在运行一个简单的动画，它会更新每一帧的显示。

不使用 `async` 和 `await` ，代码可能如下所示：

```dart
import "dart:html";

main() {
  var context = querySelector("canvas").context2D;
  var running = true;    // Set false to stop.

  tick(time) {
    context.clearRect(0, 0, 500, 500);
    context.fillRect(time % 450, 20, 50, 50);

    if (running) window.animationFrame.then(tick);
  }

  window.animationFrame.then(tick);
}
```
它不是太复杂，但也不是很简单。我们生成帧;当帧完成时，我们期望调用一个回调函数tick()，它产生下一帧（如果动画还没有停止),并递归自身作为回调，使进程永久化。函数tick()表示计算的延续，我们都知道延续是多么直观和容易。

使用我们的新语言功能，我们可以编写以下代码：

```dart
import "dart:html";

main() async {
  var context = querySelector("canvas").context2D;
  var running = true;    // Set false to stop game.

  while (running) {
    var time = await window.animationFrame;
    context.clearRect(0, 0, 500, 500);
    context.fillRect(time % 450, 20, 50, 50);
  }
}
```

这里的代码是自我解释的。在动画运行时，我们计算一个帧。这是你的选择;选择你觉得比较容易理解的版本。

## 更多信息

有关更高级异步主题的信息，例如async\*，sync\*，yield和yield\*，请参阅[Dart语言异步支持：阶段二](/dart/async/2019/05/04/Dart-Language-Asynchrony-SupportPhase2/)。

