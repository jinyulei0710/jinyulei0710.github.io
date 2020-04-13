---
layout: post
title:  "(译)Flutter中的响应式编程(Reactive Programming)、流(Streams)、业务逻辑组件(BloC)以及实际使用案例"
date:   2019-04-13 13:51:17 +0800
categories: 
   - Flutter 开发
tags:
    - 状态管理
    - BLoC
---
[原文链接](https://www.didierboelens.com/2018/12/reactive-programming---streams---bloc---practical-use-cases/)

Flutter中的响应式编程(Reactive Programming)、流(Streams)、业务逻辑组件(BloC)以及实际使用案例

难度：中级

<!--more-->

## 介绍

在前一段时间介绍了业务逻辑组件(BloC)，响应式编程(Reactive Programming)和流(Streams)这些概念之后，我想与你分享一些我经常使用的并且个人觉得十分有用的模式，应该会是很有趣的。在我的开发过程中，这些模式使我节省了大量时间，并使我的代码更易于阅读和调试。

我要讨论的话题有：

* `BLoC Provider` 以及 `InheritedWidget`
* 在哪初始化 `BLoC`
* 事件-状态

  允许基于事件对状态转换作出响应
* 表单验证
  允许依据条目和验证控制表单的行为。(我解释的解决方案还包括密码和重新输入密码的比较)。

* `Part of`

  允许Widget根据其在列表中的是否存在来调整其行为。
  
  完整的源代码可以在 [GitHub](https://github.com/boeledi/blocs) 上下载。
  
 ___
 
 
## `BlocProvider` 和 `InheritedWidget`
 
我借此文章的机会介绍我的另一个版本的`BlocProvider`，它现在依赖于一个`InheritedWidget`。

使用`InheritedWidget`的优点是我们获得了性能上的提升。

让我来解释下。

### 之前的实现

我之前版本的BlocProvider是按常规的StatefulWidget实现的，如下所示：

```dart
abstract class BlocBase {
  void dispose();
}
//泛型Bloc Provider
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

我使用了 StatefulWidget 从而从它的 dispose() 方法中受益，以确保在不再需要时释放BLoC 分配的资源。

这很好用但从性能角度来看并不是最佳的。

context.ancestorWidgetOfExactType() 是一个 O(n) 方法。为了获取要求的祖先widget，而这个祖先对应某种乐行，它从`context`开始向上遍历树，每向上移动一个都要递归一遍，直到遍历完成。如果`context`到祖先的距离很小，则这个方法调用是可以接受的，否则应该避免对其进行调用。这是这个方法的代码。

```dart
@override
Widget ancestorWidgetOfExactType(Type targetType) {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    Element ancestor = _parent;
    while (ancestor != null && ancestor.widget.runtimeType != targetType)
        ancestor = ancestor._parent;
    return ancestor?.widget;
}
```

### 新的实现

新的实现依赖于一个与 InheritedWidget 结合 StatefulWidget：

```dart

Type _typeOf<T>() => T;

abstract class BlocBase {
  void dispose();
}

class BlocProvider<T extends BlocBase> extends StatefulWidget {
  BlocProvider({
    Key key,
    @required this.child,
    @required this.bloc,
  }): super(key: key);

  final Widget child;
  final T bloc;

  @override
  _BlocProviderState<T> createState() => _BlocProviderState<T>();

  static T of<T extends BlocBase>(BuildContext context){
    final type = _typeOf<_BlocProviderInherited<T>>();
    _BlocProviderInherited<T> provider = 
            context.ancestorInheritedElementForWidgetOfExactType(type)?.widget;
    return provider?.bloc;
  }
}

class _BlocProviderState<T extends BlocBase> extends State<BlocProvider<T>>{
  @override
  void dispose(){
    widget.bloc?.dispose();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context){
    return new _BlocProviderInherited<T>(
      bloc: widget.bloc,
      child: widget.child,
    );
  }
}

class _BlocProviderInherited<T> extends InheritedWidget {
  _BlocProviderInherited({
    Key key,
    @required Widget child,
    @required this.bloc,
  }) : super(key: key, child: child);

  final T bloc;

  @override
  bool updateShouldNotify(_BlocProviderInherited oldWidget) => false;
}
```

这个解决方案的优点是性能。

由于使用了 `InheritedWidget`，它现在可以调用context.ancestorInheritedElementForWidgetOfExactType() 方法，它是一个O(1)方法。这意味祖先是立即获取的，如其源代码所示：

```dart
@override
InheritedElement ancestorInheritedElementForWidgetOfExactType(Type targetType) {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    final InheritedElement ancestor = _inheritedWidgets == null 
                                    ? null 
                                    : _inheritedWidgets[targetType];
    return ancestor;
}
```

这是因为事实上，所有`InheritedWidgets`都由框架记住了。

为什么使用 `ancestorInheritedElementForWidgetOfExactType`？

你可能已经注意到我使用 ancestorInheritedElementForWidgetOfExactType 方法,而不是通常的 inheritFromWidgetOfExactType。

原因是我不希望调用 `BlocProvider` 的 `context` 被注册为 `InheritedWidget`的依赖项，因为我不需要它。

### 如何使用新的 BlocProvider?
#### Bloc的注入

```dart
Widget build(BuildContext context){
    return BlocProvider<MyBloc>{
        bloc: myBloc,
        child: ...
    }
}
```

#### Bloc的获取

```dart
Widget build(BuildContext context){
    MyBloc myBloc = BlocProvider.of<MyBloc>(context);
    ...
}
```

## 从哪初始化 BLoc

要回答这个问题，你需要确认它的使用范围。

### 应用中随处可用

假设你必须处理与用户身份验证/配置文件，用户设置，购物篮相关的一些机制......任何需要从应用程序的任何可能部分（例如，从不同页面）获得BLoC的任何机制，存在两种方式使这个BLoC可访问。

#### 全局单例的使用

此解决方案依赖于使用 Global 对象，对所有对象实例化，而不是任何 Widget 树的一部分。

```dart
import 'package:rxdart/rxdart.dart';

class GlobalBloc {
  ///
  /// Streams related to this BLoC
  ///
  BehaviorSubject<String> _controller = BehaviorSubject<String>();
  Function(String) get push => _controller.sink.add;
  Stream<String> get stream => _controller;

  ///
  /// Singleton factory
  ///
  static final GlobalBloc _bloc = new GlobalBloc._internal();
  factory GlobalBloc(){
    return _bloc;
  }
  GlobalBloc._internal();
  
  ///
  /// Resource disposal
  ///
  void dispose(){
    _controller?.close();
}

GlobalBloc globalBloc = GlobalBloc();
```

要使用此 `BLoC`，你只需导入该类并直接调用其方法，如下所示：

```dart
import 'global_bloc.dart';

class MyWidget extends StatelessWidget {
    @override
    Widget build(BuildContext context){
        globalBloc.push('building MyWidget');
        return Container();
    }
}
```

如果你需要一个独特的 BLoC 并且需要从应用程序内部的任何位置访问，这是一个可接受的解决方案。

* 这是非常容易使用;
* 它不依赖于任何 `BuildContext`;
* 无需通过任何 `BlocProvider` 来查找 `BLoC`，
* 为了释放它的资源，只需确保将应用程序实现为 StatefulWidget，并在应用程序Widget的重载 dispose() 方法中调用 globalBloc.dispose()。

许多纯粹主义者反对这种解决方案。我不知道为什么，但是...所以让我们看看另一个......

#### 把它放在所有东西的顶部

在 Flutter 中，所有页面的祖先本身必须是 MaterialApp 的父级。这是因为页面（或路径）被包装在 OverlayEntry 中，这是所有页面的公共堆栈的子项。

换句话说，每个页面都有一个 Buildcontext，它独立于任何其他页面。这就解释了为什么在不使用任何技巧的情况下，两个页面(路由)不可能有任何共同点。

因此，如果你需要在应用程序中的任何位置使用 `BLoC`，则必须将其作为 MaterialApp 的父级，如下所示：

```dart
void main() => runApp(Application());

class Application extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocProvider<AuthenticationBloc>(
      bloc: AuthenticationBloc(),
      child: MaterialApp(
        title: 'BLoC Samples',
        theme: ThemeData(
          primarySwatch: Colors.blue,
        ),
        home: InitializationPage(),
      ),
    );
  }
}
```

### 在子树上能访问
大多数情况下，您可能需要在应用程序的某些特定部分使用`BLoC`。

作为一个例子，我们可以考虑将`BLoC`用于一个讨论线程

* 与服务器交互以获取，添加，更新帖子
* 列出要在某一页面中显示的线程
* ...

对于此示例，您不需要此 `BLoC` 可用于整个应用程序，而是需要某些 `Widget` (树的一部分)。

第一种解决方案可能是将 `BLoC` 注入 `Widget` 树的根目录，如下所示：

```dart
class MyTree extends StatelessWidget {
  @override
  Widget build(BuildContext context){
    return BlocProvider<MyBloc>(
      bloc: MyBloc(),
      child: Column(
        children: <Widget>[
          MyChildWidget(),
        ],
      ),
    );
  }
}

class MyChildWidget extends StatelessWidget {
  @override 
  Widget build(BuildContext context){
    MyBloc = BlocProvider.of<MyBloc>(context);
    return Container();
  }
}
```

这样，所有 widget 都将通过调用 `BlocProvider.of` 方法访问`BLoC`。

边注

如上所示的解决方案并不是最佳的，因为它将在每次重建时实例化BLoC。

后果：

* 你将丢失现有`BLoC`所有内容
* 它会耗费CPU时间，因为它需要在每次构建时实例化它。

在这种情况下，更好的方法是使用 `StatefulWidget` 从其持久状态中受益，如下所示：

```dart
class MyTree extends StatefulWidget {
 @override
  _MyTreeState createState() => _MyTreeState();
}
class _MyTreeState extends State<MyTree>{
  MyBloc bloc;
  
  @override
  void initState(){
    super.initState();
    bloc = MyBloc();
  }
  
  @override
  void dispose(){
    bloc?.dispose();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context){
    return BlocProvider<MyBloc>(
      bloc: bloc,
      child: Column(
        children: <Widget>[
          MyChildWidget(),
        ],
      ),
    );
  }
}
```

使用这种方法，如果 “MyTree” widget 需要重建，则不必重新实例化 BLoC 并直接重用现有实例。

### 只对一个widget可访问

这涉及 `BLoC `仅由一个 Widget 使用的情况。

在这种情况下，可以在 Widget 中实例化 BLoC。

## 事件状态

有时，处理一系列可能是顺序或并行，长或短，同步或异步以及可能导致各种结果的活动可能变得非常难以编程。你可能要随着进度或根据状态更新显示的内容。

第一个实例的目的是使这种情况更容易处理。

该解决方案是基于以下原则的：

* 一个事件被发送出来
* 此事件触发一些导致一个或多个状态的动作;
* 这些状态中的每一个都可以反过来发出其他事件或导致另一个状态;
* 然后，这些事件将根据有效的状态触发其它动作;
* 等等…

为了阐明这个概念，我们来看两个常见的例子：

* 应用初始化

假设你需要运行一系列动作来初始化应用程序。动作可能与服务器的交互相关联(例如，加载一些数据)。在此初始化过程中，您可能需要显示进度条和一系列图像来让用户等待。

* 认证

在启动时，应用程序可能需要用户进行身份验证或注册。用户通过身份验证后，将重定向到应用程序的主页面。然后，如果用户注销，则将其重定向到认证页面。

为了能够处理所有可能的情况以及事件序列，但是如果我们考虑到事件可以在应用程序中的任何地方被触发，这可能变得非常难以管理。

这正是`BlocEventState`与`BlocEventStateBuilder`相结合可以帮助很多的地方。

### BlocEventState

`BlocEventState`背后的思想是定义一个`BLoC`：
* 接受事件(`Event`)作为输入;
* 在新事件(`event`)被发送的时候调用事件处理器(`eventHandler`);
* 事件处理器(`eventHandler`)负责根据事件(`event`)采取适当的行动(`action`)并发出状态(`State`)作为响应。

下图展示了这个思想：

![](https://www.didierboelens.com/images/bloc_event_state.png)

这是这个类的源代码。解释如下：

```dart
import 'package:blocs/bloc_helpers/bloc_provider.dart';
import 'package:meta/meta.dart';
import 'package:rxdart/rxdart.dart';

abstract class BlocEvent extends Object {}
abstract class BlocState extends Object {}

abstract class BlocEventStateBase<BlocEvent, BlocState> implements BlocBase {
  PublishSubject<BlocEvent> _eventController = PublishSubject<BlocEvent>();
  BehaviorSubject<BlocState> _stateController = BehaviorSubject<BlocState>();

  ///
  /// To be invoked to emit an event
  ///
  Function(BlocEvent) get emitEvent => _eventController.sink.add;

  ///
  /// Current/New state
  ///
  Stream<BlocState> get state => _stateController.stream;

  ///
  /// External processing of the event
  ///
  Stream<BlocState> eventHandler(BlocEvent event, BlocState currentState);

  ///
  /// initialState
  ///
  final BlocState initialState;

  //
  // Constructor
  //
  BlocEventStateBase({
    @required this.initialState,
  }){
    //
    // For each received event, we invoke the [eventHandler] and
    // emit any resulting newState
    //
    _eventController.listen((BlocEvent event){
      BlocState currentState = _stateController.value ?? initialState;
      eventHandler(event, currentState).forEach((BlocState newState){
        _stateController.sink.add(newState);
      });
    });
  }

  @override
  void dispose() {
    _eventController.close();
    _stateController.close();
  }
}
```

如你所见，这是一个需要扩展的抽象类，要在扩展类中定义`eventHandler`方法的行为。

它暴露了：

* 一个`Sink`(`emitEvent`)来推送一个事件;
* 一个`Stream`(`state`)来监听发出的状态。

在初始化时（请参阅构造函数）：
* 需要提供`initialState`;
* 它创建一个`StreamSubscription`来监听传入的事件
   * 将它们发送到`eventHandler`
   * 发出作为结果的状态。

### 专门的 `BlocEventState`

用于实现此类 BlocEventState 的模板在下面给出。之后，我们将实现一个真正的。

```dart
class TemplateEventStateBloc extends BlocEventStateBase<BlocEvent, BlocState> {
  TemplateEventStateBloc()
      : super(
          initialState: BlocState.notInitialized(),
        );

  @override
  Stream<BlocState> eventHandler( BlocEvent event, BlocState currentState) async* {
     yield BlocState.notInitialized();
  }
}
``` 

如果这个模板没有编译通过，请不要担心......这是正常的，因为我们还没有定义`BlocState.notInitialized()`......过一会就有了。

此模板仅在初始化时提供 `initialState` 并覆盖了 `eventHandler`。

这里有一些非常有趣的事情需要注意。我们使用异步生成器：async* 和 yield 声明。

使用 async* 修饰符标记方法，将方法标识为异步生成器：

每次 yield 语句被调用的时候，它都会将 yield 后面的表达式结果添加到输出流。

如果我们需要发送作为一系列动作结果的一系列状态，这是特别有用的。

有关异步生成器的其它详细信息，请单击此[链接](/dart/async/2019/05/04/Dart-Language-Asynchrony-SupportPhase2/)。

### BlocEvent 和 BlocState

正如你已经注意到的，我们已经定义了一个 `BlocEvent` 和 `BlocState` 抽象类。

这些类需要被你想要发出的专门的事件和状态进行扩展。

### `BlocEventStateBuilder` widget

这个模式的最后一部分是 `BlocEventStateBuilder` Widget，它允许对`BlocEventState` 发出的`State`作出响应。

这里是源代码

```dart
typedef Widget AsyncBlocEventStateBuilder<BlocState>(BuildContext context, BlocState state);

class BlocEventStateBuilder<BlocEvent,BlocState> extends StatelessWidget {
  const BlocEventStateBuilder({
    Key key,
    @required this.builder,
    @required this.bloc,
  }): assert(builder != null),
      assert(bloc != null),
      super(key: key);

  final BlocEventStateBase<BlocEvent,BlocState> bloc;
  final AsyncBlocEventStateBuilder<BlocState> builder;

  @override
  Widget build(BuildContext context){
    return StreamBuilder<BlocState>(
      stream: bloc.state,
      initialData: bloc.initialState,
      builder: (BuildContext context, AsyncSnapshot<BlocState> snapshot){
        return builder(context, snapshot.data);
      },
    );
  }
}
```

这个 Widget 不是别的，就是一个专门的 `StreamBuilder`，它会在每次新的 BlocState被发出之后调用`builder`输入参数。
___

好的。现在我们什么都有了，现在是时候展示我们可以用它们做些什么了......

### 例1: 应用程序初始化

第一个例子举例说明了你需要应用程序在启动时执行某些任务的情况。

常见的用途是游戏最初显示启动页(不一定有动画)，同时从服务器获取一些文件，检查是否有更新，尝试连接到所有的游戏中心......在显示真正的主页面之前。为了不给应用程序什么都不做的感觉，它可能会显示一个进度条并间隔地显示一些图片，同时它会完成所有初始化过程。

我要向你展示的实现非常简单。它只会在屏幕上显示一些完成百分比，但这可以很容易地扩展从而以满足你的需求。

首先要做的是定义事件和状态......

#### 应用初始化事件

在这个例子中，我只会考虑2个事件：

* start：此事件将触发初始化过程;
* stop：该事件可用于强制初始化进程停止

这里是定义：
```dart
class ApplicationInitializationEvent extends BlocEvent {
  
  final ApplicationInitializationEventType type;

  ApplicationInitializationEvent({
    this.type: ApplicationInitializationEventType.start,
  }) : assert(type != null);
}

enum ApplicationInitializationEventType {
  start,
  stop,
}
```

#### 应用初始化件状态

该类将提供与初始化过程相关的信息。

对于这个例子，我会考虑：

 * 2个标记：
    * isInitialized指示初始化是否完成
    * isInitializing以了解我们是否处于初始化过程的中间
  
 * 进度完成率

这里是源代码：

```dart
class ApplicationInitializationState extends BlocState {
  ApplicationInitializationState({
    @required this.isInitialized,
    this.isInitializing: false,
    this.progress: 0,
  });

  final bool isInitialized;
  final bool isInitializing;
  final int progress;

  factory ApplicationInitializationState.notInitialized() {
    return ApplicationInitializationState(
      isInitialized: false,
    );
  }

  factory ApplicationInitializationState.progressing(int progress) {
    return ApplicationInitializationState(
      isInitialized: progress == 100,
      isInitializing: true,
      progress: progress,
    );
  }

  factory ApplicationInitializationState.initialized() {
    return ApplicationInitializationState(
      isInitialized: true,
      progress: 100,
    );
  }
}
```

#### 应用初始化 Bloc

该 BLoC 负责基于事件处理初始化过程。

这里是代码：

```dart
class ApplicationInitializationBloc
    extends BlocEventStateBase<ApplicationInitializationEvent, ApplicationInitializationState> {
  ApplicationInitializationBloc()
      : super(
          initialState: ApplicationInitializationState.notInitialized(),
        );

  @override
  Stream<ApplicationInitializationState> eventHandler(
      ApplicationInitializationEvent event, ApplicationInitializationState currentState) async* {
    
    if (!currentState.isInitialized){
      yield ApplicationInitializationState.notInitialized();
    }

    if (event.type == ApplicationInitializationEventType.start) {
      for (int progress = 0; progress < 101; progress += 10){
        await Future.delayed(const Duration(milliseconds: 300));
        yield ApplicationInitializationState.progressing(progress);
      }
    }

    if (event.type == ApplicationInitializationEventType.stop){
      yield ApplicationInitializationState.initialized();
    }
  }
}
```

一些解释：

* 当收到事件“ApplicationInitializationEventType.start”时，它从0到100开始计数（步骤10），并且对于每个值（0,10,20，......），它发出（通过yield）一个告诉的新状态初始化正在运行（isInitializing = true）及其进度值。
* 当收到事件“ApplicationInitializationEventType.stop”时，它认为初始化已完成。
* 正如你所看到的，我在计数器循环中放了一些延迟。这将向您展示如何使用任何Future（例如，您需要联系服务器的情况

#### 将它们全部包装在一起

现在，剩下的部分是显示显示计数器的伪 Splash 页面......

```dart
class InitializationPage extends StatefulWidget {
  @override
  _InitializationPageState createState() => _InitializationPageState();
}

class _InitializationPageState extends State<InitializationPage> {
  ApplicationInitializationBloc bloc;

  @override
  void initState(){
    super.initState();
    bloc = ApplicationInitializationBloc();
    bloc.emitEvent(ApplicationInitializationEvent());
  }

  @override
  void dispose(){
    bloc?.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext pageContext) {
    return SafeArea(
      child: Scaffold(
        body: Container(
          child: Center(
            child: BlocEventStateBuilder<ApplicationInitializationEvent, ApplicationInitializationState>(
              bloc: bloc,
              builder: (BuildContext context, ApplicationInitializationState state){
                if (state.isInitialized){
                  //
                  // Once the initialization is complete, let's move to another page
                  //
                  WidgetsBinding.instance.addPostFrameCallback((_){
                    Navigator.of(context).pushReplacementNamed('/home');
                  });
                }
                return Text('Initialization in progress... ${state.progress}%');
              },
            ),
          ),
        ),
      ),
    );
  }
}
```

说明：

* 由于`ApplicationInitializationBloc`不需要在应用程序的任何地方使用，我们可以在`StatefulWidget`中初始化它;
* 我们直接发出`ApplicationInitializationEventType.start`事件来触发`eventHandler`
* 每次发出`ApplicationInitializationState`时，我们都会更新文本
* 初始化完成后，我们将用户重定向到主页。

技巧

由于我们无法在build内部直接重定向到主页,我们使用`WidgetsBinding.instance.addPostFrameCallback()`方法请求Flutter在渲染完成后立即执行方法。

### 例2：应用程序身份验证和注销

对于此示例，我将考虑以下用例：

* 在启动时，如果用户未经过身份验证，则会自动显示“身份验证/注册”页面;
* 在用户认证期间，显示一个圆形进度条;
* 经过身份验证后，用户将被重定向到主页面;
* 在应用程序的任何地方，用户都可以注销;
* 当用户注销时，用户将自动重定向到“身份验证”页面。

当然，很有可能以编程方式处理所有这些，但将所有这些委托给`BLoC`要容易得多。

下图解释了我要解释的解决方案

![](https://www.didierboelens.com/images/bloc_authentication.png)

名为“DecisionPage”的中间页面将负责将用户自动重定向到“身份验证”页面或主页面，具体取决于用户身份验证的状态。当然，此DecisionPage从不显示，也不应被视为页面。

首先要做的是定义事件和状态......

#### AuthenticationEvent

在这个例子中，我只考虑2个事件：

* login：当用户正确验证时发出此事件;
* logout：用户注销时发出的事件。

这里是定义：

```dart
bstract class AuthenticationEvent extends BlocEvent {
  final String name;

  AuthenticationEvent({
    this.name: '',
  });
}

class AuthenticationEventLogin extends AuthenticationEvent {
  AuthenticationEventLogin({
    String name,
  }) : super(
          name: name,
        );
}

class AuthenticationEventLogout extends AuthenticationEvent {}
```

#### AuthenticationState

该类将提供与身份验证过程相关的信息。

对于这个例子，我会考虑：

* 3个标记：
	* isAuthenticated指示身份验证是否完整
	* isAuthenticating以了解我们是否处于身份验证过程的中间
	* hasFailed表示身份验证失败
* 经过身份验证的用户名

这里是源代码：

```dart
class AuthenticationState extends BlocState {
  AuthenticationState({
    @required this.isAuthenticated,
    this.isAuthenticating: false,
    this.hasFailed: false,
    this.name: '',
  });

  final bool isAuthenticated;
  final bool isAuthenticating;
  final bool hasFailed;

  final String name;
  
  factory AuthenticationState.notAuthenticated() {
    return AuthenticationState(
      isAuthenticated: false,
    );
  }

  factory AuthenticationState.authenticated(String name) {
    return AuthenticationState(
      isAuthenticated: true,
      name: name,
    );
  }

  factory AuthenticationState.authenticating() {
    return AuthenticationState(
      isAuthenticated: false,
      isAuthenticating: true,
    );
  }

  factory AuthenticationState.failure() {
    return AuthenticationState(
      isAuthenticated: false,
      hasFailed: true,
    );
  }
}
```

#### AuthenticationBloc

此`BLoC`负责根据事件处理身份验证过程。

这是代码：

```dart
class AuthenticationBloc
    extends BlocEventStateBase<AuthenticationEvent, AuthenticationState> {
  AuthenticationBloc()
      : super(
          initialState: AuthenticationState.notAuthenticated(),
        );

  @override
  Stream<AuthenticationState> eventHandler(
      AuthenticationEvent event, AuthenticationState currentState) async* {

    if (event is AuthenticationEventLogin) {
      // Inform that we are proceeding with the authentication
      yield AuthenticationState.authenticating();

      // Simulate a call to the authentication server
      await Future.delayed(const Duration(seconds: 2));

      // Inform that we have successfuly authenticated, or not
      if (event.name == "failure"){
        yield AuthenticationState.failure();
      } else {
        yield AuthenticationState.authenticated(event.name);
      }
    }

    if (event is AuthenticationEventLogout){
      yield AuthenticationState.notAuthenticated();
    }
  }
}
```

一些解释：

* 当收到事件“AuthenticationEventLogin”时，它会（通过yield）发出一个新状态，告知身份验证正在运行（isAuthenticating = true）。
* 然后它运行身份验证，一旦完成，就会发出另一个状态，告知身份验证已完成。
* 当收到事件“AuthenticationEventLogout”时，它将发出一个新状态，告诉用户不再进行身份验证。

#### AuthenticationPage

正如你将要看到的那样，为了便于解释，此页面非常基本且不会做太多的事情。

这是代码。解释如下：

```dart
class AuthenticationPage extends StatelessWidget {
  ///
  /// Prevents the use of the "back" button
  ///
  Future<bool> _onWillPopScope() async {
    return false;
  }

  @override
  Widget build(BuildContext context) {
    AuthenticationBloc bloc = BlocProvider.of<AuthenticationBloc>(context);
    return WillPopScope(
      onWillPop: _onWillPopScope,
      child: SafeArea(
        child: Scaffold(
          appBar: AppBar(
            title: Text('Authentication Page'),
            leading: Container(),
          ),
          body: Container(
            child:
                BlocEventStateBuilder<AuthenticationEvent, AuthenticationState>(
              bloc: bloc,
              builder: (BuildContext context, AuthenticationState state) {
                if (state.isAuthenticating) {
                  return PendingAction();
                }

                if (state.isAuthenticated){
                  return Container();
                }
                
                List<Widget> children = <Widget>[];

                // Button to fake the authentication (success)
                children.add(
                  ListTile(
                      title: RaisedButton(
                        child: Text('Log in (success)'),
                        onPressed: () {
                            bloc.emitEvent(AuthenticationEventLogin(name: 'Didier'));
                        },
                      ),
                    ),
                );

                // Button to fake the authentication (failure)
                children.add(
                  ListTile(
                      title: RaisedButton(
                        child: Text('Log in (failure)'),
                        onPressed: () {
                            bloc.emitEvent(AuthenticationEventLogin(name: 'failure'));
                        },
                      ),
                    ),
                );

                // Display a text if the authentication failed
                if (state.hasFailed){
                  children.add(
                    Text('Authentication failure!'),
                  );
                }

                return Column(
                  children: children,
                );    
              },
            ),
          ),
        ),
      ),
    );
  }
}
```
说明：

* 第11行：页面获取指向`AuthenticationBloc`的引用
* 第24-70行：它监听发出的`AuthenticationState`：
* 如果身份验证正在进行中，它会显示一个CircularProgressIndicator，告诉用户正在进行某些操作并阻止用户访问该页面（第25-27行）
* 如果验证成功，我们不需要显示任何内容（第29-31行）。
* 如果用户未经过身份验证，则会显示2个按钮以模拟成功的身份验证和失败。
* 当我们点击其中一个按钮时，我们发出一个`AuthenticationEventLogin`事件，以及一些参数（通常由认证过程使用）
* 如果验证失败，我们会显示错误消息（第60-64行）

仅此而已！没有别的事情需要做......是不是很容易？

小贴士

您可能已经注意到，我将页面包装在WillPopScope中。

理由是我不希望用户能够使用Android'后退'按钮，如此示例中所示，身份验证是一个必须的步骤，它阻止用户访问任何其他部分，除非经过正确的身份验证。

#### DecisionPage

如前所述，我希望应用程序根据身份验证状态自动重定向到`AuthenticationPage`或`HomePage`。

以下是此DecisionPage的代码，说明如下：

```dart
class DecisionPage extends StatefulWidget {
  @override
  DecisionPageState createState() {
    return new DecisionPageState();
  }
}

class DecisionPageState extends State<DecisionPage> {
  AuthenticationState oldAuthenticationState;

  @override
  Widget build(BuildContext context) {
    AuthenticationBloc bloc = BlocProvider.of<AuthenticationBloc>(context);
    return BlocEventStateBuilder<AuthenticationEvent, AuthenticationState>(
      bloc: bloc,
      builder: (BuildContext context, AuthenticationState state) {
        if (state != oldAuthenticationState){
          oldAuthenticationState = state;

          if (state.isAuthenticated){
            _redirectToPage(context, HomePage());
          } else if (state.isAuthenticating || state.hasFailed){
  //do nothing
          } else {
            _redirectToPage(context, AuthenticationPage());
          }
        }
        // This page does not need to display anything since it will
        // always remind behind any active page (and thus 'hidden').
        return Container();
      }
    );
  }

  void _redirectToPage(BuildContext context, Widget page){
    WidgetsBinding.instance.addPostFrameCallback((_){
      MaterialPageRoute newRoute = MaterialPageRoute(
          builder: (BuildContext context) => page
        );

      Navigator.of(context).pushAndRemoveUntil(newRoute, ModalRoute.withName('/decision'));
    });
  }
}
```

提醒

为了详细解释这一点，我们需要回到 Flutter处理Pages（= Route）的方式。要处理路由，我们使用导航器，它创建一个叠加层。

这个`Overlay`是一个OverlayEntry栈，每个都包含一个Page。

当我们通过Navigator.of(context) `pushing`,`poping`,替换页面时，后者就会更新它的`Overlay`(栈也被更新了)，也就是被重建了。

当栈被重建的时候，每个OverlayEntry(以及它的内容)也被重建了。

因此,当我们通过Navigator.of(context)进行操作时，所有剩余的页面都会重建！

* 那么，为什么我将它实现为 StatefulWidget？

为了能够响应AuthenticationState的任何更改，此“页面”需要在应用程序的整个生命周期中保持存在。

这意味着，根据上面的提醒，每次Navigator.of（context）完成操作时，都会重建此页面。

因此，它的`BlocEventStateBuilder`也将重建，调用自己的builder方法。

因为此builder负责将用户重定向到与`AuthenticationState`对应的页面，所以如果我们每次重建页面时重定向用户，它会由于持续的重建而不断地重定向。

为了防止这种情况发生，我们只需要记住我们采取行动的最后一个`AuthenticationState`，并且只在收到另一个`AuthenticationState`时采取另一个动作。

* 这是如何运作的？

如上所述，每次发出`AuthenticationState`时，BlocEventStateBuilder都会调用builder。

基于状态标志（isAuthenticated），我们知道我们需要向哪个页面重定向用户。

小技巧

由于我们无法直接从builder重定向到另一个页面，因此我们使用WidgetsBinding.instance.addPostFrameCallback()方法在渲染一完成后就请求Flutter执行方法。

此外，由于我们需要在重定向用户之前删除任何现有页面，除了需要保留在所有情况下的DecisionPage之外，我们使Navigator.of(context).pushAndRemoveUntil(...)来到达此目的。

#### 注销

要让用户注销，您现在可以创建一个“LogOutButton”并将其放在应用程序的任何位置。

此按钮只需要发出`AuthenticationEventLogout()`事件，这将导致以下操作的自动链：

1. 它将由`AuthenticationBloc`处理
2. 反过来会发出一个`AuthentiationState`（isAuthenticated = false）
3. 这将由DecisionPage通过`BlocEventStateBuilder`处理
4. 这会将用户重定向到`AuthenticationPage`

这是这个按钮的代码：

```dart
class LogOutButton extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    AuthenticationBloc bloc = BlocProvider.of<AuthenticationBloc>(context);
    return IconButton(
      icon: Icon(Icons.exit_to_app),
      onPressed: () {
        bloc.emitEvent(AuthenticationEventLogout());
      },
    );
  }
}
```

#### AuthenticationBloc

由于 AuthenticationBloc 需要可用于此应用程序的任何页面，我们还会将其作为MaterialApp的 父级注入，如下所示：

```dart
void main() => runApp(Application());

