---
layout: post
title:  "(译)简单的共享状态管理"
date:   2019-04-16 16:43:17 +0800
categories: 
   - Flutter 开发
tags:
    - 状态管理
---

[原文链接](https://flutter.dev/docs/development/data-and-backend/state-mgmt/simple)

现在你已了解[状态管理中的声明式编程思维](Start-thinking-declaratively.html)以及[短时状态和共享](Differentiate-between-ephemeral-state-and-app-state.html)，你已经做好了解简单的应用程序状态管理的准备。

在此篇文章中，我们将使用 scoped_model 包。如果你是 Flutter 的新手并且没有充分的理由选择其他方法（Redux，Rx，钩子等），这可能是你刚开始应该使用的方法。 scoped_model很容易理解，它不会使用太多代码。它还使用适用于所有其他方法的概念。

<!--more-->

也就是说，如果你来自其它响应式框架中的强大状态管理的背景，那么你将在[接下来的文章](https://flutter.cn/docs/development/data-and-backend/state-mgmt/options)上找到包和教程。

## 我们的例子

为了便于说明，请思考以下简单应用。

该应用程序有三个独立的页面：登录提示，目录和购物车（分别由 MyLoginScreen，MyCatalog 和 MyCart 这三个 widget 表示）。它可能是一个购物应用程序，但你可以在一个简单的社交网络应用程序中想象相同的结构（把购物车替代为照片墙，购物车替代为喜爱的人）。

目录屏幕包括自定义应用栏（MyAppBar）和由许多列表项（MyListItems）组成滑动视图。

这是可视化为 widget 树的应用程序。

![img](https://flutter.cn/assets/development/data-and-backend/state-mgmt/simple-widget-tree-19cb2528c56ef04924de364b4d0e08b73f4bcf7231aad0d6bc0eb1919e543fb9.png)


所以我们至少有6个 Widget 子类。它们中的许多 widget 会需要访问属于其他地方的状态。例如，每个 MyListItem 都希望能够添加到购物车。它可能还想查看它显示的项目是否已经在购物车中。

这就把我们带到了第一个问题：我们应该把购物车的当前状态放在哪里？


### 向上移动状态

在 Flutter 中，将状态保存在使用它的 widget 之上是有意义的。

为什么？在像 Flutter 这样的声明性框架中，如果要更改 UI，则必须重新构建它。 没有简单的方法可以使用 MyCart.updateWith(somethingNew)。 换句话说，通过调用其上的方法很难从外部强制更改 widget。 即使你能做到这一点，你也会跟框架作对，而不是让它帮你。

```dart
// BAD: DO NOT DO THIS
void myTapHandler() {
  var cartWidget = somehowGetMyCartWidget();
  cartWidget.updateWith(item);
}
```

即使你使用上述代码，你也必须在 MyCart widget 中处理以下内容：

```dart
// BAD: DO NOT DO THIS
Widget build(BuildContext context) {
  return SomeWidget(
    // The initial state of the cart.
  );
}

void updateWith(Item item) {
  // Somehow you need to change the UI from here.
}
```

你需要考虑 UI 的当前状态并将新数据应用于它。以这种方式很难避免错误。

在 Flutter 中，每次内容更改时都会构造一个新 widget。 你使用 MyCart（内容）（构造函数），而不是 MyCart.updateWith（somethingNew）（方法调用）。 因为你只能在父节点的构建方法中构建新的 widget，如果你想改变内容，它需要存在于 MyCart 的父级或更高级别。

```dart
// GOOD
void myTapHandler(BuildContext context) {
  var cartModel = somehowGetMyCartModel(context);
  cartModel.add(item);
}
```

现在，MyCart 只有一个代码路径来构建任何版本的 UI。

```dart
// GOOD
Widget build(BuildContext context) {
  var cartModel = somehowGetMyCartModel(context);
  return SomeWidget(
    // Just construct the UI once, using the current state of the cart.
    // ···
  );
}
```

在我们的示例中，内容需要存在于 MyApp 中。 每当它发生变化时，它都会从上面重建MyCart（稍后会详细介绍）。 因此，MyCart 不需要担心生命周期 - 它只是声明了为任何给定内容显示的内容。 当更改时，旧的 MyCart widget 将消失，并完全被新的 widget 取代。


![img](https://flutter.cn/assets/development/data-and-backend/state-mgmt/simple-widget-tree-with-cart-088b22c4ef4e4389a1cababaceaadcd36ba3de37613080942885263c36e29595.png)

当我们说 widget 是不可变的时，这就是我们的意思。 它们没有改变 - 它们被取代了。

现在我们知道了购物车状态的位置，让我们看看如何访问它。


## 访问状态

当用户单击目录中的某个条目时，它会添加到购物车中。 但由于购物车位于MyListItem之上，我们该如何做？


一个简单的选项是提供 MyListItem 单击时可以调用的回调。 Dart的方法是甲级对象，因此您可以以任何方式传递它们。 因此，在 MyCatalog 中，你可以拥有以下内容：

```dart
@override
Widget build(BuildContext context) {
  return SomeWidget(
    // Construct the widget, passing it a reference to the method above.
    MyListItem(myTapCallback),
  );
}

void myTapCallback(Item item) {
  print('user tapped on $item');
}
```

这样可以正常工作，但对于你需要从许多不同的地方进行修改的 app 状，你必须传递大量的回调 - 这很快就会使人变老。

幸运的是，Flutter 有 widget 机制为他们的后代提供数据和服务（换句话说，不仅仅是他们的孩子，而是它们下面的任何 widget）。 正如您对 Flutter 所期望的那样，所有东西是Widget™，这些机制只是特殊类型的widget-  `InheritedWidget`，`InheritedNotifier`，`InheritedModel`等。 我们不会在这里介绍那些，因为它们对我们想要做的事情来说有点过于底层。

相反，我们将使用一个适用于底层 widget 的包，但却易于使用。 它叫做 scoped_model。

使用 `scoped_model`，你无需担心回调或 `InheritedWidgets`。 但你确实需要理解3个概念：

* Model
* ScopedModel
* ScopedModelDescedant

在 scoped_model 中，Model封装了你的应用程序状态。 对于非常简单的应用程序，你可以使用单个模型。在复杂的，你会有几个模型。

在我们的购物应用示例中，我们希望在模型中管理购物车的状态。 我们创建了一个扩展 Model 的新类。 像这样：

```dart
class CartModel extends Model {
  /// Internal, private state of the cart.
  final List<Item> _items = [];

  /// An unmodifiable view of the items in the cart.
  UnmodifiableListView<Item> get items => UnmodifiableListView(_items);

  /// The current total price of all items (assuming all items cost $1).
  int get totalPrice => _items.length;

  /// Adds [item] to cart. This is the only way to modify the cart from outside.
  void add(Item item) {
    _items.add(item);
    // This call tells [Model] that it should rebuild the widgets that
    // depend on it.
    notifyListeners();
  }
}
```

唯一特定于 Model 的代码是对 notifyListeners() 的调用。 只要模型发生变化，可能会改变应用程序的 UI，就可以调用此方法。 CartModel 中的其他所有内容都是模型本身及其业务逻辑。

模型不依赖于 Flutter 中的任何高级类，因此它很容易测试（你甚至不需要使用 widget 测试)。 例如，这是 CartModel 的简单单元测试：

```dart
test('adding item increases total cost', () {
  final cart = CartModel();
  final startingPrice = cart.totalPrice;
  cart.addListener(() {
    expect(cart.totalPrice, greaterThan(startingPrice));
  });
  cart.add(Item('Dash'));
});
```

但是当与 scoped_model 包的其余部分一起使用时，Model 真正开始有意义。

## ScopedModel

ScopedModel是为其后代提供Model实例的widget。

我们已经知道在哪里放置它：在需要访问它的widget之上。 对于CartModel，这意味着在MyCart和MyCatalog之上。

你不希望将 ScopedModel 放在高于必要的位置（因为你不想污染范围）。但在我们的案例中，MyCart 和 MyCatalog 之上唯一的小部件是 MyApp。

```dart
void main() {
  final cart = CartModel();

  // You could optionally connect [cart] with some database here.

  runApp(
    ScopedModel<CartModel>(
      model: cart,
      child: MyApp(),
    ),
  );
}
```

请注意，我们正在创建 ScopedModel \<CartModel>（读作：“CartModel的ScopedModel”）。 scoped_model 包依赖于类型来查找正确的模型，<CartModel>部分清楚地说明了我们在这里提供的类型。

如果要提供多个模型，则需要嵌套 ScopedModel：

```dart
ScopedModel<SomeOtherModel>(
  model: myOtherModel,
  child: ScopedModel<CartModel>(
    model: cart,
    child: MyApp(),
  ),
)
```

## ScopedModelDescendant

现在 CartModel 通过顶部的 ScopedModel \<CartModel> 声明提供给我们应用程序中的widget，我们可以开始使用它。

这是通过 ScopedModelDescendant widget 完成的。

```dart
return ScopedModelDescendant<CartModel>(
  builder: (context, child, cart) {
    return Text("Total price: ${cart.totalPrice}");
  },
);
```

我们必须指定我们想要访问的模型的类型。 在这种情况下，我们需要 CartModel，因此我们编写 ScopedModelDescendant \<CartModel>。 如果未指定泛型（\<CartModel>），则scoped_model 包将无法帮助您。 如上所述，scoped_model 基于类型，没有类型，它不知道你想要什么。

ScopedModelDescendant widget 唯一必需的参数是构建器。 Builder 是一个在模型更改时调用的函数。(换句话说，当你在模型中调用 notifyListeners() 时，将调用所有相应ScopedModelDescendant widget 的所有构建器方法。）

构建器被调用的时候使用了三个属性。第一个是 context，你可以在每个构建方法中获得它。

第二个属性是child，它用于优化。 如果你的 ScopedModelDescendant 下有一个大的widget 子树，它在模型更改时不会更改，你可以构造它一次并通过构建器获取它。

```dart
return ScopedModelDescendant<CartModel>(
  builder: (context, child, cart) => Stack(
        children: [
          // Use SomeExpensiveWidget here, without rebuilding every time.
          child,
          Text("Total price: ${cart.totalPrice}"),
        ],
      ),
  // Build the expensive widget here.
  child: SomeExpensiveWidget(),
);
```
构建器函数的第三个参数是模型。这就是我们首先要求的。 你可以使用模型中的数据来定义 UI 在任何给定点的外观。

最佳做法是将 ScopedModelDescendant widget 尽可能深入树中。 你不希望重建UI的大部分内容只是因为某些细节发生了变化。

```dart
// DON'T DO THIS
return ScopedModelDescendant<CartModel>(
  builder: (context, child, cart) {
    return HumongousWidget(
      // ...
      child: AnotherMonstrousWidget(
        // ...
        child: Text('Total price: ${cart.totalPrice}'),
      ),
    );
  },
);
```
而是：

```dart
// DO THIS
return HumongousWidget(
  // ...
  child: AnotherMonstrousWidget(
    // ...
    child: ScopedModelDescendant<CartModel>(
      builder: (context, child, cart) {
        return Text('Total price: ${cart.totalPrice}');
      },
    ),
  ),
);
```

### ScopedModel.of

有时，你并不真正需要模型中的数据来更改 UI，但你仍然需要访问它。 例如，ClearCart 按钮希望允许用户从购物车中删除所有内容。 它不需要显示购物车的内容，只需要调用clear() 方法即可。

我们可以使用 ScopedModelDescendant \<CartModel>，但这样做会很浪费。 我们要求框架重建一个不需要重建的widget。

对于这个用例，我们可以使用 ScopedModel.of

```dart
ScopedModel.of<CartModel>(context).add(item);
```

在调用 notifyListeners 时，在构建方法中使用上述行不会导致此 widget 重建。

注意：你还可以使用 ScopedModelDescendant \<CartModel>（Builder：myBuilder，rebuildOnChange：false），但这样做更长，并且需要你定义构建器函数。


## 把它们放在一起

你可以查看本文中介绍的[示例](https://github.com/filiph/samples/tree/scoped-model-shopper/model_shopper)。 如果你想要更简单的东西，你可以看到使用[scoped_model构建简单的Counter应用程序时的样子](https://github.com/flutter/samples/tree/master/scoped_model_counter)。

当你准备好自己玩 scoped_model 时，不要忘记首先将它的依赖项添加到 pubspec.yaml。

```dart
name: my_name
description: Blah blah blah.

# ...

dependencies:
  flutter:
    sdk: flutter

  scoped_model: ^1.0.0

dev_dependencies:
  # ...
```

现在你可以导入'package：scoped_model / scoped_model.dart'; 并开始构建。


