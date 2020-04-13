---
layout: post
title:  "(译) Flutter中的响应式编程(Reactive Programming)、流(Streams)以及业务逻辑组件(BloC)"
date:   2019-04-13 11:51:17 +0800
categories: 
   - Flutter 开发
tags:
    - 状态管理
    - BLoC
keywords:
    - BloC
---
[原文链接](https://www.didierboelens.com/2018/08/reactive-programming---streams---bloc/)

本文以理论结合实例的方式介绍了流(Streams)，业务逻辑组件(BloC)以及响应式编程(Reactive Programming)这些概念的介绍。

难度：中级

<!--more-->

## 介绍

我花费很长时间才找到介绍响应式编程(Reactive Programming)，业务逻辑组件(BloC)和流(Streams)这些概念的方法。

因为这些概念是能使得应用架构的方法产生巨大改变的，所以我想要用一个实例来展示这一点：

* 不是一定要使用它们，但有时不使用的代价可能是代码编写难度更大和性能更低，
* 使用它们的好处，以及
* 使用它们造成的影响(正面的以及负面的)

我写的这个实用性的例子是一个伪应用，简而言之，它允许用户从一个在线目录查看电影列表，对电影按类型和发布日期进行过滤，对电影进行收藏和取消收藏。当然，一切都是交互式的，用户动作可以在不同的页面中或在同一个页面内发生，并对界面进行实时刷新。

这是一个显示此应用程序的动画。

![](https://www.didierboelens.com/images/streams_app_1.gif)

既然你来看这篇文章，想要获取有关响应式编程(Reactive Programming)，业务逻辑组件(BloC)和流(Streams)的信息，我将先对它们进行一一介绍。随后，再向你展示如何在实践中实现和使用它们。

本文的补充文章中给出一些实例，可以点击[此链接](/2019/04/13/Flutter/development/data-and-backend/state-mgmt/options/BloC/ReactiveProgramming-Streams-BLoC-PracticalUseCases/)找到。

___

## 什么是流

### 介绍

为了轻松可视化流(Stream)的概念，可以简单地把它当成两头都通的水管，只允许从一头插入一些东西。当你将一些东西插入水管中时，它会在水管内流动，并从另一头流出。

在 Flutter 中，

* 管道被称为 [Stream](https://api.dartlang.org/stable/2.0.0/dart-async/Stream-class.html)
* 为了控制 Stream，我们通常(*)使用 [StreamController](https://api.dartlang.org/stable/2.0.0/dart-async/StreamController-class.html)
* 为了插入一些东西到 Stream 中去，StreamController 暴露了一个叫做 StreamSink的入口，可以通过 **sink** 属性访问
* StreamController 通过 **stream** 属性暴露了从 Stream 中出来的方式

（*）：我特意使用“通常”这个词，是因为很可能不使用任何 StreamController。但是，在本文中我将只使用 StreamController。

### 什么能被流运送？

一切都能被运送。从值，事件，对象，集合，映射，错误或甚至另一个流，任何类型的数据都能被流运送。

### 我怎么知道东西是被流运送的？
当你需要在东西被运送的时候接收到通知，你只需要**监听** StreamController 的stream 属性。

每当你定义一个监听器，你就会收到一个 [StreamSubscription](https://api.dartlang.org/stable/2.0.0/dart-async/StreamSubscription-class.html) 对象。正是通过 StreamSubscription 对象，你才会在 Stream 级别收到某些事情发生的通知。

只要有一个监听器处于**活跃**状态，Stream 就会开始生成**事件**，以便在下列情况下每次都通知活跃的 StreamSubscription 对象们。

* 一些数据从流中出来了
* 当一些错误被发送到流时，
* 当流关闭时。

StreamSubscription 对象同样也允许你：

* 停止监听
* 暂停，
* 恢复

### 流只是一条简单的水管吗？

不，Stream 还允许数据在流出之前，Stream 对其进行处理。

为了控制 Stream 内部数据的处理，我们使用 [StreamTransformer](https://api.dartlang.org/stable/2.0.0/dart-async/StreamTransformer-class.html)，它只是

* 一个“捕获” Stream 内部流动数据的方法
* 对数据做了一些事情
* 这种转换的结果也是一个 Stream


你能直接从该声明中了解到，可以按顺序使用多个 StreamTransformer`。

`StreamTransformer`可被用于进行任何类型的处理，例如：

* 过滤：根据任何类型的条件过滤数据，
* 重新组合：重新组合数据，
* 修改：对数据应用任何类型的修改，
* 将数据注入其他流，
* 缓冲，
* 处理：基于数据进行任意类型的操作
* ...

## 流的类型

存在两种类型的流。

### 单订阅流

这种类型的流只允许在该流的整个生命周期内使用单个监听器。

即使在第一个订阅被取消后，也无法在此类流上监听两次。

### 广播流

第二种类型的流允许任意数量的监听器。

可以随时向广播流添加监听器。新的监听器将在它开始监听流时收到事件。

## 基础的例子

### 任何类型的数据

第一个示例展示了“单订阅”流，它只是打印输入的数据。正如你所见的，数据的类型是无关紧要的。

```dart
import 'dart:async';

void main() {
  //
  // Initialize a "Single-Subscription" Stream controller
  //
  final StreamController ctrl = StreamController();
  
  //
  // Initialize a single listener which simply prints the data
  // as soon as it receives it
  //
  final StreamSubscription subscription = ctrl.stream.listen((data) => print('$data'));

  //
  // We here add the data that will flow inside the stream
  //
  ctrl.sink.add('my name');
  ctrl.sink.add(1234);
  ctrl.sink.add({'a': 'element A', 'b': 'element B'});
  ctrl.sink.add(123.45);
  
  //
  // We release the StreamController
  //
  ctrl.close();
}
```

### StreamTransformer

第二个示例显示“广播”流，它传达整数值并仅打印偶数。为此，我们应用StreamTransformer来过滤（第14行）值，只让偶数通过。

```dart
import 'dart:async';

void main() {
  //
  // Initialize a "Broadcast" Stream controller of integers
  //
  final StreamController<int> ctrl = StreamController<int>.broadcast();
  
  //
  // Initialize a single listener which filters out the odd numbers and
  // only prints the even numbers
  //
  final StreamSubscription subscription = ctrl.stream
					      .where((value) => (value % 2 == 0))
					      .listen((value) => print('$value'));

  //
  // We here add the data that will flow inside the stream
  //
  for(int i=1; i<11; i++){
  	ctrl.sink.add(i);
  }
  
  //
  // We release the StreamController
  //
  ctrl.close();
}
```

## RxDart
如今，如果我不提及 [RxDart包](https://pub.dartlang.org/packages/rxdart)，那么有关流的介绍就不会完整。

**RxDart** 包是 [ReactiveX](http://reactivex.io/) API的 Dart 实现，它扩展了原始的 Dart 流 API 以使其符合 ReactiveX 标准。

由于它最初并未由 Google 定义，因此它使用不同的词汇。下表给出了 Dart 和 RxDart 之间的相关性。

|Dart|RxDart|
|---|----|
| Stream | Observable |
| StreamController | Subject |


正如我刚才所说，RxDart 扩展了原始的 Dart Streams API 并提供了StreamController 的三个主要变体：

### PublishSubject
[PublishSubject](https://pub.dartlang.org/documentation/rxdart/latest/rx/PublishSubject-class.html)是一个标准的 **广播** StreamController,只有一点除外，流返回的是一个 [Observable](https://pub.dartlang.org/documentation/rxdart/latest/rx/Observable-class.html) 而不是一个 Stream。

![](https://www.didierboelens.com/images/S.PublishSubject.png)

正如你所看到的，`PublishSubject`只向监听器发送在订阅之后被添加到流中的事件。

### BehaviorSubject
[BehaviorSubject](https://pub.dartlang.org/documentation/rxdart/latest/rx/BehaviorSubject-class.html) 也是一个**广播** `StreamController`，它返回一个 **Observable** 而不是一个 **Stream**。

![](https://www.didierboelens.com/images/S.BehaviorSubject.png)

与 PublishSubject 的主要区别在于 BehaviorSubject 还将订阅前最后发送的事件发送给刚刚订阅的监听器。

### ReplaySubject
[ReplaySubject](https://pub.dartlang.org/documentation/rxdart/latest/rx/ReplaySubject-class.html) 也是一个 **广播** StreamController，它返回一个 **Observable** 而不是一个 **Stream**。

![](https://www.didierboelens.com/images/S.ReplaySubject.png)

**默认情况下**，ReplaySubject 将 Stream 已经发出的所有事件作为最先的事件发送到所有新订阅的监听器。

___

### 关于资源的重要说明

> 总是释放不再需要的资源是一种非常好的习惯。

这句话适用于：

* StreamSubscription - 当你不再需要监听流的时候，取消订阅
* StreamController - 当你不再需要一个 StreamController 的时候，关闭它。
* 同样也适用于 RxDart 这个主体，当你不再需要 BehaviourSubject，PublishSubject 的时候，关闭它。

___

### 如何基于从 Stream 出来的数据构建一个 Widget？

Flutter 提供了一个十分方便的 StatefulWidget，被称为 [StreamBuilder](https://docs.flutter.io/flutter/widgets/StreamBuilder-class.html)。

StreamBuilder 监听着一个 Stream ，每当一些数据流出 Stream 时，它会调用其builder 回调自动重建。

这是如何使用 StreamBuilder 的方法：

```dart
StreamBuilder<T>(
    key: ...optional, the unique ID of this Widget...
    stream: ...the stream to listen to...
    initialData: ...any initial data, in case the stream would initially be empty...
    builder: (BuildContext context, AsyncSnapshot<T> snapshot){
        if (snapshot.hasData){
            return ...the Widget to be built based on snapshot.data
        }
        return ...the Widget to be built if no data is available
    },
)
```
以下示例模仿默认的“计数器”应用程序，但使用的是 Stream 而不再使用任何 setState 。

```dart
import 'dart:async';
import 'package:flutter/material.dart';

class CounterPage extends StatefulWidget {
  @override
  _CounterPageState createState() => _CounterPageState();
}

class _CounterPageState extends State<CounterPage> {
  int _counter = 0;
  final StreamController<int> _streamController = StreamController<int>();

  @override
  void dispose(){
    _streamController.close();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Stream version of the Counter App')),
      body: Center(
        child: StreamBuilder<int>(
          stream: _streamController.stream,
          initialData: _counter,
          builder: (BuildContext context, AsyncSnapshot<int> snapshot){
            return Text('You hit me: ${snapshot.data} times');
          }
        ),
      ),
      floatingActionButton: FloatingActionButton(
        child: const Icon(Icons.add),
        onPressed: (){
          _streamController.sink.add(++_counter);
        },
      ),
    );
  }
}
```
解释说明：

* 第24-30行：我们正在监听流，每次有一个新值流出这个流时，我们用该值更新文本;
* 第35行：当我们点击 FloatingActionButton 时，我们递增计数器并通过接收器将其发送到 Stream ;在流中注入值的事实导致侦听它的 StreamBuilder 重建并“刷新”计数器;
* 我们不再需要状态的概念，所有东西都通过 Stream 接收;
* 这是一个很大的改进，因为调用 setState() 方法会强制整个 Widget（和任何子widget）重建。在这里，只重建 StreamBuilder (当然还有子widget);
* 我们仍然在为页面使用 StatefulWidget 的唯一原因，仅仅是因为我们需要通过 dispose 方法在第15行释放 StreamController;

___

## 什么是响应式编程

**响应式编程是使用异步数据流进行编程**。换句话说，所有东西都来源于一个事件(例如点击事件)，变量、消息的变化，要求构建的请求，所有这些经由数据流触发，可能改变或发生的事情都会被传送。

很明显，所有这些意味着，通过响应式编程，应用程序：
* 变得**异步**了
* 围绕**流**和**监听者**的概念进行架构，
* 当某事发生在某处(事件，变量的变化......)时，会向 Stream 发送通知，
* 如果“某人”监听该流，无论其在应用程序中的位置如何，它将被通知并将采取适当的行动。

> 组件之间不再存在紧密耦合

简而言之，当 Widget 向 Stream 发送内容时，该 Widget **不再需要知道**：

* 接下来会发生什么，
* 谁可能使用这些信息（没有一个，一个或几个widget...）
* 可能使用此信息的地方（没有地方，同一页面，另一个页面，几个页面...），
* 当这些信息可能被使用时（几乎是直接，几秒钟之后，永远不会......）。

> ...... Widget 只关心自己的业务，仅此而已。

乍一看，这似乎会导致应用程序变得“无法控制”，但正如我们将看到的，情况正好相反。它给予你：

* 构建仅负责特定活动的部分应用程序的机会，
* 轻松模拟一些组件的行为，以允许更完整的测试覆盖，
* 轻松重用组件（应用程序或其他应用程序中的其他位置），
* 重新设计应用程序，并能够在不进行太多重构的情况下将组件从一个地方移动到另一个地方，
...

我们将很快看到优势......但在之前，我需要介绍最后一个主题：BLoC 模式。
___ 

## BLoC模式

BLoC 模式由来自谷歌的 Paolo Soares 和 Cong Hui 设计，并在 2018 年 DartConf 期间（2018年1月23日至24日）首次提出。[观看YouTube上的视频](https://www.youtube.com/watch?v=PLHln7wHgPE)。

简而言之，业务逻辑需要：

* 需要迁移到一个或多个的BloC
* 尽可能从表示层中删除业务逻辑。换句话说，UI 组件应该只关心 UI 的东西而不关心业务，
* 依靠将输入(Sink)和输出(stream)作为唯一操作 Stream 的方式
* 保持平台独立性
* 保持环境独立性

事实上，BLoC 模式最初构思是为了允许重用独立于平台（ Web 应用程序，移动应用程序，后端。
）的相同的代码。

### 它究竟意味着什么？

BLoC 模式利用了我们刚才讨论过的概念：Streams。

![](https://www.didierboelens.com/images/streams_bloc.png)

* Widget通过 sinks 向 BloC 发送事件
* Widget通过 stream 被 BloC 通知的,
* 由 BLoc 实现的业务逻辑，Widget 丝毫都不关心

从这个声明中，我们可以直接看到一个巨大的好处。

正是因为业务逻辑跟 UI 解耦了

* 我们可以随时更改业务逻辑，并且其对应用程序的影响最小，
* 我们可以更改 UI，同时不会对业务逻辑产生任何影响，
* 现在，测试业务逻辑变得更加容易

___

## 如何把这个 Bloc 模式应用到计数器应用这个例子上？

将 BLoC 模式应用于此计数器应用程序似乎有点杀鸡用牛刀，但让我先展示给你看吧......

```dart
void main() => runApp(new MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
        title: 'Streams Demo',
        theme: new ThemeData(
          primarySwatch: Colors.blue,
        ),
        home: BlocProvider<IncrementBloc>(
          bloc: IncrementBloc(),
          child: CounterPage(),
        ),
    );
  }
}

class CounterPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final IncrementBloc bloc = BlocProvider.of<IncrementBloc>(context);

    return Scaffold(
      appBar: AppBar(title: Text('Stream version of the Counter App')),
      body: Center(
        child: StreamBuilder<int>(
          stream: bloc.outCounter,
          initialData: 0,
          builder: (BuildContext context, AsyncSnapshot<int> snapshot){
            return Text('You hit me: ${snapshot.data} times');
          }
        ),
      ),
      floatingActionButton: FloatingActionButton(
        child: const Icon(Icons.add),
        onPressed: (){
          bloc.incrementCounter.add(null);
        },
      ),
    );
  }
}

class IncrementBloc implements BlocBase {
  int _counter;

  //
  // Stream to handle the counter
  //
  StreamController<int> _counterController = StreamController<int>();
  StreamSink<int> get _inAdd => _counterController.sink;
  Stream<int> get outCounter => _counterController.stream;

  //
  // Stream to handle the action on the counter
  //
  StreamController _actionController = StreamController();
  StreamSink get incrementCounter => _actionController.sink;

  //
  // Constructor
  //
  IncrementBloc(){
    _counter = 0;
    _actionController.stream
                     .listen(_handleLogic);
  }

  void dispose(){
    _actionController.close();
    _counterController.close();
  }

  void _handleLogic(data){
    _counter = _counter + 1;
    _inAdd.add(_counter);
  }
}
```
我已经听到你说，“为什么这么做，有必要吗"？

### 首先，职责分离

如果你检查 CounterPage（第21-45行），其中绝对没有任何业务逻辑。

此页面现在仅负责：

* 显示计数器，现在只在必要的时候刷新（甚至没有页面需要知道它）
* 提供按钮，当按钮按下时，请求一个动作能在 couter 上执行。

此外，整个业务逻辑集中在一个单个类 “IncrementBloc” 中。

如果现在，你需要更改业务逻辑，你只需更新方法 _handleLogic（第77-80行）。 也许新的业务逻辑将要求做非常复杂的事情...... Counter 页面永远不会知道，这是非常好的！

### 第二，可测试性

现在，测试业务逻辑变得更加容易了。

无需再通过用户界面测试业务逻辑。 只需要测试 IncrementBloc 类。

### 第三，自由组织布局

由于使用了 Streams，你现在可以独立于业务逻辑组织布局。

可以从应用程序中的任何位置启动任何操作：只需调用.incrementCounter sink 即可。

你可以在任何页面的任何位置显示计数器，只需监听 .outCounter stream 。

### 第四，减少 “build” 的数量

不使用 setState() 而是使用 StreamBuilder ，大大减少了需要使用 “build” 的数量。

从性能角度来看，这是一个巨大的提升。
___

### 有一个约束... BLoC 的访问性

为了使所有这些能够工作，BLoC 需要能够被访问。

有几种方法可以访问它：

* 通过全局单例

这种方式十分可行，但不是特别推荐。 此外，由于 Dart 中没有类析构函数，因此你永远无法正确释放资源。

* 作为局部实例

您可以实例化 BLoC 的局部实例。 在某些情况下，此解决方案完全满足某些需求。 在这种情况下，你应该始终考虑在 StatefulWidget 中初始化，以便你可以利用 dispose() 方法来释放它。

* 由祖先提供

使其可访问的最常见方式是通过**祖先** Widget，以 StatefulWidget 的方式实现。

以下代码展示了泛型 BlocProvider 的示例。

```dart
// Generic Interface for all BLoCs
abstract class BlocBase {
  void dispose();
}

// Generic BLoC provider
class BlocProvider<T extends BlocBase> extends StatefulWidget {
  BlocProvider({
    Key key,
    @required this.child,
    @required this.bloc,
  }): super(key: key);

  final T bloc;
  final Widget child;

  @override
  _BlocProviderState<T> createState() => _BlocProviderState<T>();

  static T of<T extends BlocBase>(BuildContext context){
    final type = _typeOf<BlocProvider<T>>();
    BlocProvider<T> provider = context.ancestorWidgetOfExactType(type);
    return provider.bloc;
  }

  static Type _typeOf<T>() => T;
}

class _BlocProviderState<T> extends State<BlocProvider<BlocBase>>{
  @override
  void dispose(){
    widget.bloc.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context){
    return widget.child;
  }
}
```

### 关于这个泛型 BlocProvider 的一些解释

首先，如何将其用作提供者？

如果你查看示例代码“streams_4.dart”，您将看到以下代码行（第12-15行）

```dart
home: BlocProvider<IncrementBloc>(
          bloc: IncrementBloc(),
          child: CounterPage(),
        ),
```

有了这几行代码，我们很简单地实例化一个新的 BlocProvider，它将处理一个IncrementBloc，并将 CounterPage 作为子项渲染出来。

从那一刻开始，从 BlocProvider 开始的子树的任何小部件部分都将能够通过以下行访问IncrementBloc：

### 我们能有多个BLoc吗？

当然，这是非常可取的。 建议是：

* (如果有任何业务逻辑)每页顶部有一个BLoC，
* 为什么不是 ApplicationBloc 来处理应用程序状态？
* 每个“足够复杂的组件”都有相应的BLoC。


以下示例代码在整个应用程序的顶部显示 ApplicationBloc ，然后在CounterPage顶部显示 IncrementBloc。

该示例还展示了如何获取两个 BloC。

```dart
void main() => runApp(
  BlocProvider<ApplicationBloc>(
    bloc: ApplicationBloc(),
    child: MyApp(),
  )
);

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context){
    return MaterialApp(
      title: 'Streams Demo',
      home: BlocProvider<IncrementBloc>(
        bloc: IncrementBloc(),
        child: CounterPage(),
      ),
    );
  }
}