class Application extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocProvider<AuthenticationBloc>(
      bloc: AuthenticationBloc(),
      child: MaterialApp(
        title: 'BLoC Samples',
        theme: ThemeData(
          primarySwatch: Colors.blue,
        ),
        home: DecisionPage(),
      ),
    );
  }
}
```

## 表单验证

`BLoC`的另一个有趣用途是当您需要验证表单时：

* 根据某些业务规则验证与TextField相关的条目;
* 根据规则显示验证错误消息;
* 根据业务规则自动化widget的可访问性。

我现在要做的一个例子是RegistrationForm，它由3个TextField电子邮件，密码，确认密码）和1个RaisedButton组成，以启动注册过程。

我想要实现的业务规则是：

* 电子邮件需要是有效的电子邮件地址。如果不是，则需要显示一条消息。
* 密码必须有效（必须包含至少8个字符，1个大写，1个小写，1个数字和1个特殊字符。如果无效，则需要显示一条消息。
* 重新输入密码需要符合相同的验证规则并且与密码相同。如果不相同，则需要显示消息。
* 注册按钮只有在所有规则有效时才有效。

###  RegistrationFormBloc
该`BLoC`负责处理验证业务规则，如前所述。

```dart
class RegistrationFormBloc extends Object with EmailValidator, PasswordValidator implements BlocBase {

