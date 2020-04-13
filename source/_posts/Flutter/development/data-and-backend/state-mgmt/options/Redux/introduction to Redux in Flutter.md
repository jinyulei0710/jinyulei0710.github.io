---
layout: post
title:  "(译)Flutter中的Redux简介"
date:   2019-04-20 15:16:17 +0800
categories: 
    - Flutter 开发
tags:
    - 状态管理 
    - Redux
---

[原文链接](https://blog.novoda.com/introduction-to-redux-in-flutter/)

Redux是一种单向数据流架构，可以轻松开发，维护和测试应用程序。在这篇文章中，我向你解释如何在Flutter中使用Redux架构编写移动应用程序。

Flutter是一项非常有趣的技术。在许多情况下，它被证明无论是对独立开发者还是软件公司来说都非常有用。Google目前正在使用它，所以未来可期。Flutter在考虑快速迭代的同时，它又是开发人员友好的，并且它是多平台的。让我们来看看Flutter的高层级架构，然后转到Redux。

<!--more-->


## Flutter widgets

在Flutter中，每个UI组件都是一个[`Widget`](https://flutter.io/widgets-intro/#introduction)，您可以使用包含其他多个Widget的多个Widget组合整个应用程序UI，甚至你的应用程序类也是一个Widget。

widget要么是`StatelessWidget`,要么是`StatefulWidget`：

### FLUTTER STATELESSWIDGET

`statelessWidget`是一个非常简单的Widget，它没有任何可变状态，因此需要使用不同的参数重新创建它以显示不同的数据。

StatelessWidget的一个示例可以是待办事项列表中的一行：此Widget将获取待办事项文本和“完成”标志作为构造函数参数，然后显示它们。更改待办事项文本或完成标志需要你创建另一个StatelessWidget。

### FLUTTER STATEFULWIDGET

在基于某些可变状态构建UI时，StatefulWidget非常有用。在这种情况下，每次状态发生变化时都会重新创建Widget，从而反映其Widget树中的状态更改。除了创建StatelessWidget也具有的Widget树之外，StatefulWidget还必须创建将持有可变状态的State对象。

StatefulWidget的一个示例是待办事项列表项的容器：此容器列表将扩展StatefulWidget，并且它会创建一个ToDoState。这个ToDoState是待办事项列表所在的地方，以及我们创建Widget树的地方（即ListView）。一旦有用户操作（当添加，删除待办事项等）时，我们将使用State对象中的setState更新列表，这将重建Widget树，显示添加(会删除)条目后的效果。

这样可以很好地分离那些不会改变的东西。但它也有其缺点：

* 当必须在多个页面中共享状态时，它需要位于应用程序Widget中，然后必须将其传递到每个屏幕的Widget树，这个实现需要样板代码。
* 当用户操作必须修改共享状态时，多个widget会紧密耦合，因为必须在wodget树中传达操作。
* 紧密耦合的widget不可重用，如果你计划对UI进行修改的时候，则可能很难修改widget树。
为了抵消和避免这些缺点，我们可以转向Redux。

## Redux

Redux是一种具有单向数据流的体系结构，可以轻松开发易于测试和维护的应用程序。

![](https://blog.novoda.com/content/images/2018/03/redux-architecture-overview.png)

在Redux中有一个Store，它包含一个用于表示整个应用程序的状态的State对象。每个应用程序事件（来自用户或外部）都表示为一个Action，它被分派到Reducer方法。此Reducer根据收到的操作更新Store的状态。每当通过Store推送新状态时，都会重新创建视图以反映更改。

使用Redux，大多数组件都是分离的，因此UI修改非常容易。此外，唯一的业务逻辑位于Reducer方法中。 Reducer方法是接受Action和当前应用程序状态，并返回一个新的State对象，因此可以直接测试，因为我们可以编写一个单元测试来设置一个初始状态并检查Reducer是否返回新的修改过的的状态。

## Redux Middleware

以上乍一看似乎很简单，但是当应用程序必须执行某些异步操作时会发生什么，例如从外部API加载数据？这就是人们想出一个名为`Middleware`的新组件的原因。

`Middleware`是一个可以在Action到达Reducer之前对它进行处理的组件。它接收当前应用程序状态和已发送出来的Action，它可以运行一些代码（通常是[负效应](https://en.wikipedia.org/wiki/Side_effect_(computer_science))），例如与第三方API或数据源通信。最后，`Middleware`可能会决定发送原始动作，发送一个不同的动作，或者不做任何其他动作。你可以在[此处](https://redux.js.org/advanced/middleware)了解有关`Middleware`的更多信息。

使用`Middleware`之后，上面的图表看起会是这样的：

![](https://blog.novoda.com/content/images/2018/03/redux-architecture-overview-middleware.png)

## Flutter中的Redux

将所有这些都带到Flutter，我们有着两个非常有用的包，使用这两个包之后，在Flutter应用程序中实现Redux变得非常简单方便：

* [`redux`](https://pub.dartlang.org/packages/redux)：`redux`包添加了在Dart中使用Redux的所有必要组件，即`Store`，`Reducer`和`Middleware`。
* [`flutter_redux`](https://pub.dartlang.org/packages/flutter_redux)：这是一个Flutter特有的包，它在redux库之上提供了额外的组件，这些组件对于在Flutter中实现Redux非常有用，例如：`StoreProvider`（应用程序用来把Store提供给所有那些需要它的widget的基础widget),`StoreBuilder`(从`StoreProvider`接收`Store`的`Widget`)和`StoreConnector`（可以用来代替`StoreBuilder`的非常有用的`Widget`，因为你可以将`Store`转换为`ViewModel`来构建Widget树，每当`Store`中的状态被修改，`StoreConnector`将被重建）。

## 让我看看代码
我已经创建了一个基础的待办事项列表应用程序来演示上面讨论的概念。让我们来看看重要的部分。

首先，main.dart文件（这是我们应用程序的入口点）定义了一个由初始状态，Reducer方法和`Middleware`组成的应用程序`Store`对象。然后它使用StoreProvider包装MaterialApp对象，该StoreProvider获取了Store并能够将其传递给需要它的后代Widget：

```dart
void main() => runApp(ToDoListApp());

class ToDoListApp extends StatelessWidget {
  final Store<AppState> store = Store<AppState>(
    appReducer, /* Function defined in the reducers file */
    initialState: AppState.initial(),
    middleware: createStoreMiddleware(),
  );

  @override
  Widget build(BuildContext context) => StoreProvider(
        store: this.store,
        child: MaterialApp(
          // Omitting some boilerplate here
          home: ToDoListPage(title: 'Flutter Demo Home Page'),
        ),
      );
}
```
AppState类包含待办事项列表和一个用来决定是否显示TextField以添加新的条目的字段：

```dart
class AppState {
  final List<ToDoItem> toDos;
  final ListState listState;

  AppState(this.toDos, this.listState);

  factory AppState.initial() => AppState(List.unmodifiable([]), ListState.listOnly);
}

enum ListState {
  listOnly, listWithNewItem
}
```

为了显示待办事项列表，我们定义了一个ViewModel类，它包含我们需要显示的数据的视图特定表示，以及用户可以执行的操作。此ViewModel从Store创建：

```dart
class _ViewModel {
  final String pageTitle;
  final List<_ItemViewModel> items;
  final Function onNewItem;
  final String newItemToolTip;

  _ViewModel(this.pageTitle, this.items, this.onNewItem, this.newItemToolTip, this.newItemIcon);

  factory _ViewModel.create(Store<AppState> store) {
    List<_ItemViewModel> items = store.state.toDos
        .map((ToDoItem item) => /* Omitting some boilerplate here */)
        .toList();

    return _ViewModel('To Do', items, () => store.dispatch(DisplayListWithNewItemAction()), 'Add new to-do item', Icons.add);
  }
}
```
现在我们可以使用`ViewModel`类来显示待办事项列表。请注意，我们将`Widgets`包装在`StoreConnector`中，允许我们从`Store`创建`ViewModel`并使用`ViewModel`构建我们的UI：

```dart
class ToDoListPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) => StoreConnector<AppState, _ViewModel>(
        converter: (Store<AppState> store) => _ViewModel.create(store),
        builder: (BuildContext context, _ViewModel viewModel) => Scaffold(
              appBar: AppBar(
                title: Text(viewModel.pageTitle),
              ),
              body: ListView(children: viewModel.items.map((_ItemViewModel item) => _createWidget(item)).toList()),
              floatingActionButton: FloatingActionButton(
                onPressed: viewModel.onNewItem,
                tooltip: viewModel.newItemToolTip,
                child: Icon(viewModel.newItemIcon),
              ),
            ),
      );
}
```
在上面的代码中，我们定义当用户按下“Add”按钮时，我们将发送DisplayListWithNewItemAction类型的动作，这表示我们需要修改应用程序状态，以便我们显示TextField，让用户创建一个新的待办事项条目。动作类定义为：

```dart
class DisplayListWithNewItemAction {}
```

这是针对此操作的Reducer：

```dart
AppState appReducer(AppState state, action) => AppState(toDoListReducer(state.toDos, action), listStateReducer(state.listState, action));

final Reducer<List<ToDoItem>> toDoListReducer = // Boilerplate ignored
final Reducer<ListState> listStateReducer = combineReducers<ListState>([
  TypedReducer<ListState, DisplayListOnlyAction>(_displayListOnly),
  TypedReducer<ListState, DisplayListWithNewItemAction>(_displayListWithNewItem),
]);

ListState _displayListOnly(ListState listState, DisplayListOnlyAction action) => ListState.listOnly;

ListState _displayListWithNewItem(ListState listState, DisplayListWithNewItemAction action) => ListState.listWithNewItem;
```

这是一个简单的例子，但演示了上面解释的概念。这个待办事项列表应用程序的完整源代码可以在[Github](https://github.com/xrigau/todo_demo_flutter_redux)中找到。
___

总而言之，在Flutter应用程序中使用此体系结构可以使所有关注点明确定义并彼此分离。然而，只在小项目中尝试了这个（由于Flutter仍处于测试阶段），我想看看它是否能在大型项目中很好地扩展，因为整个应用程序有一个状态会让我觉得你会用较小的`State`对象组成这个`State`对象。

在大型项目中使用Redux进行重构，可能按功能（/home，/settings等）在子文件夹中划分Redux组件，虽然我见过的大多数示例是按组件类型划分的子文件夹（/middleware，/actions等）。

最后不得不提的是，我真的想尝试使用AngularDart构建一个简单的Web前端，甚至可以使用Redux添加桌面支持，然后看看在移动客户端，网站和桌面客户端之间可以重用多少代码。

感谢您的关注。如果您有任何意见，建议或想讨论，那么请在Twitter上与我交谈💬