class CounterPage extends StatelessWidget {
  @override
  Widget build(BuildContext context){
    final IncrementBloc counterBloc = BlocProvider.of<IncrementBloc>(context);
    final ApplicationBloc appBloc = BlocProvider.of<ApplicationBloc>(context);
    
    ...
  }
}
```

### 为什么不使用 InhertitedWidget
在大多数与 BLoC 相关的文章中，你将看到 Provider 作为 InheritedWidget 的实现。

当然，没有什么能阻止这种类型的实现。 然而，

* 一个 InheritedWidget 没有提供任何 dispose 方法，记住，在不再需要资源时总是释放资源是一个很好的做法。
* 当然，没有什么能阻止你将 InheritedWidget 包装在另一个StatefulWidget中，但是，使用 InheritedWidget 增加了什么呢？
* 最后，如果不受控制，使用 InheritedWidget 经常会导致副作用（请参阅下面的InheritedWidget 上的提醒）。

这三点解释了我为什么将泛型 BlocProvider 实现为 StatefulWidget 作为我的选择，这样我就可以在处理这个 widget 时释放资源。

> **Flutter 无法实例化泛型类型**

> 不幸的是，Flutter 无法实例化泛型类型，我们必须将 BLoC 的实例传递给BlocProvider。为了在每个 BLoC 中强制执行 dispose() 方法，所有 BLoC 都必须实现BlocBase 接口。

#### 关于 InhertitedWidget 的提醒

在使用 InheritedWidget 并通过 context.inheritFromWidgetOfExactType(...)获取指定类型最近的 Widget 时，每当 InheritedWidget 的父级或者子布局发生变化时，这个方法会自动将当前“context”（= BuildContext）注册到要重建的widget当中。

> 请注意，为了完全正确，我刚才解释的与 InheritedWidget 相关的问题只发生在我们将InheritedWidget 与 StatefulWidget 结合使用时。 当你只使用没有 State 的InheritedWidget 时，问题就不会发生。 但是......我将在下一篇文章中回到这句话。

> 链接到 BuildContext 的 Widget 类型（Stateful或Stateless）是无关紧要的。

## 个人关于 BLoC 的看法

与 BLoC 相关的第三条规则是：“依靠输入(Sink)和输出(stream)作为唯一操作 Stream 的方式”。

我的个人经历与这句话有点相左......让我解释一下。

首先，BLoC 模式被设想成为跨平台共享相同的代码（AngularDart，...），并且从这个角度来看，该这句话非常有意义。

但是，如果您只打算开发一个 Flutter 应用程序，基于我粗浅的经验，这有用力过猛了。

如果我们坚持按这句话说的做，那么就没有 getter 或 setter方法了，只有 sinks 和streams。缺点就是，所有这些都是异步的。

让我们来举两个例子来说明缺点：

* 你需要从 BLoC 中获取一些数据，然后作为一个页面的输入，就应该直接在页面上展示这些参数（例如，想象一个参数页面），如果我们必须依赖 Streams，这会使页面的构建变成异步的，这是复杂的。通过Streams使其工作的示例代码可能如下所示......是不是很丑。

```dart
class FiltersPage extends StatefulWidget {
  @override
  FiltersPageState createState() => FiltersPageState();
}