  final BehaviorSubject<String> _emailController = BehaviorSubject<String>();
  final BehaviorSubject<String> _passwordController = BehaviorSubject<String>();
  final BehaviorSubject<String> _passwordConfirmController = BehaviorSubject<String>();

  //
  //  Inputs
  //
  Function(String) get onEmailChanged => _emailController.sink.add;
  Function(String) get onPasswordChanged => _passwordController.sink.add;
  Function(String) get onRetypePasswordChanged => _passwordConfirmController.sink.add;

  //
  // Validators
  //
  Stream<String> get email => _emailController.stream.transform(validateEmail);
  Stream<String> get password => _passwordController.stream.transform(validatePassword);
  Stream<String> get confirmPassword => _passwordConfirmController.stream.transform(validatePassword)
    .doOnData((String c){
      // If the password is accepted (after validation of the rules)
      // we need to ensure both password and retyped password match
      if (0 != _passwordController.value.compareTo(c)){
        // If they do not match, add an error
        _passwordConfirmController.addError("No Match");
      }
    });

  //
  // Registration button
  Stream<bool> get registerValid => Observable.combineLatest3(
                                      email, 
                                      password, 
                                      confirmPassword, 
                                      (e, p, c) => true
                                    );

  @override
  void dispose() {
    _emailController?.close();
    _passwordController?.close();
    _passwordConfirmController?.close();
  }
}
```

我详细解释一下......

* 我们首先初始化3个`BehaviorSubjec`来处理表单的每个`TextField`的`Streams`。
* 我们暴露了3个Function(String），它将用于接受来自`TextField`的输入。
* 我们暴露了3个Stream<String>，`TextField`将使用它来显示由它们各自的验证产生的潜在错误消息
* 我们暴露了1个Stream<bool>，它将被RaisedButton使用，以根据整个验证结果启用/禁用它。

好的，现在是时候深入了解更多细节了......

* 您可能已经注意到，此类的签名有点特殊。我们来回顾一下吧。

```dart
class RegistrationFormBloc extends Object 
                           with EmailValidator, PasswordValidator 
                           implements BlocBase {
  ...
}
```
with关键字表示此类正在使用MIXINS（=“在另一个类中重用某些类代码的方法”），并且为了能够使用with关键字，该类需要扩展Object类。这些mixin包含分别验证电子邮件和密码的代码。

有关Mixins的更多详细信息，我建议您阅读[Romain Rastel的这篇精彩文章](https://medium.com/flutter-community/dart-what-are-mixins-3a72344011f3)。

#### Validator Mixins

我只会解释EmailValidator，因为PasswordValidator非常相似

首先，是代码

```dart
const String _kEmailRule = r"^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$";

