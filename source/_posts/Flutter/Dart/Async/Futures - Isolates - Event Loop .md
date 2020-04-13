---
layout: post
title:  "(译) Futures - Isolates - Event Loop"
date:   2019-05-12 08:30:17 +0800
categories: 
   - Flutter 开发
tags:
   - Dart
   - Async
---

[原文链接](https://www.didierboelens.com/2019/01/futures---isolates---event-loop/)

单线程，多线程，同步和异步。本文介绍了 Flutter 中的不同代码执行模式。

难度：中级

<!--more-->


## 介绍

我最近收到了一些与 `Future` ，`async`，`await`，`Isolate` 和并行处理概念相关的问题。

除了这些问题，一些人在处理代码序列的顺序执行的时候遭遇到了问题。

我认为利用一篇文章的机会来弄清楚这些概念是非常有用的，文章主要是围绕异步和并行处理的概念展开的。

## Dart是一种单线程语言

首先，每个人都需要记住，Dart 是单线程，并且 Flutter 依赖于  Dart。

> 重要

> Dart一次执行一个操作，一个接一个地，意味着只要一个操作正在执行，它就不会被任何其他Dart代码中断。

换句话说，如果你考虑一个纯粹的同步方法，后者将是唯一一个在完成之前执行的方法。

```dart
void myBigLoop(){
    for (int i = 0; i < 1000000; i++){
        _doSomethingSynchronously();
    }
}
```

在上面的示例中，myBigLoop() 方法的执行在完成之前永远不会被中断。因此，如果此方法需要一些时间，则在整个方法执行期间将“阻塞”应用程序。

## Dart 执行模型

表象背后，Dart 实际上是如何管理要执行的操作序列的？

为了回答这个问题，我们需要看看 Dart 代码定序器，被称为`事件循环`。

当你启动 Flutter (或任何 Dart)应用程序时，将创建并启动新的 Thread 进程（在Dart语言中等价于“Isolate”）。该线程将是整个应用程序中你唯一需要关注的线程。

因此，创建此线程时，Dart 自动

1. 初始化2个队列，即 “**MicroTask**” 和 “**Event**” FIFO队列;
2. 执行`main()`方法，并且一旦代码执行完成，
3. 就启动 `Event Loop`

在线程的整个生命周期中，单个内部和不可见的进程（被称为“事件循环”）将驱动代码按顺序排列执行，具体取决于微任务队列和事件队列的内容。

事件循环对应于某种无限循环，节奏由内部时钟控制，在每次滴答时，如果没有其他 Dart 代码被执行，则执行如下操作：

```dart
void eventLoop(){
    while (microTaskQueue.isNotEmpty){
        fetchFirstMicroTaskFromQueue();
        executeThisMicroTask();
        return;
    }

    if (eventQueue.isNotEmpty){
        fetchFirstEventFromQueue();
        executeThisEventRelatedCode();
    }
}
```

正如我们所看到的，微任务队列优先于事件队列，但这2个队列的用途是什么？

### MicroTask Queue

微任务队列用于非常短的需要异步运行的内部操作，就在其他操作完成后，并且在交还给事件队列之前。

作为微任务的一个例子，你可以想象必须在资源关闭后立即处置它。由于关闭过程可能需要一些时间才能完成，你可以编写如下内容：

```dart
MyResource myResource;

    ...

    void closeAndRelease() {
        scheduleMicroTask(_dispose);
        _close();
    }

    void _close(){
        // The code to be run synchronously
        // to close the resource
        ...
    }

    void _dispose(){
        // The code which has to be run
        // right after the _close()
        // has completed
    }
```
这是大多数时候你不必使用的东西。例如，整个 Flutter 源代码仅引​​用scheduleMicroTask()方法7次。

所以，最好考虑使用事件队列。

### Event Queue

事件队列用于引用由此产生的操作

* 外部事件如
	* I/O;
	* 手势;
	* 绘制;
	* 计时器;
	* 流;
	*  ...
* futures

实际上，每次触发外部事件时，要执行的相应代码都会被引用到事件队列中。

一旦不再运行任何微任务，事件循环将考虑事件队列中的第一项并执行它。

值得注意的是，Futures 也通过事件队列处理。

### Futures

Future 对应于一个异步运行的任务，并且这个任务会在将来某个时间点完成(或失败)。

当你实例化一个新的 Future 时：

* 该 Future 的一个实例被创建并记录在由 Dart 管理的内部数组中;
* 需要由此 Future 执行的代码直接推送到事件队列中;
* future 的实例返回状态（=未完成）;
* 如果有的话，执行下一个同步代码（而不是 Future 的代码）

只要事件循环从事件队列中获取它，Future 将引用的代码将像任何其他Event一样执行。

当该代码将被执行并将完成(或失败)时，将直接执行 then() 或 catchError()。

为了说明这一点,我们来看下面的例子：

```dart
void main(){
    print('Before the Future');
    Future((){
        print('Running the Future');
    }).then((_){
        print('Future is complete');
    });
    print('After the Future');
}
```
如果我们运行此代码，输出将如下：

```
Before the Future
After the Future
Running the Future
Future is complete
```

这完全正常，因为执行流程如下：

* 打印（'Before the Future'）
* 将"(）{print（'Running the Future'）;}"添加到事件队列中;
* 打印（'After the Future'）
* 事件循环获取代码（在 bullet2 中引用）并运行它
* 当代码执行时，它会查找 then() 语句并运行它

要记住一些非常重要的事情：

> Future 不是并行执行的，而是遵循事件循环处理的常规事件序列

### Async 方法

当你使用 `async` 关键字为方法声明添加后缀时，Dart知道：

* 该方法的结果是 Future ;
* 它同步运行该方法的代码直到第一个`await`关键字，然后它暂停执行该方法的其余部分;
* 一旦由 `await` 关键字引用的Future将完成，下一行代码将立即运行。

理解这一点非常重要，因为许多开发人员，认为 `await` 会暂停执行整个流程，直到它完成。但事实并非如此。他们忘记了事件循环的工作方式。

为了更好地说明这个语句，让我们采取以下示例，让我们试着找出其执行的结果。

```dart
void main() async {
  methodA();
  await methodB();
  await methodC('main');
  methodD();
}

methodA(){
  print('A');
}

methodB() async {
  print('B start');
  await methodC('B');
  print('B end');
}

methodC(String from) async {
  print('C start from $from');
  
  Future((){                // <== This code will be executed some time in the future
    print('C running Future from $from');
  }).then((_){
    print('C end of Future from $from');
  });

  print('C end from $from');  
}

methodD(){
  print('D');
}
```
正确的顺序如下：

1. A
2. B start
3. C start from B
4. C end from B
5. B end
6. C start from main
7. C end from main
8. D
9. C running Future from B
10. C end of Future from B
11. C running Future from main
12. C end of Future from main

现在，让我们考虑上面代码中的 “methodC()” 对应于对服务器的调用，这可能需要不均匀的时间来响应。我相信很明显地说，预测确切的执行流程可能变得非常困难。

如果您对示例代码的初始期望是仅在所有内容的末尾执行 methodD()，那么你应该这么编写代码，如下所示：

```dart
void main() async {
  methodA();
  await methodB();
  await methodC('main');
  methodD();  
}

methodA(){
  print('A');
}

methodB() async {
  print('B start');
  await methodC('B');
  print('B end');
}

methodC(String from) async {
  print('C start from $from');

  await Future((){                  // <== modification is here
    print('C running Future from $from');
  }).then((_){
    print('C end of Future from $from');
  });
  print('C end from $from');  
}

methodD(){
  print('D');
}
```

这给出了以下顺序：

1. A
2. B start
3. C start from B
4. C running Future from B
5. C end of Future from B
6. C end from B
7. B end
8. C start from main
9. C running Future from main
10. C end of Future from main
11. C end from main
12. D


在 methodC() 中在 Future 级别添加简单等待的事实会改变整个行为。

同样非常重要的是要记住：

> 异步方法不是并行执行，而是遵循事件循环处理的常规事件序列。

我想告诉你的最后一个例子如下。
运行 method1 和 method2 的输出是什么？它们会一样吗？

```dart
void method1(){
  List<String> myArray = <String>['a','b','c'];
  print('before loop');
  myArray.forEach((String value) async {
    await delayedPrint(value);
  });  
  print('end of loop');
}

void method2() async {
  List<String> myArray = <String>['a','b','c'];
  print('before loop');
  for(int i=0; i<myArray.length; i++) {
    await delayedPrint(myArray[i]);
  }
  print('end of loop');
}

Future<void> delayedPrint(String value) async {
  await Future.delayed(Duration(seconds: 1));
  print('delayedPrint: $value');
}
```

Answer:

|method1()|	method2()|
|---|---|
|before loop|before loop|
|end of loop|delayedPrint: a (after 1 second)|
|delayedPrint: a (after 1 second)|delayedPrint: b (1 second later)|
|delayedPrint: b (directly after)|delayedPrint: c (1 second later)|
|delayedPrint: c (directly after)|end of loop (right after)|

你是否看到了他们的行为不一样的区别和原因？

解决方案在于，method1 使用函数 forEach() 来迭代数组。每次迭代时，它都会调用一个新的回调，它被标记为异步（因此是一个 Future )。它执行它直到它到达等待然后它将剩余的代码推送到事件队列。一旦迭代完成，它就会执行下一个语句 “print（'loop of loop'）”。完成后，事件循环将处理3个回调。

