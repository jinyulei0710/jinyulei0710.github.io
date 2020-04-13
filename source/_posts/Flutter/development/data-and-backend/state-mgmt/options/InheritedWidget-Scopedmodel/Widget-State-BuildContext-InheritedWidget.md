---
layout: post
title:  "(译)Flutter中的widget、状态（state）、BuildContext、以及Inherited Widget"
date:   2019-04-12 13:51:17 +0800
categories: 
   - Flutter 开发
tags:
    - Inherited Widget
    - State
    - Widget
    - BuildContext
---
[原文链接](https://www.didierboelens.com/2018/06/widget---state---context---inheritedwidget/)

这篇文档涵盖了Flutter应用中的重要概念Widget,状态（state）,BuildContext以及InheritedWidget。特别要引起注意的是InheritedWidget,它是最重要的一个，也是文档较少的widget。

难度：初级

<!--more-->


## 前言

[Widget](https://docs.flutter.io/flutter/widgets/Widget-class.html)，[State](https://docs.flutter.io/flutter/widgets/State-class.html)和[BuildContext](https://docs.flutter.io/flutter/widgets/BuildContext-class.html)的概念是Flutter中每个Flutter开发人员需要完全理解的最重要概念。

尽管文档量巨大，但是并不总能清楚地解释这些概念。

我会用自己的话语来解释这些概念，知道这可能会让一些纯粹主义者感到震惊，但本文的真正目的是弄清楚以下话题：

* [有状态](https://docs.flutter.io/flutter/widgets/StatefulWidget-class.html)和[无状态](https://docs.flutter.io/flutter/widgets/StatelessWidget-class.html)widget之间的区别
* 什么是[BuildContext](https://docs.flutter.io/flutter/widgets/BuildContext-class.html)
* 什么是[状态](https://docs.flutter.io/flutter/widgets/State-class.html)以及如何使用它
* BuildContext与其State对象之间的关系
* [InheritedWidget](https://docs.flutter.io/flutter/widgets/InheritedWidget-class.html)以及在Widgets树中传播信息的方式
* 重建的概念

## 第一部分：概念

### Widget 的概念

在Flutter中，几乎所有的东西都是widget。

将Widget视为可视组件（或与应用程序的可视方面交互的组件）

当您需要构建与布局直接或间接相关的任何内容时，您就正在使用widget。

### Widget树的概念

包含其他widget的widget被称为父widget（或widget容器）。

包含在父widget中的widget称为子widget。

让我们用Flutter自动生成的基本应用程序来说明这一点。

这是简化的代码，仅限于`build`方法：

```dart
@override
Widget build(BuildContext){
    return new Scaffold(
      appBar: new AppBar(
        title: new Text(widget.title),
      ),
      body: new Center(
        child: new Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            new Text(
              'You have pushed the button this many times:',
            ),
            new Text(
              '$_counter',
              style: Theme.of(context).textTheme.display1,
            ),
          ],
        ),
      ),
      floatingActionButton: new FloatingActionButton(
        onPressed: _incrementCounter,
        tooltip: 'Increment',
        child: new Icon(Icons.add),
      ),
    );
}
```


如果我们现在思考这个基础例子，我们将获得以下Widgets树结构（限于代码中存在的Widgets列表）：

![](https://cdn-images-1.medium.com/max/800/1*z_A9htJmE6THrxZeloVRiw.png)

### BuildContext

另一个重要的概念是BuildContext。

BuildContext只不过是对构建的所有widget的树结构中widget的位置的引用。

简而言之，将BuildContext视为Widgets树的一部分，BuildContext是Widget连接到此树的位置。

BuildContext仅属于一个wdiget。

如果widget“A”具有子widget，则widget“A”的BuildContext将成为直接子widget BuildContexts的父BuildContext。

读到此处，很明显BuildContexts被链接在一起的，并且组成了BuildContexts树（父子关系）。

如果我们现在尝试在上图中说明BuildContext的概念，我们获得（仍然是一个非常简化的视图），其中每种颜色代表一个BuildContext，MyApp除外，它是不同的：
![](https://cdn-images-1.medium.com/max/800/1*Tc0kB9YXL4Bj6tonRJlH0w.png)

BuildContext可见性（简化的声明）：

它是仅在其自己的BuildContext或其父BuildContext的BuildContext中可见的东西。

按这句话说的，我们可以从子BuildContext获取BuildContext，并且很容易找到一个父Widget。

有个例子是，思考下 支架 > 中心 > 列 > 文本 ：
context.ancestorWidgetOfExactType（Scaffold）=> 通过从Text上下文转到树结构来返回第一个Scaffold。

从父BuildContext，也可以找到一个后代（=子）Widget，但不建议这样做（我们稍后会讨论）。

### 两种类型的widget

Widget有两种类型：

### 无状态widget

一些可视组件中除了它们自己的配置信息之外不依赖于任何其他信息，该信息在其直接父级构建时提供。

换句话说，这些小部件一旦创建就不必关心任何变化。

这些widget称为无状态widget。

这种widget的典型是Text，Row，Column，Container等，在构建时，我们只是将一些参数传递给它们。

参数可以是装饰，尺寸甚至其它widget的任何内容。不要紧。唯一重要的是这个配置一旦应用，在下一个构建过程之前都不会改变

无状态widget只能在加载/构建widget时绘制一次，这意味着无法根据任何事件或用户操作重绘widget。

### 无状态widget生命周期

这是与无状态小组件相关的代码的典型结构。

如您所见，我们可以将一些额外的参数传递给它的构造函数。但是，请记住，这些参数不会在以后阶段发生变化（变异），只能按原样使用。

```dart
class MyStatelessWidget extends StatelessWidget {
	MyStatelessWidget({
		Key key,
		this.parameter,
	}): super(key:key);

	final parameter;

	@override
	Widget build(BuildContext context){
		return new ...
	}
}
```

即使有另一种方法可以被覆盖（createElement），后者几乎从不被重写。

唯一需要被重写的是构建。

这种无状态widget的生命周期是简单直接的：

* 初始化
* 通过build()渲染

### 有状态的widget

其他一些widget将处理一些在Widget生命周期内会发生变化的内部数据。因此，该数据变得动态。

由此Widget持有的，在此Widget的生命周期中可能会有所不同的数据集，被称为State。

这些widget称为有状态widget。

此类Widget的示例可以是用户可以选择的复选框列表，也可以是根据条件禁用的Button。

## 状态的概念

State定义StatefulWidget实例的“行为”部分。

它持有了就行为和布局布局而言有着相互作用/妨碍的信息。

所有应用于State变更都会强制Widget重建。

## State和BuildContext之间的关系

对于有状态widget，状态与BuildContext相关联的。

此关联是永久性的，State对象永远不会更改其BuildContext。

即使可以在树结构周围移动Widget BuildContext，State仍将与该BuildContext保持关联。

当State与BuildContext关联时，State被视为已挂载。

超重要的：

由于State对象与BuildContext相关联，这意味着State对象不能（直接）通过另一个BuildContext访问！（我们将在稍后讨论这个问题）。

## 有状态widget的生命周期

在介绍了基本概念之后，是时候进一步深入了

这是与Stateful Widget相关的典型代码结构。

由于本文的主要目的是用“变量”数据来解释State的概念，我将故意跳过与某些Stateful Widget overridable方法相关的任何解释，这些方法与此没有特别的关系。

这些可覆盖的方法是didUpdateWidget，deactivate，reassemble。这些将在另一篇文章中讨论。

```dart

class MyStatefulWidget extends StatefulWidget {
	MyStatefulWidget({
		Key key,
		this.parameter,
	}): super(key: key);
	
	final parameter;
	
	@override
	_MyStatefulWidgetState createState() => new _MyStatefulWidgetState();
}

class _MyStatefulWidgetState extends State<MyStatefulWidget> {

	@override
	void initState(){
		super.initState();
		
		// Additional initialization of the State
	}
	
	@override
	void didChangeDependencies(){
		super.didChangeDependencies();
		
		// Additional code
	}
	
	@override
	void dispose(){
		// Additional disposal code
		
		super.dispose();
	}
	
	@override
	Widget build(BuildContext context){
		return new ...
	}
}
```

下图显示了与创建有状态widget相关的操作/调用序列（简化版本）。

在图的右侧，您将注意到流程中的State对象的内部状态。

您还将看到上下文与状态相关联的时刻，从而变为可用（已挂载）。

![](https://cdn-images-1.medium.com/max/800/1*a4Vdp6kZAUqXh2ye_JAEwQ.png)

那么让我们用一些细节来说明：

### initState()

initState()方法是在创建State对象后要调用的第一个方法（在构造函数之后）。

当需要执行额外初始化时，将覆盖此方法。典型的初始化是与动画，控制器有关的。

如果重写此方法，则需要在第一个位置调用super.initState()方法。

在这个方法中，上下文可用但你还不能真正使用它，因为框架还没有完全将状态与它相关联。

一旦initState（）方法完成，State对象就初始化完成了，并且上下文是可用的。

在此State对象的生命周期内不再调用此方法。


didChangeDependencies（）

didChangeDependencies（）方法是要调用的第二个方法。

在此阶段，由于上下文已经可用了，你就可以使用它了。

如果您的Widget链接到InheritedWidget和/或您需要基于BuildContext初始化一些监听器（），则通常会覆盖此方法。

请注意，如果您的widget链接到了InheritedWidget，则每次重建此widget时都会调用此方法。

如果重写此方法，则应首先调用super.didChangeDependencies()。

### build()

build（BuildContext context）方法在didChangeDependencies()和didUpdateWidget(）之后调用。

这是您构建widget(以及可能的所有子树)的地方。

每次State对象更改时（或者当InheritedWidget需要通知“已注册”的widget时）都会调用此方法！

为了强制重建，您可以调用setState（（）{...}）方法。

### dispose()

丢弃widget时调用dispose()方法。

如果需要执行一些清理（例如监听器，控制器......），则重写此方法，然后立即调用super.dispose()。

## 无状态widget还是有状态widget


这是一个许多开发人员需要问自己的问题：“我需要我的Widget是无状态还是有状态的？”

为了回答这个问题，请问问自己：

在我的widget的生命周期中，我是否需要考虑一个会更改的变量，并且更改的时候是否会强制widget重建？

如果问题的答案是肯定的，那么您需要一个有状态widget，否则，您需要一个无状态widget。

举几个例子：

* 一个要展示复选框列表的widget。为了展示复选框，您需要考虑使用一个条目的数组。每个条目都是一个带着标题和状态的对象。如果单击复选框，则会切换相应的条目的状态;

在这种情况下，您需要使用有状态widget来保存条目的状态，以便能够重绘复选框。

* 表单页面。该页面允许用户填写表单widget，并将表单发送到服务器。

在这种情况下，除非在提交表单之前验证表单或执行任何其他操作，无状态widget可能就足够了。

## 有状态的widget有两部分组成的

还记得Stateful小部件的结构吗？有两个部分：

### Widget的主要部分的定义

```dart

class MyStatefulWidget extends StatefulWidget {
    MyStatefulWidget({
		Key key,
		this.color,
	}): super(key: key);
	
	final Color color;

	@override
	_MyStatefulWidgetState createState() => new _MyStatefulWidgetState();
}
```
第一部分“MyStatefulWidget”通常是Widget的公有部分。当您要将其添加到widget树时，可以实例化这部分。此部分在Widget的生命周期内不会发生变化，但可能接受可能由其相应的State实例使用的参数。

请注意，在Widget的第一部分级别定义的任何变量，通常在其生命周期内不会更改。

### widget状态定义

```dart
class _MyStatefulWidgetState extends State<MyStatefulWidget> {
    ...
	@override
	Widget build(BuildContext context){
	    ...
	}
}
```

第二部分“_MyStatefulWidgetState”，是在Widget的生命周期中变化的部分，并强制在发生改变的时候重建Widget的特定实例。

名称开头的“_”字符使该类对.dart文件是私有的。如果需要在.dart文件之外引用此类，请不要使用“_”前缀。

_MyStatefulWidgetState类可以使用widget.{变量的名称}访问存储在MyStatefulWidget中的任何变量。在此示例中：widget.color。

## widget唯一标识-Key

在Flutter中，每个Widget都是唯一标识的。这个唯一标识由构建/渲染时的框架定义。

此唯一标识对应于可选的Key参数。如果省略，Flutter将为您生成一个。

在某些情况下，您可能需要强制使用此密钥，以便可以通过key访问widget。

为此，您可以使用以下辅助类：GlobalKey <T>，LocalKey，UniqueKey或ObjectKey。

GlobalKey确保key在整个应用程序中是唯一的。

强制使用Widget的唯一标识：

```dart
GlobalKey myKey = new GlobalKey();
    ...
    @override
    Widget build(BuildContext context){
        return new MyWidget(
            key: myKey
        );
    }
```
## 第二部分：如何访问状态

如上所述，State被链接到一个Context，并且Context被链接到Widget的一个实例。

### 1. Widget本身

从理论上讲，唯一能够访问状态的是Widget状态本身。

在这种情况下，没有困难。 Widget的`State`类访问其任何变量。

### 2.一个直连的子widget

有时，父widget可能需要访问直连的子widget的状态才能执行特定任务。

在这种情况下，要访问直连的子widget，你需要了解它们。

给某人打电话的最简单方法是通过名字。在Flutter中，每个Widget都有一个唯一的标识，由框架在构建/渲染时确定。

如上所示，您可以使用key参数强制使用Widget的标识。

```dart
...
    GlobalKey<MyStatefulWidgetState> myWidgetStateKey = new GlobalKey<MyStatefulWidgetState>();
    ...
    @override
    Widget build(BuildContext context){
        return new MyStatefulWidget(
            key: myWidgetStateKey,
            color: Colors.blue,
        );
    }
```

一旦确定，父Widget可以通过以下方式访问子widget的状态：

myWidgetStateKey.currentState

让我们考虑一个基础的例子，在用户点击按钮时显示SnackBar。

由于SnackBar是Scaffold的子Widget，它不能直接访问Scaffold body 部分的任何其他孩子（请记住上下文的概念及其层次结构/树结构？）。因此，访问它的唯一方法是通过ScaffoldState，它暴露一个公共方法来显示SnackBar。

```dart
class _MyScreenState extends State<MyScreen> {
        /// the unique identity of the Scaffold
        final GlobalKey<ScaffoldState> _scaffoldKey = new GlobalKey<ScaffoldState>();

        @override
        Widget build(BuildContext context){
            return new Scaffold(
                key: _scaffoldKey,
                appBar: new AppBar(
                    title: new Text('My Screen'),
                ),
                body: new Center(
                    new RaiseButton(
                        child: new Text('Hit me'),
                        onPressed: (){
                            _scaffoldKey.currentState.showSnackBar(
                                new SnackBar(
                                    content: new Text('This is the Snackbar...'),
                                )
                            );
                        }
                    ),
                ),
            );
        }
    }
```

### 3.始祖widget

假设您有一个属于另一个Widget的子树的Widget，如下图所示。

![](https://cdn-images-1.medium.com/max/800/1*aE03QEqgcxdGe3NZXJ3wsA.png)

为了实现这一目标，需要满足3个条件：

### 1.“带状态的widget”（红色）需要暴露其状态

为了公开它的状态，Widget需要在创建时记录它，如下所示：

```dart
class MyExposingWidget extends StatefulWidget {

   MyExposingWidgetState myState;
	
   @override
   MyExposingWidgetState createState(){
      myState = new MyExposingWidgetState();
      return myState;
   }
}
```

### 2.`widget state`需要暴露一些getter / setter

为了让局外人能设置/获取State的属性，Widget State需要通过以下方式授权访问：

* 公有属性（不推荐）
* getter / setter

举个例子：

```dart
class MyExposingWidgetState extends State<MyExposingWidget>{
   Color _color;
	
   Color get color => _color;
   ...
}
```

### 3.“对获得状态感兴趣的widget”（蓝色）需要获得对状态的引用

```dart
class MyChildWidget extends StatelessWidget {
   @override
   Widget build(BuildContext context){
      final MyExposingWidget widget = context.ancestorWidgetOfExactType(MyExposingWidget);
      final MyExposingWidgetState state = widget?.myState;
		
      return new Container(
         color: state == null ? Colors.blue : state.color,
      );
   }
}
```

这个解决方案很容易实现，但widget如何知道它何时需要重建？

对于这个解决方案，它是不知道什么时候重建的。它必须等待重建发生，才能刷新其内容，这不是很方便。

下一节将讨论`Inherited Widget`的概念，它可以解决这个问题。

## InheritedWidget

简而言之，InheritedWidget使在widget树中有效地传播（和共享）信息成为可能。

InheritedWidget是一个特殊的Widget，您可以将其作为另一个子树的父级放在Widgets树中。该子树的所有widget都必须能够与该InheritedWidget公有数据进行交互。

基本

为了解释它，让我们思考以下代码：

```dart
class MyInheritedWidget extends InheritedWidget {
   MyInheritedWidget({
      Key key,
      @required Widget child,
      this.data,
   }): super(key: key, child: child);
	
   final data;
	
   static MyInheritedWidget of(BuildContext context) {
      return context.inheritFromWidgetOfExactType(MyInheritedWidget);
   }

   @override
   bool updateShouldNotify(MyInheritedWidget oldWidget) => data != oldWidget.data;
}
```
此代码定义了一个名为“MyInheritedWidget”的Widget，它作为子树的一部分，目的是在所有widget中的共享一些数据。

如上所述，为了能够传播/共享某些数据，需要将InheritedWidget摆放在widget树的顶部，这解释了所需的子widget是如何传递给InheritedWidget的基础构造器的。

“静态方法MyInheritedWidget(BuildContext context）”允许所有子widget获取最接近上下文的MyInheritedWidget的实例(具体看后面)。

最后，如果数据发生更改，“updateShouldNotify”重写方法，用于告诉InheritedWidget是否必须将通知传递给所有子widget（那些已注册/已订阅的）。

因此，我们需要将它放在如下的树节点层级

```dart
class MyParentWidget... {
   ...
   @override
   Widget build(BuildContext context){
      return new MyInheritedWidget(
         data: counter,
         child: new Row(
            children: <Widget>[
               ...
            ],
         ),
      );
   }
}
```

## 子widget是如何访问InheritedWidget的数据？

在子widget的构建过程中，后者将获得对InheritedWidget的引用，如下所示：

```dart
class MyChildWidget... {
   ...
	
   @override
   Widget build(BuildContext context){
      final MyInheritedWidget inheritedWidget = MyInheritedWidget.of(context);
		
      ///
      /// 从这个时候起，widget就能通过调用inheritedWidget.data，使用 yInheritedWidget暴露的数据
      ///
      return new Container(
         color: inheritedWidget.data.color,
      );
   }
}
```

## 如何在widget之间进行交互？

请思考以下展示widget树结构的图表。

![](https://cdn-images-1.medium.com/max/800/1*g3yJbjYt6jaVuV7DHpKKSA.png)


为了说明一种交互方式，我们作出以下假定：

* “widget A”是一个将项目添加到购物车的按钮;
* “widget B”是一个显示购物车中商品数量的文本;
* “widget C”位于widgetB旁边，是一个内置任何文本的文本widget;
* 我们希望“Widget B”在按下“Widget A”时自动在购物车中显示正确数量的项目，但我们不希望重建“Widget C”

InheritedWidget就是用于这种情况的正确的widget。

示例代码：

先让我写下代码，随后会对其进行解释

```dart
class Item {
   String reference;

   Item(this.reference);
}

class _MyInherited extends InheritedWidget {
  _MyInherited({
    Key key,
    @required Widget child,
    @required this.data,
  }) : super(key: key, child: child);

  final MyInheritedWidgetState data;

  @override
  bool updateShouldNotify(_MyInherited oldWidget) {
    return true;
  }
}

class MyInheritedWidget extends StatefulWidget {
  MyInheritedWidget({
    Key key,
    this.child,
  }): super(key: key);

  final Widget child;

  @override
  MyInheritedWidgetState createState() => new MyInheritedWidgetState();

  static MyInheritedWidgetState of(BuildContext context){
    return (context.inheritFromWidgetOfExactType(_MyInherited) as _MyInherited).data;
  }
}

class MyInheritedWidgetState extends State<MyInheritedWidget>{
  /// List of Items
  List<Item> _items = <Item>[];

  /// Getter (number of items)
  int get itemsCount => _items.length;

  /// Helper method to add an Item
  void addItem(String reference){
    setState((){
      _items.add(new Item(reference));
    });
  }

  @override
  Widget build(BuildContext context){
    return new _MyInherited(
      data: this,
      child: widget.child,
    );
  }
}

class MyTree extends StatefulWidget {
  @override
  _MyTreeState createState() => new _MyTreeState();
}

class _MyTreeState extends State<MyTree> {
  @override
  Widget build(BuildContext context) {
    return new MyInheritedWidget(
      child: new Scaffold(
        appBar: new AppBar(
          title: new Text('Title'),
        ),
        body: new Column(
          children: <Widget>[
            new WidgetA(),
            new Container(
              child: new Row(
                children: <Widget>[
                  new Icon(Icons.shopping_cart),
                  new WidgetB(),
                  new WidgetC(),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class WidgetA extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final MyInheritedWidgetState state = MyInheritedWidget.of(context);
    return new Container(
      child: new RaisedButton(
        child: new Text('Add Item'),
        onPressed: () {
          state.addItem('new item');
        },
      ),
    );
  }
}

class WidgetB extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final MyInheritedWidgetState state = MyInheritedWidget.of(context);
    return new Text('${state.itemsCount}');
  }
}

class WidgetC extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return new Text('I am Widget C');
  }
}
```

### 解释说明

在这个非常基础的例子中，

* _MyInherited是一个InheritedWidget，每次我们通过点击“Widget A”按钮添加一个Item时都会重建
* MyInheritedWidget是一个Widget，其状态包含Items列表。可以通过“（BuildContext context）的静态MyInheritedWidgetState”访问此状态。
* MyInheritedWidgetState公开一个getter（itemsCount）和一个方法（addItem），以便它们可以被widget使用，这是子wiget树的一部分
* 每次我们将一个Item添加到State时，MyInheritedWidgetState都会重建
* MyTree类只是构建一个widget树，将MyInheritedWidget作为树的父级
* WidgetA是一个简单的RaisedButton，当按下它时，从最近的MyInheritedWidget调用addItem方法
* WidgetB是一个简单的文本，显示最接近的MyInheritedWidget级别的项目数

这一切是如何运作？

注册Widget以供以后通知

当子Widget调用MyInheritedWidget.of（context）时，它会调用MyInheritedWidget的以下方法，并传递自己的BuildContext。

```dart
static MyInheritedWidgetState of(BuildContext context) {
  return (context.inheritFromWidgetOfExactType(_MyInherited) as _MyInherited).data;
}
```

在内部，除了简单地返回MyInheritedWidgetState的实例之外，它还将消费者widget订阅了以便更改通知。

在这个景象背后，对这个静态方法的简单调用实际上做了两件事：

* 当对InheritedWidget应用修改时，消费者widget会自动添加到将重建的订户者列表中（此处为_MyInherited）
* _MyInherited小部件（又名MyInheritedWidgetState）中引用的数据将返回给“消费者”。

### 流

由于'Widget A'和'Widget B'都已订阅了InheritedWidget，因此如果对_MyInherited应用了修改，则当单击Widget A的RaisedButton时，操作流程如下（简化版本）：

* 调用MyInheritedWidgetState的addItem方法
* MyInheritedWidgetState.addItem方法将新项添加到List <Item>
* 调用setState（）以重建MyInheritedWidget
* 使用List <Item>的新内容创建_MyInherited的新实例
* _MyInherited记录在参数（数据）中传递的新State
* 作为InheritedWidget，它检查是否需要“通知”“使用者”（答案为是）
* 它遍历整个消费者列表（这里是Widget A和Widget B）并请求他们重建
* 由于Wiget C不是消费者，因此不会重建。

但是，Widget A和Widget B都重建了，而重建Wiget A是没有必要的，因为它没有任何改变。

如何防止这种情况发生？

### 在仍然访问“继承的”小组件时阻止某些小组件重建

Widget A也被重建的原因来自它访问MyInheritedWidgetState的方式。

正如我们之前看到的，调用“context.inheritFromWidgetOfExactType（）”方法的事实自动将Widget订阅到“使用者”列表。

防止此自动订阅同时仍允许Widget A访问MyInheritedWidgetState的解决方案是更改MyInheritedWidget的静态方法，如下所示：

```dart
static MyInheritedWidgetState of([BuildContext context, bool rebuild = true]){
    return (rebuild ? context.inheritFromWidgetOfExactType(_MyInherited) as _MyInherited
                    : context.ancestorWidgetOfExactType(_MyInherited) as _MyInherited).data;
  }
```


通过添加布尔额外参数...

* 如果“rebuild”参数为true（默认情况下），我们使用常规方法（并且Widget将添加到订阅者列表中）
* 如果“rebuild”参数为false，我们仍然可以访问数据，但不使用InheritedWidget的内部实现
因此，要完成解决方案，我们还需要稍微更新Widget A的代码，如下所示（我们添加false额外参数）：

```dart
class WidgetA extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final MyInheritedWidgetState state = MyInheritedWidget.of(context, false);
    return new Container(
      child: new RaisedButton(
        child: new Text('Add Item'),
        onPressed: () {
          state.addItem('new item');
        },
      ),
    );
  }
}
```

就这样，按下它时就不再重建Widget A.

## 针对路由，对话框的特别说明
路由，对话框的上下文是与应用程序绑定的。
这意味着即使在页面A内部您要求显示另一个页面B（例如，在当前的屏幕上），两个屏幕中的任何一个都没有“简单的方法”来关联它们自己的上下文。
页面B了解页面A上下文的唯一方法是从页面A把它作为Navigator.of（context）.push（...。）的参数获取。

## 结论

关于这些主题还有很多话要说......特别是在InheritedWidget上。

在下一篇文章中，我将介绍通知器/监听器的概念，这在使用状态和传送数据的方式中也非常有趣。

感谢您阅读这篇相当长的文章，请继续关注，做一个快乐编码的人。