class EmailValidator {
  final StreamTransformer<String,String> validateEmail = 
      StreamTransformer<String,String>.fromHandlers(handleData: (email, sink){
        final RegExp emailExp = new RegExp(_kEmailRule);

        if (!emailExp.hasMatch(email) || email.isEmpty){
          sink.addError('Entre a valid email');
        } else {
          sink.add(email);
        }
      });
}
```

该类暴露了一个`final`方法（“validateEmail”），它是一个`StreamTransformer`。

提醒

`StreamTransformer`的调用方式如下：stream.transform(StreamTransformer）。

`StreamTransformer`通过transform方法从`Stream`引用它的输入。然后它处理此输入，并将转换后的输入重新注入初始`Stream`。

在此代码中，输入的处理包括根据正则表达式进行检查。如果输入与正则表达式匹配，我们只需将输入重新注入流中，否则，我们会向流中注入错误消息。

#### 为什么使用stream.tansform()?

如前所述，如果验证成功，StreamTransformer会将输入重新注入Stream。为什么有用？

以下是与Observable.combineLatest3()相关的解释...此方法在它引用的所有Streams之前不会发出任何值，至少发出一个值。

让我们看看下面的图片来说明我们想要实现的目标。

![](https://www.didierboelens.com/images/bloc_combine.png)

* 如果用户输入电子邮件并且后者经过验证，它将由电子邮件流发出，该电子邮件流将是Observable.combineLatest3()的一个输入;
* 如果电子邮件无效，则会向流中添加错误（并且流中没有值）;
* 这同样适用于密码和重新输入密码;
* 当所有这三个验证都成功时（意味着所有这三个流都会发出一个值），Observable.combineLatest3（）将依次发出一个真正的感谢“（e，p，c）=> true”（见第35行）。

#### 验证2个密码
我在互联网上看到了很多与这种比较有关的问题。存在几种解决方案，让我解释其中的两种。

##### 基本解决方案 - 没有错误消息
第一个解决方案可能是以下一个：

```dart
Stream<bool> get registerValid => Observable.combineLatest3(
                                      email, 
                                      password, 
                                      confirmPassword, 
                                      (e, p, c) => (0 == p.compareTo(c))
                                    );
