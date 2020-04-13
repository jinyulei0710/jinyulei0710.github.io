---
layout: post
title:  "(译)动画概观"
date:   2019-04-23 15:16:17 +0800
categories: 
   - Flutter 开发
tags:
    - Animations
---

[原文链接](https://flutter.dev/docs/development/ui/animations/overview)

Flutter中的动画系统是基于类型化的[`Animation`](https://api.flutter.dev/flutter/animation/Animation-class.html)对象的。widget可以通过读取它们当前的状态和监听它们的状态变化把这些动画合并进widget的build方法中，也可以使用动画作为更精心制作的动画的基础，并将这些精心制作的动画传递给其它widget。

<!--more-->


## 动画(Animation)

动画系统的主要构建块是[`Animation`](https://api.flutter.dev/flutter/animation/Animation-class.html)类。一个动画代表了一个在动画的生命周期内可变化的特定类型的值。大多数执行动画的widget都会接收一个`Animation`对象作为参数，从中可以读取动画的当前值并在动画上监听值的变化。

### addListener

每当动画的值发生变化时，动画都会通知添加了`addListener`的所有监听者。通常，监听动画的State对象将在其监听回调中调用`setState`，以通知widget系统它需要使用动画的新值进行重建。

这种模式很常见，有两个widget可以帮助widget在动画改变值时重建：[`AnimatedWidget`](https://api.flutter.dev/flutter/widgets/AnimatedWidget-class.html)和[`AnimatedBuilder`](https://api.flutter.dev/flutter/widgets/AnimatedBuilder-class.html)。第一个是`AnimatedWidget`，对于无状态动画widget最有用。要使用`AnimatedWidget`，只需继承它并实现[build](https://api.flutter.dev/flutter/widgets/AnimatedWidget/build.html)函数。第二个是`AnimatedBuilder`，它对于希望将一个动画作为更大的build函数的一部分包含在内的更复杂的widget非常有用。要使用`AnimatedBuilder`，只需构造widget并将其传递给`builder`函数。

### addStatusListener

动画还提供了一个`AnimationStatus`，它指示动画将如何随时间演变。每当动画的状态发生变化时，动画都会通知添加过`addStatusListener`的所有监听器。通常情况下，动画从`dimissed`状态开始，这意味着它们处于其范围的开始。例如,从0.0行进到1.0的动画，当它们的值是0.0的时候会处于`dismissed`状态。然后，动画可能然后运行播放(`forward`)(例如...从0.0到1.0)或者进入倒放(`reverse`)状态(例如..从1.0到0.0)。最终，如果动画到达其范围的末尾(例如，1.0),则动画到达完成(`completed`)状态。

## 动画控制器(AnimationController)

要创建动画，首先要创建一个`AnimationController`。除了作为动画本身能控制动画外，`AnimationController`还可以让你控制动画。例如，你可以告诉控制器播放动画或停止动画。你还可以弹射([fling](https://api.flutter.dev/flutter/animation/AnimationController/fling.html))动画，它使用物理仿真-弹簧来驱动动画。

一旦动画控制器创建完成后，你可以开始基于它构建其他动画。例如，您可以创建反映原始动画的`ReverseAnimation`，但以相反的方向运行（例如，从1.0到0.0）。同样，您可以创建一个`CurvedAnimation`，其值由曲线[`curve`](https://api.flutter.dev/flutter/animation/Curves-class.html)校正。

## 补间动画(Tweens)

要设置超过0.0到1.0间隔的动画，可以使用Tween\<T>，它在其开始值和结束值之间进行插值。许多类型都有特定的Tween子类，它们提供特定于类型的插值。例如，ColorTween在颜色之间插值，而RectTween在rects之间插值。你可以通过创建自己的Tween子类并覆盖其lerp函数来定义自己的插值。

补间本身只定义了如何在两个值之间进行插值。要获取动画当前帧的具体值，还需要动画来确定当前状态。有两种方法可以将补间与动画组合在一起以获得具体值：

1. 您可以在动画的当前值处评估补间。这种方法对于已经监听动画并因此在动画改变值时重建的widget最有用。

2. 您可以根据动画为补间设置动画。animate函数返回一个包含补间的新动画，而不是返回单个值。当你想要将新创建的动画提供给另一个widget时，此方法最有用，该weiget可以读取包含补间的当前值以及监听值的变化。

## 架构

动画实际上是由许多核心构建块构建的。

### 调度器(Scheduler)

SchedulerBinding是一个暴露Flutter调度原语(scheduling primitives)的单例类。

对于此讨论，关键原语(primitive)是帧回调。每次需要在屏幕上显示一个帧时，Flutter的引擎会触发一个“开始帧”回调，调度程序将多路复用到使用scheduleFrameCallback()注册的所有监听器。所有这些回调都给出了框架的官方时间戳，以距离一些时间点的持续时间的形式，由于所有回调都具有相同的时间，因此从这些回调触发的任何动画看起来都是完全同步的，即使它们需要几毫秒才能执行。

### Tickers
[Ticker](https://api.flutter.dev/flutter/scheduler/Ticker-class.html)类钩进调度程序的`scheduleFrameCallback()`机制，从而能在每个tick中调用回调。

Ticker可以被启动和停止。启动时，它返回一个将在停止时解析到的Future。

每个tick，Ticker为回调提供自启动后第一个tick以来的持续时间。

因为代码总是在它们启动后给出相对于第一个刻度的过去的时间，所以代码都是同步的。如果你在两帧之间的不同时间开始三个tick，它们将以相同的开始时间同步，并随后以固定步伐进行tick。

### 仿真(Simulations)
`Simulation`抽象类将相对时间值(过去的时间)映射到double值，并有着一个完成的概念。

原则上，仿真是无状态的，但在实践中，一些仿真（例如，[`BouncingScrollSimulation`](https://api.flutter.dev/flutter/widgets/BouncingScrollSimulation-class.html)和[`ClampingScrollSimulation`](https://api.flutter.dev/flutter/widgets/ClampingScrollSimulation-class.html)）在查询时会不可逆地改变状态。

针对不同的效果，`Simulation`类有[各种具体实现](https://api.flutter.dev/flutter/physics/physics-library.html)。

### 动画特征(Animatables)

[`Animatable`](https://api.flutter.dev/flutter/animation/Animatable-class.html)抽象类将double映射到特定类型的值。

`Animatable`类是无状态的和不可变的。

#### 补间动画(Tween)
`Tween`抽象类将名义上在0.0-1.0范围内的double值映射到类型化值（例如Color或另一个double)。它是一个`Animatable`。

它有一个输出类型(T)的概念，该类型的开始值和结束值，以及在给定输入值的起始值和结束值之间插值（lerp）的方法（名义上在0.0-1.0的范围内）。

`Tween`类是无状态和不可变的。

#### 组合动画(Composing animatables)
将Animatable\<double>（父级）传递给Animatable的chain()方法会创建一个新的Animatable子类，该子类应用父级的映射，然后应用子级的映射。

### 曲线(Curves)
曲线抽象类在名义上将0.0到1.0的范围内的双精度映射到名义上在0.0-1.0范围内的双精度。

曲线类是无状态和不可变的。

### 动画(Animations)
`Animation`抽象类提供给定类型的值，动画方向和动画状态的概念，以及用于注册在值或状态更改时调用的回调的监听器接口。

动画的某些子类具有永不改变的值（[kAlwaysCompleteAnimation](https://api.flutter.dev/flutter/animation/kAlwaysCompleteAnimation-constant.html)，[kAlwaysDismissedAnimation](https://api.flutter.dev/flutter/animation/kAlwaysDismissedAnimation-constant.html)，[AlwaysStoppedAnimation](https://api.flutter.dev/flutter/animation/AlwaysStoppedAnimation-class.html)）;在这些上注册回调没有任何影响，因为永远不会调用回调。

Animation\<double>变量是特殊的，因为它可以用于表示0.0-1.0范围内的双精度，这是Curve和Tween类所期望的输入，以及动画的一些其他子类。

某些动画子类是无状态的，只是将监听器器转发给其父级。有些是非常有状态的。

#### 可组合动画(Composable animations)
大多数动画子类取得的是一个显式的“父级”Animation\<double>。它们是由那个父级驱动的。

`CurvedAnimation`子类将Animation\<double>类(父级)和几个Curve类(前向和反向曲线)作为输入，并使用父级的值作为曲线的输入来确定它的输出。`CurvedAnimation`是不可变的和无状态的。

`ReverseAnimation`子类将Animation\<double>类作为其父级，并反转动画的所有值。它假定父级使用的值一般在0.0-1.0范围内，并返回1.0-0.0范围内的值。父动画的状态和方向也相反。`ReverseAnimation`是不可变的，无状态的。

`ProxyAnimation`子类将Animation\<double>类作为其父级，并仅转发该父级的当前状态。但是,父级可变的。

`TrainHoppingAnimation`子类接受两个父项，并在它们的值交叉时在它们之间切换。

#### 动画控制器(Animation Controllers)

`AnimationController`是一个有状态的Animation\<double>，它使用Ticker来赋予自己生命。它可以启动和停止。每个trck，它需要从启动以来经过的时间并将其传递给`Simulation`来获取一个值。那就是它报告的价值。如果`Simulation`报告当时它已结束，则控制器自行停止。仿真

可以给予动画控制器下限和上限以及持续时间，然后在其中控制动画。

在简单的情况下（使用`forward()`，`reverse()`，`play(`)或`resume()`,动画控制器在给定的时间内只是从下限到上限进行线性插值(反之亦然)。

使用`repeat()`时，动画控制器在给定的持续时间内使用给定边界之间的线性插值，但不会停止。

使用`animateTo()`时，动画控制器会在给定的持续时间内从当前值到给定目标执行线性插值。如果没有给出方法的持续时间，则控制器的默认持续时间和控制器的下限和上限描述的范围被用来确定动画的速度。
仿真
使用`fling()`时，`Force`被用来创建特定的仿真，然后被用来驱动控制器。

使用`animateWith()`时，给定的仿真被用于驱动控制器。

这些方法都返回Ticker提供的future，这个future会在控制器下次停止或更改仿真时解析到。

#### 将动画特征附加到动画
将Animation\<double>(新父级)传递给一个`Animatable`的`animate()`方法会创建一个新的`Animation`子类，其作用类似于`Animatable`，但是从给定的父级动作驱动。