关于方法2，一切都在相同的代码“块”内运行，因此一行一行地运行（在这个例子中）。

正如你所看到的，即使在看起来非常简单的代码中，我们仍然需要记住事件循环的工作方式。

## 多线程

因此，我们如何在 Flutter 中运行并行代码？这可能吗？

是的，多亏了[Isolates](https://api.dartlang.org/stable/2.1.0/dart-isolate/Isolate-class.html)的概念。

### 什么是 Isolate?
如前所述，Isolate 对应于 Thread 概念的 Dart 版本。

然而，与“线程”的通常实现存在重大差异，这就是它们被命名为“隔离”的原因。

> Flutter中的 “Isolates” 不共享内存。不同 “Isolate” 之间的交互是通过“消息”在通信方面进行的。

### 每个Isolate都有自己的“事件循环”

每个“Isolate”都有自己的“事件循环”和队列（微任务和事件）。这意味着代码在Isolate内运行时，与另一个Isolate无关。

多亏了这一点，我们可以获得并行处理。

### 如何启动隔离？
根据你运行 Isolate 的需要，你可能需要考虑不同的方法。

#### 1.低级解决方案
第一个解决方案不使用任何软件包，完全依赖 Dart 提供的低级 API。

#### 1.1。第1步：创造和握手
正如我之前所说，Isolates不共享任何内存并通过消息进行通信，因此，我们需要找到一种方法来在“调用者”和新 Isolate 之间建立这种通信。

每个 Isolate 都公开一个端口，用于将消息传递给该 Isolate 。这个端口叫做“SendPort”（我个人觉得这个名字有点误导，因为它是一个旨在接收/收听的端口，但这是官方名称）。

这意味着“调用者”和“新隔离者”都需要知道彼此的端口才能进行通信。这个握手的过程如下所示：


```dart
//
// The port of the new isolate
// this port will be used to further
// send messages to that isolate
//
SendPort newIsolateSendPort;

//
// Instance of the new Isolate
//
Isolate newIsolate;

//
// Method that launches a new isolate
// and proceeds with the initial
// hand-shaking
//
void callerCreateIsolate() async {
    //
    // Local and temporary ReceivePort to retrieve
    // the new isolate's SendPort
    //
    ReceivePort receivePort = ReceivePort();

    //
    // Instantiate the new isolate
    //
    newIsolate = await Isolate.spawn(
        callbackFunction,
        receivePort.sendPort,
    );

    //
    // Retrieve the port to be used for further
    // communication
    //
    newIsolateSendPort = await receivePort.first;
}

//
// The entry point of the new isolate
//
static void callbackFunction(SendPort callerSendPort){
    //
    // Instantiate a SendPort to receive message
    // from the caller
    //
    ReceivePort newIsolateReceivePort = ReceivePort();

    //
    // Provide the caller with the reference of THIS isolate's SendPort
    //
    callerSendPort.send(newIsolateReceivePort.sendPort);

    //
    // Further processing
    //
}
```
> 约束
隔离的“入口点”必须是顶级函数或STATIC方法。


#### 1.2。第2步：向隔离区提交消息

现在我们有了用于向 Isolate 发送消息的端口，让我们看看如何做到这一点：

```dart
//
// Method that sends a message to the new isolate
// and receives an answer
// 
// In this example, I consider that the communication
// operates with Strings (sent and received data)
//
Future<String> sendReceive(String messageToBeSent) async {
    //
    // We create a temporary port to receive the answer
    //
    ReceivePort port = ReceivePort();

    //
    // We send the message to the Isolate, and also
    // tell the isolate which port to use to provide
    // any answer
    //
    newIsolateSendPort.send(
        CrossIsolatesMessage<String>(
            sender: port.sendPort,
            message: messageToBeSent,
        )
    );

    //
    // Wait for the answer and return it
    //
    return port.first;
}

//
// Extension of the callback function to process incoming messages
//
static void callbackFunction(SendPort callerSendPort){
    //
    // Instantiate a SendPort to receive message
    // from the caller
    //
    ReceivePort newIsolateReceivePort = ReceivePort();

    //
    // Provide the caller with the reference of THIS isolate's SendPort
    //
    callerSendPort.send(newIsolateReceivePort.sendPort);

    //
    // Isolate main routine that listens to incoming messages,
    // processes it and provides an answer
    //
    newIsolateReceivePort.listen((dynamic message){
        CrossIsolatesMessage incomingMessage = message as CrossIsolatesMessage;

        //
        // Process the message
        //
        String newMessage = "complemented string " + incomingMessage.message;

        //
        // Sends the outcome of the processing
        //
        incomingMessage.sender.send(newMessage);
    });
}

//
// Helper class
//
class CrossIsolatesMessage<T> {
    final SendPort sender;
    final T message;

    CrossIsolatesMessage({
        @required this.sender,
        this.message,
    });
}
```


##### 1.3. Step 3: destruction of the new Isolate

当你不再需要新的Isolate实例时，最好通过以下方式释放它：


```dart
//
// Routine to dispose an isolate
//
void dispose(){
    newIsolate?.kill(priority: Isolate.immediate);
    newIsolate = null;
}
```


##### 1.4. 特别说明 - 单侦听器流
您可能已经注意到我们正在使用Streams在“调用者”和新隔离之间进行通信。这些Streams的类型为：“Single-Listener”Streams。

### 2.一次性计算

如果你只需要运行一些代码来完成一些特定的工作，并且在完成工作后不需要与那个Isolate交互，那么就会有一个非常方便的Helper，称为[compute](https://docs.flutter.io/flutter/foundation/compute.html)。

这个功能：

* 产生隔离，
* 在该隔离上运行一个回调函数，传递一些数据，
* 返回值，回调结果，
* 并在执行回调时终止Isolate。

> 约束
“回调”函数必须是顶级函数，不能是闭包或类的方法（静态或非静态）


### 3.重要限制

在撰写本文时，请务必注意这一点

> 平台通道仅由主隔离支持。此主隔离对应于启动应用程序时创建的隔离。

换句话说，通过编程创建的隔离实例，无法实现平台通道通信...

然而，有一个解决方法...请参阅[此链接](https://github.com/flutter/flutter/issues/13937)以获得有关此主题的讨论。

### 我什么时候应该使用 Futures 和 Isolates ？

用户将根据不同因素评估应用程序的质量，例如：

* 特征
* 外观
* 用户友好性
* ...

你的应用程序可以满足所有这些因素，但如果用户在某些处理过程中遇到滞后，则很可能会对您不利。

因此，以下是你在开发过程中应系统考虑的一些提示：

1. 如果代码片段不能被中断，请使用正常的同步过程（一种方法或多种相互调用的方法）;
2. 如果代码片段可以独立运行而不影响应用程序的流动性，请考虑通过使用Futures使用Event Loop;
3. 如果繁重的处理可能需要一些时间才能完成并且可能会影响应用程序的流动性，请考虑使用Isolates。

换句话说，建议尽可能多地使用 Futures 的概念（直接或间接通过异步方法），因为一旦事件循环有时间了，这些 Futures 的代码就会运行。这将使用户感觉事物正在被并行处理（而我们现在知道情况并非如此）。

另一个可以帮助你决定是使用 Future 还是 Isolate 的因素是运行某些代码所需的平均时间。



* 如果一个方法需要几毫秒 => Future
* 如果处理可能需要几百毫秒 => Isolate

以下是Isolates的一些很好的候选者：

* JSON解码

解码JSON（HttpRequest的结果）可能需要一些时间=>使用 compute

* 加密

加密可能非常消耗=> Isolate

* 图像处理

处理图像（例如，裁剪）确实需要一些时间来完成=> Isolate

* 从Web加载图像

在这种情况下，为什么不将它委托给一个 Isolate ，它会在完全加载后返回完整的图像？

## 结论

我认为了解事件循环的工作原理至关重要。

同样重要的是要记住 Flutter(Dart）是单线程因此，为了取悦用户，开发人员必须确保应用程序尽可能顺利运行。Futures 和 Isolates 是非常强大的工具，可以帮助你实现这一目标。

请继续关注新文章，同时......祝你编程愉快！