```

这个解决方案只需验证两个密码，如果它们匹配，就会发出一个值（= true）。

我们很快就会看到，Register按钮的可访问性将取决于registerValid流。

如果两个密码不匹配，则该流不会发出任何值，并且“注册”按钮保持不活动状态，但用户不会收到任何错误消息以帮助他理解原因。


##### 带错误消息的解决方案

另一种解决方案包括扩展confirmPassword流的处理，如下所示：

```dart
Stream<String> get confirmPassword => _passwordConfirmController.stream.transform(validatePassword)
    .doOnData((String c){
      // If the password is accepted (after validation of the rules)
      // we need to ensure both password and retyped password match
      if (0 != _passwordController.value.compareTo(c)){
        // If they do not match, add an error
        _passwordConfirmController.addError("No Match");
      }
    });
```

一旦验证了重新输入密码，它就会被Stream发出，并且使用doOnData，我们可以直接获取此发出的值并将其与密码流的值进行比较。如果两者不匹配，我们现在可以发送错误消息。

### 注册表单

现在让我们先解释一下RegistrationForm：

```dart
class RegistrationForm extends StatefulWidget {
  @override
  _RegistrationFormState createState() => _RegistrationFormState();
}

class _RegistrationFormState extends State<RegistrationForm> {
  RegistrationFormBloc _registrationFormBloc;

