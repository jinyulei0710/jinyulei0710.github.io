---
layout: post
title:  "(译)Flutter bloc包"
date:   2019-04-22 14:16:17 +0800
categories: 
   - Flutter 开发
tags:
    - 状态管理 
    - BLoC
---
[原文链接](https://medium.com/flutter-community/flutter-bloc-package-295b53e95c5c)

![](https://cdn-images-1.medium.com/max/1600/1*lP-1aF6Rg8jo459f87l3zg.png)

在使用Flutter一段时间之后，我决定创造一个包帮忙我经常使用的东西—BLoC模式做一些事情。

<!--more-->


对于那些不熟悉BLoC模式的人来说，它是一种设计模式，它有助于将表示层与业务逻辑分开。你从[这里](https://www.youtube.com/watch?v=fahC3ky_zW0)能够了解更多。

虽然使用BLoC模式可能会因为设置以及对`Streams`和`Reactive Programming`的理解而具有挑战性，但它的核心BLoC是非常简单的：

BLoC将事件流作为输入，并将它们转换为状态流作为输出。

![](https://cdn-images-1.medium.com/max/1600/1*_EgLk67LpREOEMCSIhc_4Q.png)

我们现在可以在bloc包的帮助下使用这种强大的设计模式。

该软件包抽象了BLoC模式的响应式方面，允许开发人员专注于将事件转换为状态。

让我们从定义这些术语开始。


## 词汇表

事件(`Events`)是BLoC的输入。它们通常是UI事件，例如按钮按下。事件(`Events`)被发送(`dispatched`)并转换为状态(`States`)。

状态(`State`)是BLoC的产物。表示组件可以监听状态流并根据给定状态对其自身的部分进行重绘(有关详细信息，请参阅`BlocBuilder`)。

转换(`Transitions`)发生在mapEventToState被调用之后，当事件被发送之后，但在bloc状态被修改之前。一个转换(`Transition`)由当前状态(`currentState`)，已分送的事件(`event`)和下个状态(`nextState`)组成。

现在我们了解事件和状态，我们可以看一下Bloc API了。

## Bloc API

mapEventToState是一个类在扩展Bloc时必须实现的方法。该方法将传入事件作为参数。只要有一个事件被表示层发送(`dispatched`)，就会调用`mapEventToState`。 `mapEventToState`必须将该事件转换为新状态，并`Stream`的形式返回新状态,而这个新状态会被表示层使用。

`dispatch`是一个接收事件并触发`mapEventToState`的方法。可以从表示层或从Bloc内部调用dispatch（参见示例）并向Bloc通知新事件来了。

`initialState`是任何事件都没有被处理之前的状态（在调用`mapEventToState`之前）。initialState是一个可选的getter。如果未实现，则initialState将为null。

`transform`是一个方法，可以在调用`mapEventToState`之前重写Stream<Event>。这其中允许使用distinct()和debounce()之类的操作。

`onTransition`是一个方法，可以在发生转换时覆盖对其加以处理。转换(`Transition`)发生在新的事件(`event`)被发送以及`mapEventToState`被调用的时候。`onTransition`在bloc的状态被更新之前被调用。这是添加bloc特有的日志记录/分析的好地方。

让我们创建一个计数器bloc！

```dart
enum CounterEvent { increment, decrement }

class CounterBloc extends Bloc<CounterEvent, int> {
  @override
  int get initialState => 0;

  @override
  Stream<int> mapEventToState(CounterEvent event) async* {
    switch (event) {
      case CounterEvent.decrement:
        yield currentState - 1;
        break;
      case CounterEvent.increment:
        yield currentState + 1;
        break;
    }
  }
}
```

为了创建一个计数器bloc，我们需要做的就是：

* 定义我们的事件和状态
* 扩展BloC
* 覆盖`initialState`和`mapEventToState`。

在这种情况中，我们的事件是`CounterEvents`，我们的状态是整数类型。

我们的`CounterBloc`将`CounterEvents`转换为整数类型。

我们可以通过像这样调用`dispatch`方法来通知CounterBloc发出事件：

```dart
void main() {
  final counterBloc = CounterBloc();

  counterBloc.dispatch(CounterEvent.increment);
  counterBloc.dispatch(CounterEvent.decrement);
}
```

为了观察状态变化(`Transitions`），我们可以覆盖`onTransition`。

```dart
enum CounterEvent { increment, decrement }

class CounterBloc extends Bloc<CounterEvent, int> {
  @override
  int get initialState => 0;
  
  @override
  void onTransition(Transition<CounterEvent, int> transition) {
    print(transition);
  }

  @override
  Stream<int> mapEventToState(CounterEvent event) async* {
    switch (event) {
      case CounterEvent.decrement:
        yield currentState - 1;
        break;
      case CounterEvent.increment:
        yield currentState + 1;
        break;
    }
  }

```

现在，每当我们调度CounterEvent时，我们的Bloc将以新的整数状态响应，我们将看到一个转换记录被打印到到控制台。

现在让我们使用Flutter构建一个UI，并使用[flutter_bloc](https://pub.dartlang.org/packages/flutter_bloc)包将表示层连接到我们的`CounterBloc`。

[flutter_bloc](https://pub.dartlang.org/packages/flutter_bloc)包提供了两个widget，使得可以轻松地与Bloc进行交互：

## BlocBuilder
`BlocBuilder`是一个Flutter widget，它需要一个Bloc和一个builder方法。 BlocBuilder处理了构建widget的工作以响应新状态。BlocBuilder与StreamBuilder非常相似，但它有一个更简单的API来减少所需的样板代码量。

## BlocProvider
`BlocProvider`是一个Flutter widget，它通过`BlocProvider.of(context)`为其子节点提供一个集合。它被用作依赖注入(DI)widget，以便可以将BloC的单例提供给子树中的多个widget。

现在让我们构建我们的Counter App！

```dart

class App extends StatefulWidget {
  @override
  State<StatefulWidget> createState() => _AppState();
}

class _AppState extends State<App> {
  final CounterBloc _counterBloc = CounterBloc();

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      home: BlocProvider<CounterBloc>(
        bloc: _counterBloc,
        child: CounterPage(),
      ),
    );
  }

  @override
  void dispose() {
    _counterBloc.dispose();
    super.dispose();
  }
}

class CounterPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final CounterBloc _counterBloc = BlocProvider.of<CounterBloc>(context);

    return Scaffold(
      appBar: AppBar(title: Text('Counter')),
      body: BlocBuilder<CounterEvent, int>(
        bloc: _counterBloc,
        builder: (BuildContext context, int count) {
          return Center(
            child: Text(
              '$count',
              style: TextStyle(fontSize: 24.0),
            ),
          );
        },
      ),
      floatingActionButton: Column(
        crossAxisAlignment: CrossAxisAlignment.end,
        mainAxisAlignment: MainAxisAlignment.end,
        children: <Widget>[
          Padding(
            padding: EdgeInsets.symmetric(vertical: 5.0),
            child: FloatingActionButton(
              child: Icon(Icons.add),
              onPressed: () {
                _counterBloc.dispatch(CounterEvent.increment);
              },
            ),
          ),
          Padding(
            padding: EdgeInsets.symmetric(vertical: 5.0),
            child: FloatingActionButton(
              child: Icon(Icons.remove),
              onPressed: () {
                _counterBloc.dispatch(CounterEvent.decrement);
              },
            ),
          ),
        ],
      ),
    );
  }
}
```


我们的App widget是`StatefulWidget`，负责创建和处理CounterBloc。它使用我们上面提到的BlocProvider widget使CounterBloc可用于CounterPage widget。

我们的`CounterPage` widget是一个`StatelessWidget`，它使用`BlocBuilder`重建UI以响应`CounterBloc`的状态变化。

此时，我们已经成功地将我们的表示层与业务逻辑层分开。请注意，CounterPage widget 不知道用户点击按钮时会发生什么。widget只是告诉CounterBloc用户按下了递增或递减按钮。

就这么多。

有关更多示例和详细文档，请查看[官方bloc文档](https://felangel.github.io/bloc)。

如果你喜欢这个bloc库，你可以通过⭐️[仓库](https://github.com/felangel/bloc)或者👏来支持我这个文章。