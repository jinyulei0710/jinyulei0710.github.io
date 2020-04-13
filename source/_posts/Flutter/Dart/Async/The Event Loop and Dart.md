---
layout: post
title:  "(译) Event Loop 和 Dart"
date:   2019-04-29 10:45:17 +0800
categories: 
   - Flutter 开发
tags:
   - Dart
   - Async
---

[原文链接](https://webdev.dartlang.org/articles/performance/event-loop#annotated-sample-and-output)


由 Kathy Walrath 撰写

2013年9月(2013年10月更新）

Dart中的异步代码无处不在。许多库函数返回 Future 对象，你可以注册处理程序以响应鼠标单击，文件 I/O 完成和计时器到期等事件。

本文介绍了 Dart 的事件循环体系结构，以便你可以编写更好的异步代码，减少意外。你将学习调度 future 任务的选项，并且你将能够预知执行的顺序。

<!--more-->


> 注意：本文中的所有内容都适用于原生运行的 Dart 应用程序（使用Dart VM）和已编译为JavaScript 的 Dart 应用程序（ dart2js 的输出结果）。本文使用术语Dart来区分 Dart 应用程序和用其他语言编写的软件。


在阅读本文之前，你应该熟悉[ futures 和错误处理](/2019/04/29/Dart/Async/Futures%20and%20Error%20Handling/)的基础知识。

## 基本概念
如果你编写过 UI 代码，那么你可能熟悉事件循环和事件队列的概念。它们确保图形操作和鼠标点击等事件能一次被处理。

### 事件循环和队列
事件循环的作用是从事件队列中获取一个条目并处理它，只要队列中有条目，就重复这两个步骤。

![](event_queue1.webp)

队列中的项可能表示用户输入，文件 I/O 通知，计时器等。例如，这是包含计时器和用户输入事件的事件队列的图片：

![](event_queue2.webp)

所有这些可能可能跟你熟悉的非 Dart 语言是类似的。现在让我们来谈谈它如何融入 Dart 平台。

### Dart 的单线程执行

一旦 Dart 函数开始执行，它将继续执行直到它退出。换句话说，Dart 函数不能被其他 Dart 代码中断。

>注意：Dart 命令行应用程序可以通过创建隔离来并行运行代码。(Dart Web应用程序目前无法创建其他隔离区，但它们可以创建子线程。）隔离区不共享内存;它们就像是通过传递消息相互通信的独立应用程序。除了应用程序明确在其他隔离区或工作程序中运行的代码之外，所有应用程序的代码都在应用程序的主隔离区中运行。有关详细信息，请参阅本文后面的“[在必要的时候使用隔离或子线程](https://webdev.dartlang.org/articles/performance/event-loop#use-isolates-or-workers-if-necessary)”。

如下图所示，Dart 应用程序在其主隔离执行应用程序的 `main()` 函数时开始执行。`main()` 退出后，主隔离的线程开始逐个处理应用程序事件队列中的任何项目。

![](event-loop-and-main.webp)

实际上，这有点过度简化。


### Dart 的事件循环和队列

Dart 应用程序具有单个事件循环，其中包含两个队列 - 事件队列和微任务队列。

事件队列包含所有外部事件：I/O，鼠标事件，绘图事件，计时器，Dart隔离之间的消息等。

微任务队列是必要的，因为事件处理代码有时需要稍后完成任务，但在将控制权返回给事件循环之前。例如，当可观察对象发生更改时，它会将多个突变更改组合在一起并以异步方式报告它们。微任务队列允许可观察对象在 DOM 显示不一致状态之前报告这些突变变化。

事件队列包含来自 Dart 和系统中其他位置的事件。目前，微任务队列仅包含源自 Dart 代码的条目，但我们希望 Web 实现插入浏览器微任务队列。(有关最新状态，请参阅dartbug.com/13433。）

如下图所示，当 main() 退出时，事件循环开始工作。首先，它按 FIFO 顺序执行任何微任务。然后它出列并处理事件队列中的第一个项目。然后它重复循环：执行所有微任务，然后处理事件队列中的下一个项目。一旦两个队列都为空并且不再需要更多事件，应用程序的嵌入器（例如浏览器或测试框架）就可以处置该应用程序。

注意：如果 Web 应用程序的用户关闭其窗口，则 Web 应用程序可能会在其事件队列为空之前退出。

![](both-queues.webp)

> 重要提示：当事件循环从微任务队列执行任务时，事件队列被卡住：应用程序无法绘制图形，处理鼠标点击，对 I/O 做出反应等等。

虽然你可以预知任务执行的顺序，但你无法准确预测事件循环何时将任务从队列中删除。 Dart 事件处理系统基于单线程循环;它不是基于刻度或任何其他类型的时间测量。例如，在创建延迟任务时，事件会在你指定时排队。但是，直到处理事件队列中的所有内容（以及微任务队列中的每个任务）之前，才能处理该事件。


## 提示：链接的 futures 指定任务顺序

如果你的代码具有依赖关系，请将它们显式化。显式依赖关系有助于其他开发人员理解您的代码，并使你的程序更能抵抗代码重构。

以下是错误编码方式的示例：

```dart
// BAD because of no explicit dependency between setting and using
// the variable.
future.then(...set an important variable...);
Timer.run(() {...use the important variable...});
```
相反，编写如下代码：

```dart
// BETTER because the dependency is explicit.
future.then(...set an important variable...)
  .then((_) {...use the important variable...});

```
更好的代码使用 then() 来指定必须先设置变量才能使用它。(如果希望代码执行即使发生错误，也可以使用 whenComplete() 而不是 then()。）

如果使用变量需要花费时间并且可以在以后完成，请考虑将该代码放在新的 Future 中：

```dart
// MAYBE EVEN BETTER: Explicit dependency plus delayed execution.
future.then(...set an important variable...)
  .then((_) {new Future(() {...use the important variable...})});

```
使用新的 Future 为事件循环提供了处理事件队列中其他事件的机会。下一节详细介绍了稍后运行的调度代码。

### 如何调度任务
当你需要指定稍后要执行的代码时，可以使用 dart:async 库提供的以下API：

1. **Future** 类，它将一个项添加到**事件队列**的末尾。
2. 顶级 **scheduleMicrotask()** 函数，它将项添加到**微任务队列**的末尾。

> 注意：`scheduleMicrotask()` 函数曾被命名为 `runAsync()` 。(见公告。）

使用这些API的示例位于`Event queue:new Future()` 和`Microtask queue:scheduleMicrotask()`下的下一节中。

### 使用适当的队列（通常是：事件队列）

尽可能使用 Future 在事件队列上调度任务。使用事件队列有助于缩短微任务队列的速度，从而降低微任务队列使事件队列匮乏的可能性。

如果在处理事件队列中的任何项之前绝对必须完成任务，那么通常应该立即执行该函数。如果不能，则使用 scheduleMicrotask() 将项添加到微任务队列。例如，在Web应用程序中使用微任务来避免过早释放js-interop代理或结束IndexedDB事务或事件处理程序。

![](scheduling-tasks.webp)


### 事件队列：新的future()

要在事件队列上计划任务，请使用`new Future()`或`new Future.delayed()`。这些是dart:async 库中定义的两个 Future 构造函数。

> 注意：你也可以使用 Timer 来安排任务，但如果任务中发生任何未捕获的异常，你的应用程序将退出。相反，我们推荐 Future ，它建立在 Timer 之上，并添加了检测任务完成和响应错误等功能。

要立即将项目放在事件队列中，请使用`new Future()`:

```dart
// Adds a task to the event queue.
new Future(() {
  // ...code goes here...
});
```

你可以添加对 `then()` 或 `whenComplete()` 的调用，以在新 Future 完成后立即执行某些代码。例如，以下代码在新 Future 的任务出列时打印“42”：

```dart
new Future(() => 21)
    .then((v) => v*2)
    .then((v) => print(v));
```

要在一段时间后将条目排入队列，请使用`new Future.delayed()`:

```dart
// After a one-second delay, adds a task to the event queue.
new Future.delayed(const Duration(seconds:1), () {
  // ...code goes here...
});
```

虽然前面的示例在一秒钟之后将任务添加到事件队列，但是在主隔离空闲，微任务队列为空以及事件队列中先前排队的条目消失之前，该任务无法执行。例如，如果 main() 函数或事件处理程序正在运行昂贵的计算，则该任务在该计算完成之后才能执行。在这种情况下，延迟可能远远超过一秒。

> 提示：如果您在Web应用程序中为动画绘制帧，请不要使用 Future（或 Timer 或 Stream ）。相反，使用 [animationFrame](https://api.dartlang.org/stable/dart-html/Window/animationFrame.html) ，它是 requestAnimationFrame 的 Dart接口。

关于future的有趣事实：

1. 传递给 Future 的 **then()** 方法的函数在 Future 完成时立即执行。(该函数未入队，只是被调用。）
2. 如果在调用 **then()** 之前已经完成了Future，那么就会在微任务队列中添加一个任务，该任务执行传递给 **then()** 函数。
3. **Future()** 和 **Future.delayed()** 构造函数不会立即完成;他们将一个条目添加到事件队列中。
4. **Future.value()** 构造函数在微任务中完成，类​​似于＃2。
5. **Future.sync()** 构造函数立即执行其函数参数（除非该函数返回Future）在微任务中完成，类​​似于＃2。

### 微任务队列：scheduleMicrotask（）

dart:async 库将 scheduleMicrotask() 定义为顶级函数。你可以像这样调用scheduleMicrotask() ：

```dart
scheduleMicrotask(() {
  // ...code goes here...
});
```

由于错误[9001](https://github.com/dart-lang/sdk/issues/9001)和[9002](https://github.com/dart-lang/sdk/issues/9002)，第一次调用 scheduleMicrotask() 会在事件队列上调度任务;此任务创建微任务队列并将指定的函数排入 scheduleMicrotask() 。只要微任务队列至少有一个条目，对 scheduleMicrotask() 的后续调用就会正确地添加到微任务队列中。一旦微任务队列为空，就必须在下次调用 scheduleMicrotask() 时再次创建它。

这些错误的结果：你使用 scheduleMicrotask() 调度的第一个任务看起来就像是在事件队列中。

解决方法是在第一次调用 new Future() 之前先调用 scheduleMicrotask() 。这会在事件队列上执行其他任务之前创建微任务队列。但是，它不会阻止将外部事件添加到事件队列中。当你有一个延迟的任务时它也没有帮助。

将任务添加到微任务队列的另一种方法是在已经完成的 Future 上调用 then()。有关更多信息，请参阅上一节。

### 必要时使用隔离器或工作线程

如果要运行计算密集型任务，该怎么办？为了使您的应用程序保持响应，你应该将任务放入其自己的隔离或工作线程。隔离可能在单独的进程或线程中运行，具体取决于 Dart 实现。在1.0中，我们不希望Web应用程序支持隔离区或Dart语言工作者。但是，您可以使用dart: html Worker类将JavaScript工作线程添加到 Dart Web应用程序。

你应该使用多少个隔离？对于计算密集型任务，通常应该使用尽可能多的隔离区来提供可用的CPU。如果它们纯粹是计算的话，任何额外的隔离都会被浪费掉。但是，如果隔离区执行异步调用 - 例如执行 I/O  - 那么它们将不会在CPU上花费太多时间，因此拥有比CPU更多的隔离区是有意义的。

如果这是一个适合您的应用程序的良好架构，你还可以使用比CPU更多的隔离。例如，你可以为每个功能使用单独的隔离，或者在需要确保不共享数据时使用。

## 测试你的理解
现在你已经阅读了有关计划任务的所有内容，让我们测试你的理解。

请记住，您不应该依赖 Dart 的事件队列实现来指定任务顺序。实现可能会改变，Future 的 then() 和whenComplete() 方法是更好的选择。如果你能正确回答这些问题，你会不会觉得自己很聪明？

问题＃1
这个样本打印出来的是什么？

```dart
import 'dart:async';
void main() {
  print('main #1 of 2');
  scheduleMicrotask(() => print('microtask #1 of 2'));

  new Future.delayed(new Duration(seconds:1),
                     () => print('future #1 (delayed)'));
  new Future(() => print('future #2 of 3'));
  new Future(() => print('future #3 of 3'));

  scheduleMicrotask(() => print('microtask #2 of 2'));

  print('main #2 of 2');
}

```
答案是:

```
main #1 of 2
main #2 of 2
microtask #1 of 2
microtask #2 of 2
future #2 of 3
future #3 of 3
future #1 (delayed)
```
该顺序应该是你所期望的，因为示例的代码分三批执行：

1. main() 函数中的代码
2. 微任务队列中的任务（ scheduleMicrotask() ）
3. 事件队列中的任务（ new Future() 或 new Future.delayed() ）

请记住，main() 函数中的所有调用都是同步执行，从头到尾完成。首先 main() 调用 print() ，然后调用 scheduleMicrotask() ，然后调用 new Future.delayed()，然后调用 new Future()，依此类推。只有回调 - 指定为 scheduleMicrotask() ，new Future.delayed() 和 new Future() 的参数的闭包体中的代码 - 稍后执行。

注意：目前，如果您注释掉第一次调用scheduleMicrotask()，那么futures＃2和＃3的回调将在微任务＃2之前执行。这是由于错误9001和9002，如Microtask queue：scheduleMicrotask（）中所讨论的。

问题2
这是一个更复杂的例子。如果你能正确预知此代码的输出，你将获得一颗金星。

```dart
import 'dart:async';
void main() {
  print('main #1 of 2');
  scheduleMicrotask(() => print('microtask #1 of 3'));

  new Future.delayed(new Duration(seconds:1),
      () => print('future #1 (delayed)'));

  new Future(() => print('future #2 of 4'))
      .then((_) => print('future #2a'))
      .then((_) {
        print('future #2b');
        scheduleMicrotask(() => print('microtask #0 (from future #2b)'));
      })
      .then((_) => print('future #2c'));

  scheduleMicrotask(() => print('microtask #2 of 3'));

  new Future(() => print('future #3 of 4'))
      .then((_) => new Future(
                   () => print('future #3a (a new future)')))
      .then((_) => print('future #3b'));

  new Future(() => print('future #4 of 4'));
  scheduleMicrotask(() => print('microtask #3 of 3'));
  print('main #2 of 2');
}
```

输出，假设错误9001/9002未修复：


```dart
main #1 of 2
main #2 of 2
microtask #1 of 3
microtask #2 of 3
microtask #3 of 3
future #2 of 4
future #2a
future #2b
future #2c
future #3 of 4
future #4 of 4
microtask #0 (from future #2b)
future #3a (a new future)
future #3b
future #1 (delayed)

```

注意：由于错误9001/9002，微任务＃0在未来＃4之后执行;它应该在未来＃3之前执行。这个错误出现了，因为在未来＃2b执行时，没有微任务排队，因此微任务＃0会在事件队列上产生一个新任务，从而创建一个新的微任务队列。该微任务队列包含微任务＃0。如果你注释掉微任务＃1，那么微任务就会在未来＃2c和未来＃3之前出现在一起。


像以前一样，执行main()函数，然后执行微任务队列上的所有操作，然后执行事件队列上的任务。以下是一些有趣的观点：

* 当 future() 回调为将来3调用 new Future() 时，它会创建一个新任务（＃3a），它被添加到事件队列的末尾。
* 所有 then() 回调在完成后调用 Future 时立即执行。因此，在控制返回到嵌入器之前，将来的2,2a，2b和2c一次执行。同样，未来的3a和3b将一次性执行。
* 如果你将3a代码从那时（（_）=>新的Future（...））更改为（（_）{new Future（...）;}），那么“future＃3b”就会更早出现（之后）未来＃3，而不是未来＃3a）。原因是从回调中返回Future是你如何得到 then()（它本身返回一个新的Future）将这两个Futures链接在一起，这样当回调返回的Future完成后，then() 返回的Future完成。有关更多信息，请参阅 [then() 参考](https://api.dartlang.org/stable/dart-async/Future/then.html)。

带注释的例子和输出
以下是一些可能澄清问题＃2答案的数字。首先，这是带注释的程序源：

![](https://webdev.dartlang.org/articles/performance/images/test-annotated.png)

假设没有外部事件进入，这就是各个时间点的队列和输出的样子：

![](https://webdev.dartlang.org/articles/performance/images/test-queue-output.png)

## 总结
你现在应该了解 Dart 的事件循环以及如何安排任务。以下是Dart中事件循环的一些主要概念：

* Dart 应用程序的事件循环从两个队列执行任务：事件队列和微任务队列。
* 事件队列包含来自Dart（future，计时器，隔离消息等）和系统（用户操作，I/O 等）的条目。
* 目前，微任务队列只有来自 Dart 的条目，但我们希望它与浏览器微任务队列合并。
* 事件循环在出列并处理事件队列中的下一个项之前清空微任务队列。
* 一旦两个队列都为空，应用程序已完成其工作并且（取决于其嵌入器）可以退出。
* main()函数以及微任务和事件队列中的所有项都在 Dart 应用程序的主隔离上运行。

安排任务时，请遵循以下规则：

* 如果可能，将其放在事件队列中（使用新的 Future() 或新的 Future.delayed() ）。
* 使用 Future 的 then() 或 whenComplete() 方法指定任务顺序。
* 为避免使事件循环匮乏，请将微任务队列保持尽可能短。
* 要使应用程序保持响应，请在任一事件循环中避免计算密集型任务。
* 要执行计算密集型任务，请创建其他隔离区或工作线程。