  @override
  void initState() {
    super.initState();
    _registrationFormBloc = RegistrationFormBloc();
  }

  @override
  void dispose() {
    _registrationFormBloc?.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Form(
      child: Column(
        children: <Widget>[
          StreamBuilder<String>(
              stream: _registrationFormBloc.email,
              builder: (BuildContext context, AsyncSnapshot<String> snapshot) {
                return TextField(
                  decoration: InputDecoration(
                    labelText: 'email',
                    errorText: snapshot.error,
                  ),
                  onChanged: _registrationFormBloc.onEmailChanged,
                  keyboardType: TextInputType.emailAddress,
                );
              }),
          StreamBuilder<String>(
              stream: _registrationFormBloc.password,
              builder: (BuildContext context, AsyncSnapshot<String> snapshot) {
                return TextField(
                  decoration: InputDecoration(
                    labelText: 'password',
                    errorText: snapshot.error,
                  ),
                  obscureText: false,
                  onChanged: _registrationFormBloc.onPasswordChanged,
                );
              }),
          StreamBuilder<String>(
              stream: _registrationFormBloc.confirmPassword,
              builder: (BuildContext context, AsyncSnapshot<String> snapshot) {
                return TextField(
                  decoration: InputDecoration(
                    labelText: 'retype password',
                    errorText: snapshot.error,
                  ),
                  obscureText: false,
                  onChanged: _registrationFormBloc.onRetypePasswordChanged,
                );
              }),
          StreamBuilder<bool>(
              stream: _registrationFormBloc.registerValid,
              builder: (BuildContext context, AsyncSnapshot<bool> snapshot) {
                return RaisedButton(
                  child: Text('Register'),
                  onPressed: (snapshot.hasData && snapshot.data == true)
                      ? () {
                          // launch the registration process
                        }
                      : null,
                );
              }),
        ],
      ),
    );
  }
}
```

说明：

* 由于RegisterFormBloc仅供此表单使用，因此适合在此处初始化它。
* 每个TextField都包装在StreamBuilder<String>中，以便能够响应验证过程的任何结果（请参阅errorText：snapshot.error）
* 每次对TextField的内容进行修改时，我们都会通过onChanged发送输入到BLoC进行验证：_registrationFormBloc.onEmailChanged（电子邮件输入的情况）
* 对于RegisterButton，后者也包含在StreamBuilder <bool>中。
  * 如果_registrationFormBloc.registerValid发出一个值，onPressed方法将执行某些操作
  * 如果未发出任何值，则onPressed方法将被指定为null，这将取消激活该按钮。
  
仅此而已！表单中没有任何业务规则，这意味着可以更改规则而无需对表单进行任何修改，这非常好！

## Part of

有时，Widget知道它是否是驱动其行为的集合的一部分是有趣的。

对于本文的最后一个实际例子，我将考虑以下场景：

* 一个处理多个商品的应用程序
* 用户可以选择商品并将其放入购物篮篮中;
* 一件商品只能放入购物篮一次;
* 存放在购物篮中的物品可以从购物篮中取出;
* 一旦被移除，有可能把它放回来。

对于此示例，每个商品将显示为一个按钮，该按钮将取决于购物篮篮物品的存在与否。如果不是购物篮的一部分，该按钮将允许用户将其添加到购物篮中。如果是购物篮的一部分，该按钮将允许用户将其从购物篮中取出。

为了更好地说明“Part of”模式，我将考虑以下架构：

* 购物页面将显示所有可能的条目列表;
* 购物页面中的每个商品都会显示一个按钮，用于将商品添加到购物篮或将其移除，具体取决于其在购物篮中是否存在;
* 如果购物页面中的商品被添加到购物篮中，其按钮将自动更新以允许用户将其从购物篮中移除（反之亦然），而无需重建购物页面
* 另一个页面，购物篮，将列出篮子里的所有物品;
* 可以从此页面中删除购物篮中的任何商品。

边注

Part Of这个名字是我给的个人名字。这不是官方名称。

### ShoppingBloc

正如您现在可以想象的那样，我们需要考虑一个专门用于处理所有可能商品列表的`BLoC`，以及购物篮的一部分。

这个BLoC可能如下所示：

```dart
class ShoppingBloc implements BlocBase {
  // List of all items, part of the shopping basket
  Set<ShoppingItem> _shoppingBasket = Set<ShoppingItem>();

