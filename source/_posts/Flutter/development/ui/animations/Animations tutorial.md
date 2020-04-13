---
layout: post
title:  "(译)动画教程"
date:   2019-04-24 09:16:17 +0800
categories: 
   - Flutter 开发
tags:
    - Animations
---

[原文链接](https://flutter.dev/docs/development/ui/animations/tutorial)

你会学到什么

* 如何使用动画库中的基础类将动画添加到widget。
* AnimatedWidget和AnimatedBuilder的使用场景的比较。

本教程将向您展示如何在Flutter中构建显式动画。在介绍了动画库中的一些基本概念、类和方法之后，这个教程会带你亲历5个动画相关的例子。这些示例建立在彼此之上，向你介绍了动画库的不同方面。

<!--more-->


Flutter SDK还提供过渡动画，例如[`FadeTransition`](https://api.flutter.dev/flutter/widgets/FadeTransition-class.html)，[`SizeTransition`](https://api.flutter.dev/flutter/widgets/SizeTransition-class.html)和[`SlideTransition`](https://api.flutter.dev/flutter/widgets/SlideTransition-class.html)。这些简单的动画是通过设置起点和终点来触发的。它们比这里所描述的显式动画更容易实现。

## 基本的动画概念和类

重点是什么 

* `Animation`，是Flutter动画库中的一个核心类，插入了那些用来指导动画的值。
* `Animation`对象知道动画的当前状态（例如，它是开始(started)、停止(stopped)还是向前(moving foreard)或反向(in reverse),但它不知道显示在屏幕上的任何事情。
* `AnimationController`管理着`Animation`
* `CurvedAnimation`把过程定义为非线性曲线
* `Tween`在正在执行动画的对象所使用的数据范围之间插入值。例如，Tween可能会定义从红到蓝的一个插值，或者是从0到255的一个插值。
* 使用Listeners和StatusListeners来监控动画状态变化。

Flutter中的动画系统是基于类型化的`Animation`对象的。widget可以通过读取它们当前的状态和监听它们的状态变化把这些动画合并进widget的build方法中，也可以使用动画作为更精心制作的动画的基础，并将这些精心制作的动画传递给其它widget。

### Animation\<double\>

在Flutter中，Animation对象对屏幕上的东西一无所知。`Animation`是一个抽象类，它可以理解其当前值及其状态(已完成或已解除)。一种比较常用的动画类型是Animation\<double>。

一个`Animation`对象会在一定持续时间内连续生成两个值之间的插值数。`Animation`对象的输出可以是线性，曲线，阶梯函数或你其它你能想出的任何其它映射。根据动画对象的控制方式，它可以反向运行，甚至可以在中间切换方向。

动画还可以插入除double之外的类型，例如`Animation\<Color>`或`Animation\<Size>`。

`Animation`对象具有状态。其当前值始终在.value成员中访问。

`Animation`对象对渲染或build()函数一无所知。

### Curved­Animation

[CurvedAnimation](https://api.flutter.dev/flutter/animation/CurvedAnimation-class.html)将动画的过程定义为非线性曲线。

```dart
animation = CurvedAnimation(parent: controller, curve: Curves.easeIn);
```

注意：Curves类定义了许多常用曲线，或者你可以创建自己的曲线。例如：

```dart
import 'dart:math';

class ShakeCurve extends Curve {
  @override
  double transform(double t) => sin(t * pi * 2);
}
```
`CurvedAnimation`和`AnimationController`（会在下一节讲到）都是Animation\<double>类型，因此你可以互换地传递它们。`CurvedAnimation`包装了它所修改的对象-你不必继承AnimationController来实现一条曲线。

### AnimationController
`AnimationController`是一个特殊的Animation对象，只要硬件准备好新帧，它就会生成一个新值。默认情况下，`AnimationController`在给定的持续时间内线性生成从0.0到1.0的数字。例如，此代码创建一个Animation对象，但不启动它运行：

```dart
controller =
    AnimationController(duration: const Duration(seconds: 2), vsync: this);
```

`AnimationController`派生自`Animation\<double>`，因此可以在需要`Animation`对象的任何地方使用它。但是，`AnimationController`还有其他方法来控制动画。例如，使用.forward()方法启动动画。数字的生成与屏幕刷新有关，因此通常每秒生成60个数字。生成每个数字后，每个Animation对象都会调用附加在它上面的监听对象。要为每个子项创建自定义显示列表，请参阅[RepaintBoundary](https://api.flutter.dev/flutter/widgets/RepaintBoundary-class.html)。

在创建`AnimationController`时，您将传递一个`vsync`参数。`vsync`的存在可防止屏幕外动画消耗不必要的资源。通过将`SingleTickerProviderStateMixin`添加到类定义，可以将有状态对象用作vsync。你可以在GitHub上的[animate1](https://github.com/flutter/website/tree/master/examples/animation/animate1/lib/main.dart)中看到这样的示例。

注意：在某些情况下，位置可能会超出`AnimationController`的0.0-1.0范围。例如，fling()函数允许你通过Force对象提供力度，力和位置。位置可以是任何值，因此可以在0.0到1.0范围之外。

`CurvedAnimation`也可以超过0.0到1.0范围，即使`AnimationController`没有超过。根据所选的曲线，CurvedAnimation的输出可以具有比输入更宽的范围。例如，弹性曲线（如Curves.elasticIn）将明显超出或低于默认范围。

### Tween

默认情况下，`AnimationController`对象的范围为0.0到1.0。如果需要不同的范围或不同的数据类型，可以使用Tween将动画配置为插入到不同的范围或数据类型。例如，以下Tween的范围是从-200.0到为0.0：

```dart
tween = Tween<double>(begin: -200, end: 0);
```

Tween是一个只接受开始和结束的无状态对象。 Tween的唯一工作是定义从输入范围到输出范围的映射。输入范围通常为0.0到1.0，但这不是必需的。

Tween继承自Animatable\<T>，而不是Animation<T>。像Animation一样,Animatable不是一定要输出double。例如，ColorTween具体说明两种颜色之间的发展。

```dart
colorTween = ColorTween(begin: Colors.transparent, end: Colors.black54);
```

`Tween`对象不存储任何状态。相反，它提供了evaluate（Animation\<double> animation）方法，该方法将映射函数应用于动画的当前值。可以在.value方法中找到`Animation`对象的当前值。evaluate函数还执行一些内务处理，例如确保在动画值分别为0.0和1.0时返回开始和结束。

#### Tween.animate

要使用Tween对象，请在Tween上调用animate（），并传入控制器对象。例如，以下代码在500ms的过程中生成0到255之间的整数值。

```dart
AnimationController controller = AnimationController(
    duration: const Duration(milliseconds: 500), vsync: this);
Animation<int> alpha = IntTween(begin: 0, end: 255).animate(controller);
```

注意：animate()方法返回的是`Animation`，而不是`Animatable`。

以下示例展示了控制器(controller)，曲线(curve)和补间动画(Tween)：

```dart
AnimationController controller = AnimationController(
    duration: const Duration(milliseconds: 500), vsync: this);
final Animation curve =
    CurvedAnimation(parent: controller, curve: Curves.easeOut);
Animation<int> alpha = IntTween(begin: 0, end: 255).animate(curve);
```

### Animation notifications

Animation对象可以有监听者(Listeners)和状态监听者(StatusListeners)，用addListener()和addStatusListener()来定义。只要动画的值发生变化，就会调用监听器。监听器最常见的行为是调用setState()来引起重建。动画开始，结束，前进或反向时调用`StatusListener`，如AnimationStatus所定义的。下一节有addListener()方法的示例，监视动画的过程(Monitoring the progress of the animation)展示了addStatusListener()的示例。


## Animation 例子

本节会带你亲历5个动画的例子。每个部分都提供了该示例源代码的链接。

### 渲染动画

重点是什么？

* 如何使用addListener()和setState()向widget添加基础动画。
* 每次动画生成一个新数字时，addListener()函数都会调用setState()。
* 如何使用所需的vsync参数定义`AnimatedController`。
* 理解“..addListener”中的“..”语法，也称为Dart的级联表示法。
* 要使类成为私有，请使用下划线(_)开始其名称。


到目前为止，您已经学会了如何随着时间的推移生成一系列数字。没有任何内容呈现在屏幕上。要使用`Animation`对象进行渲染，请将`Animation`对象存储为widget的成员，然后使用其值来决定如何绘制。

想想看以下绘制没有动画的Flutter logo的应用程序：

```dart
import 'package:flutter/material.dart';

void main() => runApp(LogoApp());

class LogoApp extends StatefulWidget {
  _LogoAppState createState() => _LogoAppState();
}

class _LogoAppState extends State<LogoApp> {
  @override
  Widget build(BuildContext context) {
    return Center(
      child: Container(
        margin: EdgeInsets.symmetric(vertical: 10),
        height: 300,
        width: 300,
        child: FlutterLogo(),
      ),
    );
  }
}
```
应用来源：[animate0](https://github.com/flutter/website/tree/master/examples/animation/animate0)

以下展示了相同的代码修改，以使logo动画从无到有增长。定义`AnimationController`时，必须传入vsync对象。vsync参数在AnimationController部分中描述。

与非动画示例的相比的改动，做了高亮显示处理：

![](https://qiantun.oss-cn-beijing.aliyuncs.com/img/WeChat9b8da866d2a9ec4b3372bf11e3597cf3.png)

App源代码:[animate1](https://github.com/flutter/website/tree/master/examples/animation/animate1)

addListener()函数调用setState()，因此每当`Animation`生成一个新数字时，当前帧都标记为脏的(dirty)，这会强制再次调用build()。在build()中，容器会改变大小，因为它的高度和宽度现在使用animation.value而不是硬编码值。动画完成时销毁控制器以防止内存泄漏。

通过这些少量更改，您已经在Flutter中创建了第一个动画！

Dart语言技巧：你可能不熟悉Dart的级联符号 -  ..addListener（）中的两个点。此语法表示使用animate()的返回值调用addListener()方法。请想想看以下示例：

```dart
animation = Tween<double>(begin: 0, end: 300).animate(controller)
  ..addListener(() {
    // ···
  });
```

此代码相当于：
```dart
animation = Tween<double>(begin: 0, end: 300).animate(controller);
animation.addListener(() {
    // ···
  });
```
你可以在[Dart语言之旅](https://www.dartlang.org/guides/language/language-tour)中了解有关级联表示法的更多信息。

### 使用`AnimatedWidget`进行简化

重点是什么？

* 如何使用`AnimatedWidget`帮助类(而不是addListener()和setState()）来创建有动画效果的widget
* 使用`AnimatedWidget`创建一个执行可重用动画的widget。要将widget的过渡分开，请使用`AnimatedBuilder`。
* Flutter API中的AnimatedWidgets示例：AnimatedBuilder，AnimatedModalBarrier，DecoratedBoxTransition，FadeTransition，PositionedTransition，RelativePositionedTransition，RotationTransition，ScaleTransition，SizeTransition，SlideTransition。

`AnimatedWidget`基类允许你从动画代码中分离核心widget代码。`AnimatedWidget`不需要维护State对象来持有动画。添加以下`AnimatedLogo`类：

```dart
class AnimatedLogo extends AnimatedWidget {
  AnimatedLogo({Key key, Animation<double> animation})
      : super(key: key, listenable: animation);

  Widget build(BuildContext context) {
    final Animation<double> animation = listenable;
    return Center(
      child: Container(
        margin: EdgeInsets.symmetric(vertical: 10),
        height: animation.value,
        width: animation.value,
        child: FlutterLogo(),
      ),
    );
  }
}
```

`AnimatedLogo`在绘制自身时使用动画的当前值。

LogoApp仍然管理`AnimationController`和`Tween`，它将`Animation`对象传递给`AnimatedLogo`：

![](https://qiantun.oss-cn-beijing.aliyuncs.com/img/1555986061511.jpg)

App源代码:[animate2](https://github.com/flutter/website/tree/master/examples/animation/animate2)

### 监控动画的过程

重点是什么？

* 使用addStatusListener()通知动画状态的更改，例如启动，停止或反转方向。
* 通过在动画完成或返回其起始状态时反转方向，在无限循环中运行动画。

知道动画何时改变状态通常很有帮助，例如完成，前进或后退。您可以使用addStatusListener()获取此通知。以下代码修改前一个示例，以便它监听状态更改并打印更新。

```dart
class _LogoAppState extends State<LogoApp> with SingleTickerProviderStateMixin {
  Animation<double> animation;
  AnimationController controller;

  @override
  void initState() {
    super.initState();
    controller =
        AnimationController(duration: const Duration(seconds: 2), vsync: this);
    animation = Tween<double>(begin: 0, end: 300).animate(controller)
      ..addStatusListener((state) => print('$state'));
    controller.forward();
  }
  // ...
}
```

运行此代码会产生以下输出：

```dart
AnimationStatus.forward
AnimationStatus.completed
```

接下来，使用addStatusListener()在开头或结尾反转动画。这会产生“呼吸”效果：

![](https://qiantun.oss-cn-beijing.aliyuncs.com/img/1555986414707.jpg)

App源代码:[animate3](https://github.com/flutter/website/tree/master/examples/animation/animate3)

### 使用AnimatedBuilder进行重构

重点是什么？

* `AnimatedBuilder`知道如何渲染过渡。
* `AnimatedBuilder`不知道如何渲染widget，也不管理`Animation`对象。
* 使用`AnimatedBuilder`将动画描述为另一个widget的build方法的一部分。如果你只想使用可重复使用的动画定义widget，请使用`AnimatedWidget`。
* Flutter API中的AnimatedBuilders示例：`BottomSheet`，`ExpansionTile`，`PopupMenu`，`ProgressIndicator`，`RefreshIndicator`，`Scaffold`，`SnackBar`，`TabBar`，`TextField`。

animate3示例中的代码的一个问题是，更改动画需要更改渲染logo的widget。更好的解决方案是将职责分成不同的类：

* 渲染logo
* 定义`Animation`对象
* 渲染过渡

你可以在`AnimatedBuilder`类的帮助下完成此分离。 `AnimatedBuilder`是渲染树中的单独类。与`AnimatedWidget`一样，AnimatedBuilder会自动监听来自`Animation`对象的通知，并根据需要将widget树标记为脏的(dirty)，因此您无需调用addListener()。

[animate4](https://github.com/flutter/website/tree/master/examples/animation/animate4/lib/main.dart)示例的widget树如下所示：

![](https://flutter.dev/assets/ui/AnimatedBuilder-WidgetTree-99e58a8bbf50268bcb0586c276889534bf31e0dc09f17e355a863b04b06a0ec4.png)

从widget树的底部开始，用于渲染logo的代码非常简单：

```dart
class LogoWidget extends StatelessWidget {
  // Leave out the height and width so it fills the animating parent
  Widget build(BuildContext context) => Container(
        margin: EdgeInsets.symmetric(vertical: 10),
        child: FlutterLogo(),
      );
}
```

图中的中间三个块都是在GrowTransition的build()方法中创建的，如下所示。 GrowTransition widget本身是无状态的，并包含定义过渡动画所需的最终变量集。 build()函数创建并返回`AnimatedBuilder`，它将（Anonymous builder）方法和LogoWidget对象作为参数。渲染过渡的工作实际上发生在（Anonymous builder）方法中，该方法创建适当大小的Container以强制LogoWidget缩小以适应。

下面代码中的一个棘手问题是孩子看起来像是指定了两次。事情是这样的，child的外部引用传递给`AnimatedBuilder`，`AnimatedBuilder`将其传递给匿名闭包，然后匿名闭包将该对象用作其子对象。最终结果是AnimatedBuilder插入到渲染树中的两个widget之间。

```dart
class GrowTransition extends StatelessWidget {
  GrowTransition({this.child, this.animation});

  final Widget child;
  final Animation<double> animation;

  Widget build(BuildContext context) => Center(
        child: AnimatedBuilder(
            animation: animation,
            builder: (context, child) => Container(
                  height: animation.value,
                  width: animation.value,
                  child: child,
                ),
            child: child),
      );
}
```

最后，初始化动画的代码看起来与[animate2](https://github.com/flutter/website/tree/master/examples/animation/animate2/lib/main.dart)示例非常相似。 initState()方法创建一个`AnimationController`和一个`Tween`，然后用`animate()`绑定它们。魔术发生在build()方法中，该方法返回一个带LogoWidget作为子项的`GrowTransition`对象，以及一个驱动转换的动画对象。这些是上面要点中列出的三个要素。

![](https://qiantun.oss-cn-beijing.aliyuncs.com/img/1555986662346.jpg)

应用源代码:[animate4](https://github.com/flutter/website/tree/master/examples/animation/animate4)

### 同步动画

重点是什么？

* Curves类定义了一个常用曲线数组，你可以将它们与CurvedAnimation一起使用。

在本节中，您将基于监视动画过程([animate3](https://github.com/flutter/website/tree/master/examples/animation/animate3/lib/main.dart)）的示例构建，该动画使用`AnimatedWidget`连续进行动画制作。想想看在不透明度从透明到不透明设置动画时要进行动画处理的情况。

注意：此示例显示如何在同一动画控制器上使用多个补间，其中每个补间管理动画中的不同效果。它仅用于说明目的。如果你在生产代码中补间不透明度和大小，你可能会使用`FadeTransition`和`SizeTransition`。

每个补间管理动画的一个方面。例如：

```dart
controller =
    AnimationController(duration: const Duration(seconds: 2), vsync: this);
sizeAnimation = Tween<double>(begin: 0, end: 300).animate(controller);
opacityAnimation = Tween<double>(begin: 0.1, end: 1).animate(controller);
```
您可以使用sizeAnimation.value获取大小，使用opacityAnimation.value获取不透明度，但AnimatedWidget的构造函数只接受一个Animation对象。要解决此问题，该示例将创建自己的Tween对象并显式计算值。

更改AnimatedLogo以封装其自己的Tween对象，并且其build()方法在父动画对象上调用Tween.evaluate()以计算所需的大小和不透明度值。

```dart
class AnimatedLogo extends AnimatedWidget {
  // Make the Tweens static because they don't change.
  static final _opacityTween = Tween<double>(begin: 0.1, end: 1);
  static final _sizeTween = Tween<double>(begin: 0, end: 300);

  AnimatedLogo({Key key, Animation<double> animation})
      : super(key: key, listenable: animation);

  Widget build(BuildContext context) {
    final Animation<double> animation = listenable;
    return Center(
      child: Opacity(
        opacity: _opacityTween.evaluate(animation),
        child: Container(
          margin: EdgeInsets.symmetric(vertical: 10),
          height: _sizeTween.evaluate(animation),
          width: _sizeTween.evaluate(animation),
          child: FlutterLogo(),
        ),
      ),
    );
  }
}

class LogoApp extends StatefulWidget {
  _LogoAppState createState() => _LogoAppState();
}

class _LogoAppState extends State<LogoApp> with SingleTickerProviderStateMixin {
  Animation<double> animation;
  AnimationController controller;

  @override
  void initState() {
    super.initState();
    controller =
        AnimationController(duration: const Duration(seconds: 2), vsync: this);
    animation = CurvedAnimation(parent: controller, curve: Curves.easeIn)
      ..addStatusListener((status) {
        if (status == AnimationStatus.completed) {
          controller.reverse();
        } else if (status == AnimationStatus.dismissed) {
          controller.forward();
        }
      });
    controller.forward();
  }

  @override
  Widget build(BuildContext context) => AnimatedLogo(animation: animation);

  @override
  void dispose() {
    controller.dispose();
    super.dispose();
  }
}
```
App 源代码:[animate5](https://github.com/flutter/website/tree/master/examples/animation/animate5)

## 下一步

本教程为你使用Tweens在Flutter中创建动画提供了基础，但还有许多其他类需要探索。你可以研究专门的`Tween`类，特定于`Material Design`的动画，`ReverseAnimation`，共享元素过渡（也称为Hero动画),物理模拟和fling()方法。有关最新的可用文档和示例，请参阅[动画登录页面](https://flutter.dev/docs/development/ui/animations)。