---
layout: post
title:  "(译)为你的 Flutter 应用加入交互体验"
date:   2019-04-18 08:55:17 +0800
categories: 
   - Flutter 开发
tags:
    - 状态管理
    - setState
---
[原文链接](https://flutter.dev/docs/development/ui/interactive)

你将学到什么

* 如何对点击做出响应
* 如何创建一个自定义的widget
* stateless widget和 stateful widget的区别

<!--more-->


如何修改您的应用程序以使其对用户输入做出响应？在本教程中，您将为仅包含非交互式widget的应用程序添加交互性。具体来说，您将通过创建自定义有状态widget来对两个无状态widget进行管理的方式修改图标，使其可以被点击。

[布局教程](https://flutter.dev/docs/development/ui/layout/tutorial)已经向你展示了如何为以下的截图创建布局。

![](https://flutter.dev/assets/ui/layout/lakes-2e8707102ca4f56f44e40ce3703606e1600ac1574fe5544c0f2d96f966bed853.jpg)

当应用程序首次启动时，星星是实心的红色，表明这个湖泊以前曾受到喜欢。这颗星旁边的数字表明有41人喜欢这个湖。完成本教程后，点击星形会移除其喜爱状态，用描边的星星替换实心的星星并减少数量。再次点击喜欢湖泊，就会绘制一颗实心的星星并增加数量。

![](https://flutter.dev/assets/ui/favorited-not-favorited-e819c6dfba41b33418caa51282b524f04c12ec5217c41b19cefed685fb4d814b.png)

要实现此目的，您将创建一个包含星星和计数的widget，它们本身就是widget。因为点击星星会改变两个widget的状态，所以widget应该同时管理两个widget。

您可以马上开始 第2步：Subclass StatefulWidget中的代码。如果您想尝试不同的管理状态方法，请跳至管理状态。


为Flutter构建布局显示了如何为以下屏幕截图创建布局。
![img](https://flutter.cn/docs/development/ui/interactive/images/lakes.jpg)

![img](https://flutter.cn/docs/development/ui/interactive/images/favorited-not-favorited.png)

## 有状态和无状态的widget

widget可以是有状态的也可以是无状态的。例如，一个widget当用户与之交互的时候它能发生变化，它就是有状态的。

无状态widget从不发生变化。Icon,IconButton以及Text是无状态widget。无状态widget是StatelessWidget。

有状态widget是动态的，例如，它可以响应用户交互触发的事件或接收数据时改变它的外观。CheckBox,Radio,Slider,InkWell,Form,以及TextField是有状态widget。有状态的 Widget是StatefulWidget。

widget的状态被保存在[State](https://api.flutter.dev/flutter/widgets/State-class.html)对象中，将widget的状态与它的外观分离开来。状态包含了可以改变的值，像是进度条当前的值或是复选框是否选中。当widget的状态发生了变化，状态对象调用了setState(),告知框架对widget进行重新绘制。

## 创建有状态小部件

重点是什么？

* 有状态的widget是由两个类实现的:一个StatefulWidget的子类以及一个State的子类。
* state类包含了widget的可修改的状态以及widget的build()方法。
* 当widget的状态发生变化，state对象调用setState(),告知框架对widget进行重新绘制。

在这个章节中，你会要创建一个自定义的有状态widget。你会把两个无状态的widget-实心红心以及它旁边的数量放到一个自定义的有状态widget中。这个widget管理了由两个孩子widget，IconButton以及Text组成的行。

实现一个自定义的widget需要创建两个类：
* 定义这个widget的一个StatefulWidget的子类
* 包含这个widget状态和定义这个widget的build()方法的State子类。

在这个章节中向你展示了如何为湖泊应用程序构建一个叫做FavoriteWidget的有状态widget。设置好之后，你要做的第一件事是选择如何管理FavoriteWidget的状态。

## 第0步：做好准备

如果您已经在Flutter中的Building Layouts中构建了布局，请跳到下一部分。

*  确保您已设置环境。
*  创建一个基本的“Hello world”Flutter应用程序。
*  用GitHub中的main.dart替换lib/main.dart文件。
*  将pubspec.yaml文件替换为来自GitHub的pubspec.yaml。
*  在项目中创建一个images目录，然后添加lake.jpg。

一旦你有一个连接和启用的设备，或者你已经启动了iOS模拟器（Flutter安装的一部分),你就已经做好准备了。

### 第1步：确定哪个对象管理widget的状态
可以通过多种方法管理widget的状态，但在我们的示例中，widget本身（FavoriteWidget）将管理自己的状态。在此示例中，切换星形是一个独立的操作，不会影响父widget或UI的其余部分，因此widget件可以在内部处理其状态。

在管理状态中，详细了解窗口小部件和状态的分离以及状态的管理方法。

### 第2步：子类StatefulWidget
FavoriteWidget类管理自己的状态，因此它会覆盖createState()以创建State对象。框架在想要构建widget时调用createState()。在此示例中，createState()创建_FavoriteWidgetState的实例，您将在下一步中实现该实例。

```dart
class FavoriteWidget extends StatefulWidget {
  @override
  _FavoriteWidgetState createState() => _FavoriteWidgetState();
}
```

注意：以下划线（_）开头的成员或类是私有的。有关更多信息，请参阅[Dart语言导览](https://www.dartlang.org/guides/language/language-tour)中的[库和可见性](https://www.dartlang.org/guides/language/language-tour#libraries-and-visibility)部分。

### 第3步：子类状态
 _FavoriteWidgetState类存储了可以在widget生命周期内盖板的可修改数据。当应用程序首次启动时，UI会显示一个红色实心星星，表示该湖被喜欢了，并且有41个“喜欢”。 state对象将此信息存储在_isFavorited和_favoriteCount变量中:
 
```dart
class _FavoriteWidgetState extends State<FavoriteWidget> {
  bool _isFavorited = true;
  int _favoriteCount = 41;
  // ···
}
``` 
这个类还定义了一个build()方法。此build()方法创建一个包含红色IconButton和Text的行。widget使用IconButton(而不是Icon)，因为它有一个onPressed属性，用于定义处理tap的回调方法。你接下里会定义回调方法。

```dart
class _FavoriteWidgetState extends State<FavoriteWidget> {
  // ···
  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisSize: MainAxisSize.min,
      children: [
        Container(
          padding: EdgeInsets.all(0),
          child: IconButton(
            icon: (_isFavorited ? Icon(Icons.star) : Icon(Icons.star_border)),
            color: Colors.red[500],
            onPressed: _toggleFavorite,
          ),
        ),
        SizedBox(
          width: 18,
          child: Container(
            child: Text('$_favoriteCount'),
          ),
        ),
      ],
    );
  }
}
```

提示：将文本放在SizedBox中并设置其宽度可防止在文本在40和41之间发生变化时出现明显的“跳跃” - 否则会发生这种情况，因为这些值具有不同的宽度。

_toggleFavorite()方法，会在IconButton被按下的时候被调用，然后会调用setState()。调用setState()是至关重要的，因为这会告诉框架widget的状态以及发生变化了，并且widget需要被重绘。setState()的方法参数切换这两种状态之间的UI：

*  一个星星图标以及数字41
*  一个描边的星星图标以及数字40

```dart
void _toggleFavorite() {
  setState(() {
    if (_isFavorited) {
      _favoriteCount -= 1;
      _isFavorited = false;
    } else {
      _favoriteCount += 1;
      _isFavorited = true;
    }
  });
}
```

### 第4步：将有状态窗口小部件插入窗口小部件树
在应用程序的build()方法中将自定义有状态widget添加到widget树。首先，找到创建图标和文本的代码，然后将其删除：

```dart
// ...
Icon(
  Icons.star,
  color: Colors.red[500],
),
Text('41')
// ...
```

在同一位置，创建有状态小部件：
```dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    Widget titleSection = Container(
      // ...
      child: Row(
        children: [
          Expanded(
            child: Column(
              // ...
          ),
          FavoriteWidget(),
        ],
      ),
    );

    return MaterialApp(
      // ...
    );
  }
}
```

仅此而已！当你热重新加载应用程序时，星形图标现在应该能响应点击了。

### 问题？
如果无法运行代码，请在IDE中查找可能的错误。调试Flutter Apps可能会有所帮助。如果仍然无法找到问题，请在GitHub上针对交互式Lakes示例检查代码。

[lib/main.dart](https://github.com/cfug/flutter.cn/tree/master/src/_includes/code/layout/lakes-interactive/main.dart)

[pubspec.yaml](https://github.com/cfug/flutter.cn/tree/master/src/_includes/code/layout/lakes-interactive/pubspec.yaml) 


[lakes.jpg](https://github.com/flutter/website/tree/master/src/_includes/code/layout/lakes-interactive/images/lake.jpg)


如果您仍有疑问，请参阅[获取支持](https://flutter.cn/community)。

本页的其余部分介绍了可以管理widget状态的几种方法，并列出了其他可用的交互式widget。

## 管理状态

重点是什么？

* 管理状态有不同的方法。
* 作为widget的设计者，你可以选择使用哪种方法。
* 如果有疑问，请从父widget中管理状态开始。

谁对有状态widget的状态进行管理？widget本身？父widget？两者都？另一个对象？答案是......看情况而定。有几种有效的方法可以使您的小部件具有交互性。作为widget设计者，您可以根据你所预期widget的使用方式做出决策。以下是管理状态的最常用方法：

* widget管理它自己的状态
* 父widget管理widget的状态
* 混合搭配的方法

你如何作出使用哪种方法决定？以下原则可以帮助您做出决定：

* 如果所讨论的状态是用户数据，例如复选框的已选中或未选中模式，或滑块的位置，则状态最好由父窗口小部件管理。

* 如果所讨论的状态是美学的，例如动画，那么状态最好由widget本身管理。

如果有疑问，请从父widget中管理状态开始。

我们将通过创建三个简单示例（TapboxA，TapboxB和TapboxC）来举例说明管理状态的不同方法。这些示例的工作方式类似 - 每个都创建了一个容器，当轻敲时，可以在绿色或灰色框之间切换。 _active布尔值确定颜色：绿色表示活动，灰色表示不活动。

![img](https://flutter.cn/docs/development/ui/interactive/images/tapbox-active-state.png)
![img](https://flutter.cn/docs/development/ui/interactive/images/tapbox-inactive-state.png)


这些示例使用GestureDetector捕获Container上的活动。

### widget管理自己的状态


有时，widget在内部管理其状态是最有意义的是。例如，ListView在其内容超出渲染框时自动滚动。大多数使用ListView的开发人员不想管理ListView的滚动行为，因此ListView本身管理其滚动偏移。

_TapboxAState类：

* 管理TapboxA的状态。
* 定义_active布尔值，用于确定框的当前颜色。
* 定义_handleTap（）函数，该函数在点击框时更新_active并调用setState（）函数来更新UI。
* 实现窗口小部件的所有交互行为。

```dart
// TapboxA manages its own state.

//------------------------- TapboxA ----------------------------------

class TapboxA extends StatefulWidget {
  TapboxA({Key key}) : super(key: key);

  @override
  _TapboxAState createState() => _TapboxAState();
}

class _TapboxAState extends State<TapboxA> {
  bool _active = false;

  void _handleTap() {
    setState(() {
      _active = !_active;
    });
  }

  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: _handleTap,
      child: Container(
        child: Center(
          child: Text(
            _active ? 'Active' : 'Inactive',
            style: TextStyle(fontSize: 32.0, color: Colors.white),
          ),
        ),
        width: 200.0,
        height: 200.0,
        decoration: BoxDecoration(
          color: _active ? Colors.lightGreen[700] : Colors.grey[600],
        ),
      ),
    );
  }
}

//------------------------- MyApp ----------------------------------

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      home: Scaffold(
        appBar: AppBar(
          title: Text('Flutter Demo'),
        ),
        body: Center(
          child: TapboxA(),
        ),
      ),
    );
  }
}
```

### 父widget管理widget的状态
通常，父widget要告知其子窗口小部件何时更新，这时候由父widget管理是最有意义的。例如，IconButton允许您将图标视为可点击按钮。 IconButton是一个无状态小部件，因为我们确定父widget需要知道按钮是否已被点击，从而它可以采取适当的操作。

在以下示例中，TapboxB通过回调将其状态导出到其父级。因为TapboxB不管理任何状态，所以它是StatelessWidget的子类。

ParentWidgetState类：

* 管理TapboxB的_active状态。
* 实现_handleTapboxChanged()，即点击框时调用的方法。
* 当状态改变时，调用setState（）来更新UI。

TapboxB类：

* 扩展StatelessWidget，因为所有状态都由其父级处理。
* 检测到点击时，它会通知父母。

```dart
// ParentWidget manages the state for TapboxB.

//------------------------ ParentWidget --------------------------------

class ParentWidget extends StatefulWidget {
  @override
  _ParentWidgetState createState() => _ParentWidgetState();
}

class _ParentWidgetState extends State<ParentWidget> {
  bool _active = false;

  void _handleTapboxChanged(bool newValue) {
    setState(() {
      _active = newValue;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      child: TapboxB(
        active: _active,
        onChanged: _handleTapboxChanged,
      ),
    );
  }
}

//------------------------- TapboxB ----------------------------------

class TapboxB extends StatelessWidget {
  TapboxB({Key key, this.active: false, @required this.onChanged})
      : super(key: key);

  final bool active;
  final ValueChanged<bool> onChanged;

  void _handleTap() {
    onChanged(!active);
  }

  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: _handleTap,
      child: Container(
        child: Center(
          child: Text(
            active ? 'Active' : 'Inactive',
            style: TextStyle(fontSize: 32.0, color: Colors.white),
          ),
        ),
        width: 200.0,
        height: 200.0,
        decoration: BoxDecoration(
          color: active ? Colors.lightGreen[700] : Colors.grey[600],
        ),
      ),
    );
  }
}
```

提示：在创建API时，请考虑对代码所依赖的任何参数使用@required注释。要使用@required，请导入基础库（重新导出Dart的meta.dart库）：

content_copy
import'package：flutter / foundation.dart';

### 混合搭配的方法
对于某些小部件，混合搭配方法最有意义。在这种情况下，有状态窗口小部件管理一些状态，父窗口小部件管理状态的其他方面。

在TapboxC示例中，在点击时，框周围会出现深绿色边框。点击后，边框消失，框的颜色也会改变。 TapboxC将其_active状态导出到其父级，但在内部管理其_highlight状态。此示例有两个State对象，_ParentWidgetState和_TapboxCState。

_ParentWidgetState对象：

* 管理_active状态。
* 实现_handleTapboxChanged（），即点击框时调用的方法。
* 调用setState（）以在点击发生且_active状态更改时更新UI。

_TapboxCState对象：

* 管理_highlight状态。
* GestureDetector监听所有点击事件。当用户点击时，它会添加突出显示（实现为深绿色边框）。当用户释放水龙头时，它会删除突出显示。
* 调用setState（）以在点击，点击或点击取消时更新UI，并且_highlight状态更改。
* 在tap事件上，将状态更改传递给父窗口小部件，以使用窗口小部件属性执行适当的操作。

```dart
//---------------------------- ParentWidget ----------------------------

class ParentWidget extends StatefulWidget {
  @override
  _ParentWidgetState createState() => _ParentWidgetState();
}

class _ParentWidgetState extends State<ParentWidget> {
  bool _active = false;

  void _handleTapboxChanged(bool newValue) {
    setState(() {
      _active = newValue;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      child: TapboxC(
        active: _active,
        onChanged: _handleTapboxChanged,
      ),
    );
  }
}

//----------------------------- TapboxC ------------------------------

class TapboxC extends StatefulWidget {
  TapboxC({Key key, this.active: false, @required this.onChanged})
      : super(key: key);

  final bool active;
  final ValueChanged<bool> onChanged;

  _TapboxCState createState() => _TapboxCState();
}

class _TapboxCState extends State<TapboxC> {
  bool _highlight = false;

  void _handleTapDown(TapDownDetails details) {
    setState(() {
      _highlight = true;
    });
  }

  void _handleTapUp(TapUpDetails details) {
    setState(() {
      _highlight = false;
    });
  }

  void _handleTapCancel() {
    setState(() {
      _highlight = false;
    });
  }

  void _handleTap() {
    widget.onChanged(!widget.active);
  }

  Widget build(BuildContext context) {
    // This example adds a green border on tap down.
    // On tap up, the square changes to the opposite state.
    return GestureDetector(
      onTapDown: _handleTapDown, // Handle the tap events in the order that
      onTapUp: _handleTapUp, // they occur: down, up, tap, cancel
      onTap: _handleTap,
      onTapCancel: _handleTapCancel,
      child: Container(
        child: Center(
          child: Text(widget.active ? 'Active' : 'Inactive',
              style: TextStyle(fontSize: 32.0, color: Colors.white)),
        ),
        width: 200.0,
        height: 200.0,
        decoration: BoxDecoration(
          color:
              widget.active ? Colors.lightGreen[700] : Colors.grey[600],
          border: _highlight
              ? Border.all(
                  color: Colors.teal[700],
                  width: 10.0,
                )
              : null,
        ),
      ),
    );
  }
}
```

替代实现可能已将高亮状态导出到父级，同时保持活动状态为内部，但如果您要求某人使用该分接框，他们可能会抱怨它没有多大意义。开发人员关心该框是否处于活动状态。开发人员可能并不关心如何管理突出显示，并且更喜欢点按框处理这些细节。

## 其它交互式widget
Flutter提供各种按钮和类似的交互式widget。这些小部件中的大多数都实现了Material Design准则，该准则定义了一组具有固定用户界面的组件。

如果您愿意，可以使用GestureDetector在任何自定义小部件中构建交互性。您可以在“管理状态”和“颤动图库”中找到GestureDetector的示例。

注意：Flutter还提供了一组名为Cupertino的iOS风格小部件。

当您需要交互性时，最简单的方法是使用其中一个预制小部件。这是一个部分列表：

### 标准widgets

* [Form](https://api.flutter.dev/flutter/widgets/Form-class.html)
* [FormField](https://api.flutter.dev/flutter/widgets/FormField-class.html)

### Material 组件
* [checkbox](https://api.flutter.dev/flutter/material/Checkbox-class.html)
* [DropdownButton](https://api.flutter.dev/flutter/material/DropdownButton-class.html)
* [FlatButton](https://api.flutter.dev/flutter/material/FlatButton-class.html)
* [FloatingActionButton](https://api.flutter.dev/flutter/material/FloatingActionButton-class.html)
* [IconButton](https://api.flutter.dev/flutter/material/IconButton-class.html)
* [Radio](https://api.flutter.dev/flutter/material/Radio-class.html)
* [RaisedButton](https://api.flutter.dev/flutter/material/RaisedButton-class.html)
* [slider](https://api.flutter.dev/flutter/material/Slider-class.html)
* [Switch](https://api.flutter.dev/flutter/material/Switch-class.html)
* [TextField](https://api.flutter.dev/flutter/material/TextField-class.html)

### 资源列表

在为您的应用添加交互性时，以下资源可能会有所帮助。


* [手势](https://flutter.dev/docs/development/ui/widgets-intro#handling-gestures)，[Flutter Widget框架之旅](https://flutter.dev/docs/development/ui/widgets-intro)中的一个部分

  如何创建按钮并使其响应输入。
  
* [Flutter的手势](https://flutter.dev/docs/development/ui/advanced/gestures)

  Flutter手势机制的描述。
  
* [Flutter API文档](https://api.flutter.dev/)

  所有Flutter库的参考文档。
  
* [Flutter相册](https://github.com/flutter/flutter/tree/master/examples/flutter_gallery)

  演示应用程序展示了许多Material组件和其他Flutter功能。
  
* [Flutter的分层设计](https://www.youtube.com/watch?v=dkyY9WCGMi0)(视频)

  此视频包含有关由状态和无状态widget的信息。由Google工程师Ian Hickson主讲。