  // Stream to list of all possible items
  BehaviorSubject<List<ShoppingItem>> _itemsController = BehaviorSubject<List<ShoppingItem>>();
  Stream<List<ShoppingItem>> get items => _itemsController;

  // Stream to list the items part of the shopping basket
  BehaviorSubject<List<ShoppingItem>> _shoppingBasketController = BehaviorSubject<List<ShoppingItem>>(seedValue: <ShoppingItem>[]);
  Stream<List<ShoppingItem>> get shoppingBasket => _shoppingBasketController;

  @override
  void dispose() {
    _itemsController?.close();
    _shoppingBasketController?.close();
  }

  // Constructor
  ShoppingBloc() {
    _loadShoppingItems();
  }

  void addToShoppingBasket(ShoppingItem item){
    _shoppingBasket.add(item);
    _postActionOnBasket();
  }

  void removeFromShoppingBasket(ShoppingItem item){
    _shoppingBasket.remove(item);
    _postActionOnBasket();
  }

  void _postActionOnBasket(){
    // Feed the shopping basket stream with the new content
    _shoppingBasketController.sink.add(_shoppingBasket.toList());
    
    // any additional processing such as
    // computation of the total price of the basket
    // number of items, part of the basket...
  }

  //
  // Generates a series of Shopping Items
  // Normally this should come from a call to the server
  // but for this sample, we simply simulate
  //
  void _loadShoppingItems() {
    _itemsController.sink.add(List<ShoppingItem>.generate(50, (int index) {
      return ShoppingItem(
        id: index,
        title: "Item $index",
        price: ((Random().nextDouble() * 40.0 + 10.0) * 100.0).roundToDouble() /
            100.0,
        color: Color((Random().nextDouble() * 0xFFFFFF).toInt() << 0)
            .withOpacity(1.0),
      );
    }));
  }
}
```

唯一可能需要解释的方法是_postActionOnBasket（）方法。每次在篮子中添加或删除商品时，我们都需要“刷新”_shoppingBasketController Stream的内容，以便通知所有正在监听此Stream更改的Widgets并能够刷新/重建。

### ShoppingPage
此页面非常简单，只显示所有项目。

```dart
class ShoppingPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    ShoppingBloc bloc = BlocProvider.of<ShoppingBloc>(context);

    return SafeArea(
        child: Scaffold(
      appBar: AppBar(
        title: Text('Shopping Page'),
        actions: <Widget>[
          ShoppingBasket(),
        ],
      ),
      body: Container(
        child: StreamBuilder<List<ShoppingItem>>(
          stream: bloc.items,
          builder: (BuildContext context,
              AsyncSnapshot<List<ShoppingItem>> snapshot) {
            if (!snapshot.hasData) {
              return Container();
            }
            return GridView.builder(
              gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
                crossAxisCount: 3,
                childAspectRatio: 1.0,
              ),
              itemCount: snapshot.data.length,
              itemBuilder: (BuildContext context, int index) {
                return ShoppingItemWidget(
                  shoppingItem: snapshot.data[index],
                );
              },
            );
          },
        ),
      ),
    ));
  }
}
```
说明：

* AppBar显示一个按钮：
  * 显示出现在购物篮中的商品数量
  * 单击时将用户重定向到ShoppingBasket页面
* 项目列表使用GridView构建，包含在StreamBuilder<List<ShoppingItem>>中
* 每个商品对应一个`ShoppingItemWidget`

### ShoppingBasketPage

此页面与ShoppingPage非常相似，只是StreamBuilder现在正在监听由ShoppingBloc暴露的_shoppingBasket流的变体。

### ShoppingItemWidget和ShoppingItemBloc

Part of 模式的依赖于这两个元素的组合：

* ShoppingItemWidget负责：
   * 显示商品和
   * 用于在购物篮中添加商品或从中取出商品的按钮
* ShoppingItemBloc负责告诉ShoppingItemWidget后者是否属于购物篮的一部分。


让我们看看它们如何一起工作......
 
#### ShoppingItemBloc
ShoppingItemBloc由每个ShoppingItemWidget实例化，赋予它“身份”。

此BLoC监听`ShoppingBasket`流的所有变体，并检查特定商品标识是否是购物篮的一部分。

如果是，它会发出一个布尔值（= true），它将被`ShoppingItemWidget`捕获，以确定它是否是购物篮的一部分。

这里是BLoc的代码

```dart
class ShoppingItemBloc implements BlocBase {
  // Stream to notify if the ShoppingItemWidget is part of the shopping basket
  BehaviorSubject<bool> _isInShoppingBasketController = BehaviorSubject<bool>();
  Stream<bool> get isInShoppingBasket => _isInShoppingBasketController;