class FiltersPageState extends State<FiltersPage> {
  MovieCatalogBloc _movieBloc;
  double _minReleaseDate;
  double _maxReleaseDate;
  MovieGenre _movieGenre;
  bool _isInit = false;

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();

    // As the context of not yet available at initState() level,
    // if not yet initialized, we get the list of the 
    // filter parameters
    if (_isInit == false){
      _movieBloc = BlocProvider.of<MovieCatalogBloc>(context);
      _getFilterParameters();
    }
  }

  @override
  Widget build(BuildContext context) {
    return _isInit == false
      ? Container()
      : Scaffold(
    ...
    );
  }

  ///
  /// Very tricky.
  /// 
  /// As we want to be 100% BLoC compliant, we need to retrieve
  /// everything from the BLoCs, using Streams...
  /// 
  /// This is ugly but to be considered as a study case.
  ///
  void _getFilterParameters() {
    StreamSubscription subscriptionFilters;

    subscriptionFilters = _movieBloc.outFilters.listen((MovieFilters filters) {
        _minReleaseDate = filters.minReleaseDate.toDouble();
        _maxReleaseDate = filters.maxReleaseDate.toDouble();

        // Simply to make sure the subscriptions are released
        subscriptionFilters.cancel();
        
        // Now that we have all parameters, we may build the actual page
        if (mounted){
          setState((){
            _isInit = true;
          });
        }
      });
    });
  }
}
```
* 在 BLoC 级别，你还需要转换某些数据的“假”注入，以触发提供你希望通过流接收的数据。 使这项工作的示例代码可以是：

```dart
class ApplicationBloc implements BlocBase {
  ///
  /// Synchronous Stream to handle the provision of the movie genres
  ///
  StreamController<List<MovieGenre>> _syncController = StreamController<List<MovieGenre>>.broadcast();
  Stream<List<MovieGenre>> get outMovieGenres => _syncController.stream;

