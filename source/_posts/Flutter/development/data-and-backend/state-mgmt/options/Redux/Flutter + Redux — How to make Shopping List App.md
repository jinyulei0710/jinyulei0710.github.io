---
layout: post
title:  "(译)Flutter+Redux- 如何做一个购物清单应用"
date:   2019-04-21 15:16:17 +0800
categories: 
   - Flutter 开发
tags:
    - 状态管理 
    - Redux
---
[原文链接](http://127.0.0.1:4000/flutter/state/management/2019/04/21/Flutter-+-Redux-How-to-make-Shopping-List-App/)

嗨，大家好！在本文中，我想向你展示如何使用Redux创建Flutter应用程序。如果你不知道Flutter是什么，我推荐你阅读我的文章[Flutter - 你可能喜欢它的5个理由](https://medium.com/@pszklarska/flutter-5-reasons-why-you-may-love-it-55021fdbf1aa)。但是，如果你知道Flutter是什么，并且你想要创建一个设计良好，易于测试且具有非常可预测的行为的应用程序 - 那么请继续看下去！

<!--more-->


## 什么是Redux？
首先，让我们首先解释一下Redux是什么。Redux是一种应用程序架构，最初是为JavaScript构建的，现在用于使用反应式框架构建的应用程序(例如React Native或Flutter)。Redux是由Facebook制作的Flux架构的简化版本。但Redux有什么用呢？基本上，你需要知道三件事：

1. 有一个单一的数据源-你的整个应用程序状态只保存在一个地方（被称为仓库）
2. 状态是只读的 - 要修改应用状态，你需要发送一个`action`，然后新状态就被创建出来的
3. 使用纯方法进行修改 - 纯方法（简单的说，它是没有负效应的方法），采用先前的状态和操作，并返回新状态

听起来很酷，但该解决方案的优点是什么？

* 我们控制了状态 - 这意味着我们确切地知道导致状态变化的原因，我们没有重复状态，我们可以轻松地跟踪数据流
* 纯reducer方法很容易对其进行测试 -我们可以传入状态和`action`,然后看结果是否正确
* 应用程序结构清晰 - 我们为`action`，`model`，业务逻辑等提供了不同的层 - 因此你要加新功能的时候，你会知道要放在哪里。
* 对于更复杂的应用程序来说，这是一个很棒的架构 - 您不需要将整个视图树中的状态从父级传递到子级
* 还有一个......

### Redux时间旅行
Redux中有一个很酷的功能 - 🎉时间旅行！使用Redux和适当的工具，您可以随时跟踪应用程序状态，检查实际状态并随时重新创建。看看这个功能到底是什么样的：

![](https://cdn-images-1.medium.com/max/1600/1*KL-z2sJRBYEHrnzFASVvVw.gif)

## Redux相关Widget在一个简单例子上的应用

所有上述规则使数据在Redux是单向流动的。但是这是什么意思？事实上，这都是由多个`action`,多个`reducer`,`store`以及多个`state`完成的。让我们想象显示按钮计数器的应用程序：

![](https://cdn-images-1.medium.com/max/1200/1*vM852avXATOs1_1Mq7Bn7w.png)

* 你的应用程序在开始的有一些状态（点击次数，即0）
* 基于该状态视图被渲染出来了。
* 如果用户点击按钮，则会发送`action`（例如IncrementCounter）
* `reducer`收到动作，它知道先前的状态（计数器0），接收动作（IncrementCounter）并且可以返回新状态（计数器1）
* 你的应用程序有了新状态（计数器1）
* 基于新状态，视图再次被渲染出来

正如你能看到的，通常都是与状态相关的。你有单个应用程序状态，状态对于视图是只读的，并且要创建新的状态的时候你需要发送一个`action`。发送`action`会触发`reducer`创建并发送一个新的应用状态。循环往复。

![](https://cdn-images-1.medium.com/max/1600/1*S_BZUDym3j0Vxjfi8D21Zg.png)Redux数据流

## 使用Redux的购物清单应用示例
让我展示一下Redux在更为复杂例子中的实践。我们将创建一个简单的购物车应用程序。在这个应用程序中，将具有以下功能：

* 添加条目
* 将条目标记为已选中
* 就那么多

该应用程序的样子是这样的:

![](https://cdn-images-1.medium.com/max/1600/1*F_VsjS0EfcjFI5S-fsTvRQ.png)
你可以在Github上看到[该程序的完整代码](https://github.com/pszklarska/FlutterShoppingCart)。

让我们从编码开始吧！ 👇

### 先决条件
在本文中，我不会展示应用程序UI创建的部分。您可以查看这个[实现Redux之前的Shopping List应用程序的代码](https://github.com/pszklarska/FlutterShoppingCart/tree/a8120a23232a05d380384bb377f3994ef65ad221)。我们将从这个基础上，把Redux添加到此应用程序中。

如果你之前从未使用过Flutter，我建议您尝试使用Google推出的[Flutter Codelabs](https://codelabs.developers.google.com/codelabs/flutter/)。

### 设置
要在Flutter上使用Redux运行，您需要向pubspec.yaml文件添加依赖项：

```yaml
flutter_redux: ^0.5.2
```

您可以在[flutter_redux](https://pub.dartlang.org/packages/flutter_redux)包页面上查看最新版本。


### 模型(Model)
我们的应用程序需要管理添加和修改条目，因此我们将使用简单的CartItem模型来存储单个条目的状态。我们的整个应用程序状态将只是多个CartItem组成的列表。如你所见，CartItem只是一个普通的Dart对象

```dart
class CartItem {
  String name;
  bool checked;

  CartItem(this.name, this.checked);
}
```

[注意：这里是这个文件的完整源代码](https://github.com/pszklarska/FlutterShoppingCart/blob/4756839d5749dfa36073e830b208bb45cb5f8874/lib/model/CartItem.dart)


### `Actions`
首先，我们需要声明`action`。 Action基本上是可以被调用以更改应用程序状态的任何意图。在我们的应用程序中，我们将有两个`action`，用于添加和修改条目：

```dart
class AddItemAction {
  final CartItem item;

  AddItemAction(this.item);
}

class ToggleItemStateAction {
  final CartItem item;

  ToggleItemStateAction(this.item);
}

```
[注意：这里是这个文件的完整源代码](https://github.com/pszklarska/FlutterShoppingCart/blob/4756839d5749dfa36073e830b208bb45cb5f8874/lib/redux/actions.dart)


### `Reducers`
然后，我们需要告诉我们的应用程序应该如何处理这些操作。这就是Reducer的用途 - 它们只是采用当前的`State`和`action`，然后它们会创建并返回新的`State`。我们将有两种`Reducer`方法：

```dart
List<CartItem> appReducers(List<CartItem> items, dynamic action) {
  if (action is AddItemAction) {
    return addItem(items, action);
  } else if (action is ToggleItemStateAction) {
    return toggleItemState(items, action);
  } 
  return items;
}

List<CartItem> addItem(List<CartItem> items, AddItemAction action) {
  return List.from(items)..add(action.item);
}

List<CartItem> toggleItemState(List<CartItem> items, ToggleItemStateAction action) {
  return items.map((item) => item.name == action.item.name ?
    action.item : item).toList();
}
```
[注意：这里是这个文件的完整源代码](https://github.com/pszklarska/FlutterShoppingCart/blob/4756839d5749dfa36073e830b208bb45cb5f8874/lib/redux/reducers.dart)

方法`appReducers()`将`action`委托给正确的方法。 addItem()和toggleItemState()方法都返回新列表-这个列表就是我们的新应用程序状态。如你所见，您不应修改当前列表，而是每次都创建新的列表。

### StoreProvider
现在，当我们有了`action`和`reducer`时，我们需要提供存储应用程序状态的位置。它在Redux中称为仓库，它是我们应用程序的唯一数据源。

```dart
void main() {
  final store = new Store<List<CartItem>>(
      appReducers,
      initialState: new List());

  runApp(new FlutterReduxApp(store));
}
```
[注意：这里是这个文件的完整源代码](https://github.com/pszklarska/FlutterShoppingCart/blob/4756839d5749dfa36073e830b208bb45cb5f8874/lib/main.dart)

要创建仓库，我们需要传递`reducers`方法和初始应用程序状态。如果我们创建了仓库，我们必须将它传递给`StoreProvider`，来告诉我们的应用程序它可以被所有请求应用状态的对象使用：

```dart
class FlutterReduxApp extends StatelessWidget {
  final Store<List<CartItem>> store;

  FlutterReduxApp(this.store);

  @override
  Widget build(BuildContext context) {
    return new StoreProvider<List<CartItem>>(
      store: store,
      child: new ShoppingCartApp(),
    );
  }
}
```
[注意：这里是这个文件的完整源代码](https://github.com/pszklarska/FlutterShoppingCart/blob/4756839d5749dfa36073e830b208bb45cb5f8874/lib/main.dart)


在上面的示例中，ShoppingCartApp()是主要的应用程序widget。

### StoreConnector
目前我们拥有除了实际添加和更改条目之外的所有内容都已经做了。那么怎么添加和修改条目呢，我们需要使用`StoreConnector`。这是一种获取仓库并采取一些行动或读取它的状态的方法。

首先，我们想要读取当前数据并在列表中显示：

```dart
class ShoppingList extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return new StoreConnector<List<CartItem>, List<CartItem>>(
      converter: (store) => store.state,
      builder: (context, list) {
        return new ListView.builder(
            itemCount: list.length,
            itemBuilder: (context, position) =>
                new ShoppingListItem(list[position]));
      },
    );
  }
}
```
[注意：这里是这个文件的完整源代码](https://github.com/pszklarska/FlutterShoppingCart/blob/4756839d5749dfa36073e830b208bb45cb5f8874/lib/list/shopping_list.dart)

上面的代码使用`StoreConnector`包装默认的`ListView.builder`。 `StoreConnector`可以获取当前应用程序状态（List<CartItem>）并将其与转换器方法映射到任何内容。出于这种情况的目的，它将是相同的状态（List<CartItem>)，因为我们需要这里的整个列表。

接下来，在builder方法中我们获取到了列表 - 这基本上是来自仓库的CartItems列表，我们可以使用它来构建ListView。

___

好的，很酷 - 我们到读取数据这一步了。现在如何去设置一些数据？

为此，我们还将使用`StoreConnector`，但方式略有不同。

```dart
class AddItemDialog extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return new StoreConnector<List<CartItem>, OnItemAddedCallback>(
        converter: (store) {
      return (itemName) =>
          store.dispatch(AddItemAction(CartItem(itemName, false)));
    }, builder: (context, callback) {
      return new AddItemDialogWidget(callback);
    });
  }
}
typedef OnItemAddedCallback = Function(String itemName);
```
[注意：这里是这个文件的完整源代码](https://github.com/pszklarska/FlutterShoppingCart/blob/4756839d5749dfa36073e830b208bb45cb5f8874/lib/add_item/add_item_dialog.dart)

我们来看看代码吧。我们使用了`StoreConnector`，就像前面的例子一样，但这一次，我们将把它映射到OnItemAddedCallback，而不是将CartItems列表映射到同一个列表中。这样我们就可以将回调传递给AddItemDialogWidget，并在用户添加一些新项时调用它：

```dart
class AddItemDialogWidgetState extends State<AddItemDialogWidget> {
  String itemName;

  final OnItemAddedCallback callback;
  AddItemDialogWidgetState(this.callback);

  @override
  Widget build(BuildContext context) {
    return new AlertDialog(
      ...
      actions: <Widget>[
        ...
        new FlatButton(
            child: const Text('ADD'),
            onPressed: () {
              ...
              callback(itemName);
            })
      ],
    );
  }
}
```

[注意：这里是这个文件的完整源代码](https://github.com/pszklarska/FlutterShoppingCart/blob/4756839d5749dfa36073e830b208bb45cb5f8874/lib/add_item/add_item_dialog.dart)

现在，每次用户按“ADD”按钮时，回调都会发送AddItemAction()事件。

现在，我们可以为切换条目状态做类似的事情：

```dart
class ShoppingListItem extends StatelessWidget {
  final CartItem item;

  ShoppingListItem(this.item);

  @override
  Widget build(BuildContext context) {
    return new StoreConnector<List<CartItem>, OnStateChanged>(
        converter: (store) {
      return (item) => store.dispatch(ToggleItemStateAction(item));
    }, builder: (context, callback) {
      return new ListTile(
        title: new Text(item.name),
        leading: new Checkbox(
            value: item.checked,
            onChanged: (bool newValue) {
              callback(CartItem(item.name, newValue));
            }),
      );
    });
  }
}

typedef OnStateChanged = Function(CartItem item);
```
注意：[这里是这个文件的完整源代码](https://github.com/pszklarska/FlutterShoppingCart/blob/4756839d5749dfa36073e830b208bb45cb5f8874/lib/list/shopping_list_item.dart)

与前面的示例一样，我们使用`StoreConnector`将List<CartItem>映射到OnStateChanged回调。现在每次更改复选框（在onChanged方法中),回调都会触发ToggleItemStateAction事件。

## 摘要
以上就是全部内容！在本文中，我们使用Redux架构创建了一个简单的购物清单应用程序。在我们的应用程序中，我们可以添加一些条目并更改其状态。向此应用程序添加新功能就像添加新action和reducer一样简单。

[在这里](https://github.com/pszklarska/FlutterShoppingCart)，您可以查看此应用程序的完整源代码，包括时间旅行widge：

希望你喜欢这篇文章并敬请期待更多！ 🙌