  // Stream that receives the list of all items, part of the shopping basket
  PublishSubject<List<ShoppingItem>> _shoppingBasketController = PublishSubject<List<ShoppingItem>>();
  Function(List<ShoppingItem>) get shoppingBasket => _shoppingBasketController.sink.add;

  // Constructor with the 'identity' of the shoppingItem
  ShoppingItemBloc(ShoppingItem shoppingItem){
    // Each time a variation of the content of the shopping basket
    _shoppingBasketController.stream
                          // we check if this shoppingItem is part of the shopping basket
                         .map((list) => list.any((ShoppingItem item) => item.id == shoppingItem.id))
                          // if it is part
                         .listen((isInShoppingBasket)
                              // we notify the ShoppingItemWidget 
                            => _isInShoppingBasketController.add(isInShoppingBasket));
  }

  @override
  void dispose() {
    _isInShoppingBasketController?.close();
    _shoppingBasketController?.close();
  }
}
``` 
#### ShoppingItemWidget

此Widget负责：

* 创建ShoppingItemBloc的实例并将其自己的标识传递给BLoC
* 监听ShoppingBasket内容的任何变化并将其转移到BLoC
* 监听ShoppingItemBloc从而知道它是否是篮子的一部分
* 显示相应的按钮（添加/删除），具体取决于它在购物篮中是否存在
* 响应按钮的用户操作
  * 当用户点击添加按钮时，将自己添加到购物篮中
  * 当用户点击删除按钮时，将自己从购物篮中移除。

让我们看看它是如何工作的（解释在代码中给出）。

```dart
class ShoppingItemWidget extends StatefulWidget {
  ShoppingItemWidget({
    Key key,
    @required this.shoppingItem,
  }) : super(key: key);

  final ShoppingItem shoppingItem;

  @override
  _ShoppingItemWidgetState createState() => _ShoppingItemWidgetState();
}

class _ShoppingItemWidgetState extends State<ShoppingItemWidget> {
  StreamSubscription _subscription;
  ShoppingItemBloc _bloc;
  ShoppingBloc _shoppingBloc;

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();

    // As the context should not be used in the "initState()" method,
    // prefer using the "didChangeDependencies()" when you need
    // to refer to the context at initialization time
    _initBloc();
  }

  @override
  void didUpdateWidget(ShoppingItemWidget oldWidget) {
    super.didUpdateWidget(oldWidget);

    // as Flutter might decide to reorganize the Widgets tree
    // it is preferable to recreate the links
    _disposeBloc();
    _initBloc();
  }

  @override
  void dispose() {
    _disposeBloc();
    super.dispose();
  }

  // This routine is reponsible for creating the links
  void _initBloc() {
    // Create an instance of the ShoppingItemBloc
    _bloc = ShoppingItemBloc(widget.shoppingItem);

    // Retrieve the BLoC that handles the Shopping Basket content 
    _shoppingBloc = BlocProvider.of<ShoppingBloc>(context);

    // Simple pipe that transfers the content of the shopping
    // basket to the ShoppingItemBloc
    _subscription = _shoppingBloc.shoppingBasket.listen(_bloc.shoppingBasket);
  }

  void _disposeBloc() {
    _subscription?.cancel();
    _bloc?.dispose();
  }

  Widget _buildButton() {
    return StreamBuilder<bool>(
      stream: _bloc.isInShoppingBasket,
      initialData: false,
      builder: (BuildContext context, AsyncSnapshot<bool> snapshot) {
        return snapshot.data
            ? _buildRemoveFromShoppingBasket()
            : _buildAddToShoppingBasket();
      },
    );
  }

  Widget _buildAddToShoppingBasket(){
    return RaisedButton(
      child: Text('Add...'),
      onPressed: (){
        _shoppingBloc.addToShoppingBasket(widget.shoppingItem);
      },
    );
  }

  Widget _buildRemoveFromShoppingBasket(){
    return RaisedButton(
      child: Text('Remove...'),
      onPressed: (){
        _shoppingBloc.removeFromShoppingBasket(widget.shoppingItem);
      },
    );
  }

  @override
  Widget build(BuildContext context) {
    return Card(
      child: GridTile(
        header: Center(
          child: Text(widget.shoppingItem.title),
        ),
        footer: Center(
          child: Text('${widget.shoppingItem.price} €'),
        ),
        child: Container(
          color: widget.shoppingItem.color,
          child: Center(
            child: _buildButton(),
          ),
        ),
      ),
    );
  }
}
```
### 它是如何工作的？

下图显示了所有部分是如何协同工作。

![](https://www.didierboelens.com/images/bloc_part_of.png)

## 结论

又是一篇长章，我希望我将其变得简略一点，但我一些解释是必要的。

正如我在介绍中所说，我个人在我的开发中经常使用这些“模式”。这让我节省了大量的时间和精力;我的代码更易读，更容易调试。

此外，它有助于将业务与视图分离。

大概率肯定有其他方法可以做到这一点，甚至做得更好。但它对我有用的，这就是我想与你分享的一切。

请继续关注新文章，同时祝你编程愉快。