  ///
  /// Stream to handle a fake command to trigger the provision of the list of MovieGenres via a Stream
  ///
  StreamController<List<MovieGenre>> _cmdController = StreamController<List<MovieGenre>>.broadcast();
  StreamSink get getMovieGenres => _cmdController.sink;

  ApplicationBloc() {
    //
    // If we receive any data via this sink, we simply provide the list of MovieGenre to the output stream
    //
    _cmdController.stream.listen((_){
      _syncController.sink.add(UnmodifiableListView<MovieGenre>(_genresList.genres));
    });
  }

  void dispose(){
    _syncController.close();
    _cmdController.close();
  }

  MovieGenresList _genresList;
}

// Example of external call

```

我不知道你的看法是怎么样的，但就个人而言，如果我没有任何与代码移植/共享相关的限制，我发现这太重了以至于我宁愿在需要时使用常规的 getter/setter并使用 Streams/Sinks来保持分离责任并在需要的地方广播信息。

## 是时候在实战中来看所有这些了...

正如本文开头所提到的，我构建了一个伪应用程序来展示如何使用所有这些概念。 完整的源代码可以在 [Github](https://github.com/boeledi/Streams-Block-Reactive-Programming-in-Flutter) 上找到。

请宽容，因为这段代码距离完美有很大的距离，它有进步的空间又或是采用更好的架构，但在这里的唯一目的是，向你展示所有这些事如何工作的。

由于源代码注释已经很多，我只会解释主要原则。

### 电影目录的来源

我正在使用免费的 TMDB API 来获取所有电影的列表，以及海报，评分和描述。

为了能够运行此示例应用程序，您需要注册并获取API密钥（完全免费），然后将您的API密钥放在文件“/api/tmdb_api.dart”第15行。

### 应该程序的架构

该应用程序使用：

* 3个主要的BLoC：
   *  ApplicationBloc（在所有顶部），负责提供所有电影类型的列表;
   *  FavoriteBloc（就在ApplicationBloc下面），负责处理“收藏夹”的概念;
   *  MovieCatalogBloc（在2个主要页面之上），负责基于过滤器提供电影列表;
   
* 6个页面：
	* 主页面：登陆页面，允许导航到3个子页面;
	* ListPage：将电影列为GridView的页面，允许过滤，收藏夹选择，访问收藏夹以及在后续页面中显示电影详细信息;
	* ListOnePage：类似于ListPage，但电影列表显示为水平列表，下面是详细信息;
	* 收藏页面：列出收藏夹的页面，允许取消选择任何收藏夹;
	* 过滤器页面：允许定义过滤器的EndDrawer：流派和最小/最大发布日期。从ListPage或* ListOnePage调用此页面;
	* 详细信息页面：页面仅由ListPage调用以显示电影的详细信息，但也允许选择/取消选择电影作为收藏;
	
* 1个子BLoC：
   * FavoriteMovieBloc，链接到MovieCardWidget或MovieDetailsWidget以处理作为收藏的电影的选择/取消选择
   
* 5个主要小部件：
	* FavoriteButton：负责显示收藏夹的数量wiget，实时，并在按下时重定向到FavoritesPage;
	* FavoriteWidget：负责显示一个喜欢的电影的细节的widget，并允许其取消选择;
	* FiltersSummary：负责显示当前定义的过滤器的widget;
	* MovieCardWidget: 负责将一部电影显示为卡片，电影海报，评级和名称，以及一个图标，表示选择该特定电影作为收藏的widget;
	* MovieDetailsWidget：负责显示与特定电影相关的详细信息，并允许其选择/取消选择作为收藏的widget。

### 不同BLoc/流的和谐结合

下图显示了如何使用主要的3个 BLoC：

* 在 BLoC 的左侧，哪些组件调用 Sink
* 在右侧，哪些组件监听流

例如，当 MovieDetailsWidget 调用 inAddFavorite Sink 时，会触发2个流：

* outTotalFavorites 流强制重建 FavoriteButton，和
* outFavorites 流
	* 强制重建 MovieDetailsWidget（“最喜欢的”图标）
	* 强制重建 _buildMovieCard（“最喜欢的”图标）
	* 用于构建每个 MovieDetailsWidget
![](https://www.didierboelens.com/images/streams_flows.png) 

### 观察者

大多数小部件和页面都是 StatelessWidgets，这意味着：

* 强制重建的 setState()几乎从未使用过。 例外情况是：
	* 当 ListOnePage 用户点击 MovieCard 时，刷新 MovieDetailsWidget。 这也可能是由一个流驱动的......
	* 在 FiltersPage 中允许用户在接受过滤器之前通过 Sink 更改过滤器。
应用程序不使用任何 InheritedWidget
	* 应用程序几乎是 100％ BLoC/Stream 驱动，这意味着大多数小部件彼此独立，并且它们在应用程序中的位置

一个实际的例子是 FavoriteButton，它显示徽章中所选收藏夹的数量。 该应用程序计算此FavoriteButton 的3个实例，每个实例显示在3个不同的页面中。

### 显示电影列表（显示无限列表的技巧说明）

要显示符合过滤条件的电影列表，我们使用 GridView.builder(ListPage)或ListView.builder(ListOnePage）作为无限滚动列表。

使用 TMDB API 一次以20个电影的页面提取电影。

提醒一下，GridView.builder 和 ListView.builder 都将 itemCount 作为输入，如果提供，则表示要显示的项目数。调用 itemBuilder，索引从 0 到 itemCount-1 不等。

正如你将在代码中看到的那样，我随意为 GridView.builder 添加了30多个。理由是，在这个例子中，我们正在操纵假定的无限数量的项目（这不是完全正确但是谁在乎这个例子）。这将强制 GridView.builder 请求显示“最多30个”项目。

此外，GridView.builder和ListView.builder只在认为必须在视口中呈现某个项目（索引）时才调用itemBuilder。

这个MovieCatalogBloc.outMoviesList返回一个List <MovieCard>，它被迭代以构建每个Movie Card。第一次，这个List <MovieCard>是空的但是由于itemCount：... + 30，我们欺骗系统，它将要求通过 _buildMovieCard（...）呈现30个不存在的项目。

正如你将在代码中看到的，此例程对 Sink 进行了一次奇怪的调用：

```dart
//通知MovieCatalogBloc我们正在渲染MovieCard [索引]
movieBloc.inMovieIndex.add(index);
```
此调用告诉 MovieCatalogBloc 我们要渲染 MovieCard[index]。

然后_buildMovieCard（...）继续验证与MovieCard [index]相关的数据是否存在。如果是，则渲染后者，否则显示CircularProgressIndicator。

对StreamCatalogBloc.inMovieIndex.add（index）的调用由StreamSubscription监听，该索引将索引转换为某个pageIndex数字（一页最多可计数20个电影）。如果尚未从TMDB API获取相应页面，则会调用API。获取页面后，所有已获取电影的新列表将发送到_moviesController。当GridView.builder监听该流（= movieBloc.outMoviesList）时，后者请求重建相应的MovieCard。由于我们现在有数据，我们可能会渲染它。

___

## 致谢与附加链接

描述PublishSubject，BehaviorSubject和ReplaySubject的图像由ReactiveX发布。

其他一些值得阅读的有趣文章：

* [Dart Streams的基础知识](https://www.burkharts.net/apps/blog/)[Thomas Burkhart]
* [rx_command包](https://pub.dartlang.org/packages/rx_command)[Thomas Burkhart]
* [在Flutter中构建响应式移动应用程序 - 配套文章](https://medium.com/flutter-io/build-reactive-mobile-apps-in-flutter-companion-article-13950959e381)[Filip Hracek]
* [使用Streams和RxDart写Flutter](https://skillsmatter.com/skillscasts/12254-flutter-with-streams-and-rxdart)[Brian Egan]

___

## 结论

很长的文章，但是还有很多要说的话。就对我而言，显而易见的，这是推动Flutter应用程序开发的方法，它提供了很大的灵活性。

请继续关注新的文章。 快乐编程。
