---
layout: post
title:  "(译)交织动画"
date:   2019-04-25 10:16:17 +0800
categories: 
   - Flutter 开发
tags:
    - Animations
---

你将学到什么

* 交织动画由连续或重叠的动画组成。
* 要创建交织动画，请使用多个动画对象。
* 一个 `AnimationController` 控制所有动画。
* 每个 `Animation` 对象在间隔期间指定动画。
* 对于要设置动画的每个属性，请创建一个 `Tween`。

<!--more-->


术语：如果补间或渐变的概念对你来说是新的，请参阅[`Flutter动画教程`](/flutter/animations/2019/04/24/Animations-tutorial/)。

交织的动画是一个直截了当的概念：视觉变化发生在一系列操作中，而不是一次性发生。动画可能是纯粹的顺序动画，在下一个动画之后会发生一个变化，或者它可能部分或完全重叠。它也可能有间隙，没有发生变化。

本指南介绍了如何在Flutter中构建交织动画。


例子
本指南解释了 basic_staggered_animation 示例。你还可以参考更复杂的示例staggered_pic_selection。

[basic_staggered_animation](https://github.com/flutter/website/tree/master/examples/_animation/basic_staggered_animation)

展示单个widget的一系列连续和重叠动画。点击屏幕会开始一个动画，可以改变不透明度，大小，形状，颜色和内边距。

[staggered_pic_selection](https://github.com/flutter/website/tree/master/examples/_animation/staggered_pic_selection)

展示从以三种尺寸之一显示的图像列表中删除图像。此示例使用两个动画控制器：一个用于图像选择/取消选择，另一个用于图像删除。选择/取消选择动画是交织的。(要查看此效果，您可能需要增加timeDilation值。）选择一个最大的图像 - 它会缩小，因为它在蓝色圆圈内显示一个复选标记。接下来，选择一个最小的图像 - 大图像随着复选标记消失而扩展。在大图像完成展开之前，小图像会缩小以显示其复选标记。这种交织行为类似于你在Google相册中看到的行为。

以下视频演示了basic_staggered_animation执行的动画：

<iframe width="772" height="435" src="https://www.youtube.com/embed/0fFvnZemmh8" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

在视频中，您会看到widget的以下动画，该widget以带有略微圆角的边框蓝色方块开始。方块按以下顺序运行更改：

1. 淡入
2. 扩大
3. 向上移动时变得更高
4. 转变为有边界的圆圈
5. 将颜色更改为橙​​色

正向运行之后，动画反向运行。

Flutter新手？本页假定你知道如何使用Flutter的widget创建布局。有关更多信息，请参阅[在Flutter中构建布局](https://flutter.dev/docs/development/ui/layout)。

## 交织动画的基本结构

重点是什么？
* 所有动画都由同一个AnimationController驱动。
* 无论动画实时持续多长时间，控制器的值必须介于0.0和1.0之间。
* 每个动画的间隔介于0.0和1.0之间。
* 对于在间隔中设置动画的每个属性，请创建一个Tween。 Tween指定该属性的开始值和结束值。
* Tween生成一个由控制器管理的Animation对象。


下图显示了 [basic_staggered_animation](https://github.com/flutter/website/tree/master/examples/_animation/basic_staggered_animation) 示例中使用的间隔。你可能会注意到以下特征：

* 不透明度在时间轴的前10％期间发生变化。
* 不透明度的变化与宽度的变化之间存在微小的差距。
* 在最后25％的时间线中没有任何动画。
* 增加内边距使widget看起来升高了。
* 将边框半径增加到0.5，将带圆角的方形转换为圆形。
* 内边距和边界半径变化发生在相同的精确间隔期间，但它们不必。

![](https://flutter.dev/assets/ui/animations/StaggeredAnimationIntervals-ea26220d9436a04e001ebaa4fc0b8dd69496a0274563d0f9df145cc2a5fa8299.png)

要设置动画：

* 创建一个管理所有动画的AnimationController。
* 为每个正在设置动画的属性创建一个Tween。
	* Tween定义了一系列值。
	* Tween的animate方法需要父控制器，并为该属性生成一个Animation。
* 在“动画”曲线属性上指定间隔。

当控制动画的值更改时，新动画的值会更改，从而触发UI更新。

以下代码为width属性创建补间。它构建一个[CurvedAnimation](https://api.flutter.dev/flutter/animation/CurvedAnimation-class.html)，指定一个缓和的曲线。有关其他可用的预定义动画曲线，请参阅[曲线](https://api.flutter.dev/flutter/animation/Curves-class.html)。

```dart
width = Tween<double>(
  begin: 50.0,
  end: 150.0,
).animate(
  CurvedAnimation(
    parent: controller,
    curve: Interval(
      0.125, 0.250,
      curve: Curves.ease,
    ),
  ),
),
```

开始值(begin)和结束值(end)不必是双精度数。下面的代码使用 `BorderRadius.circular`() 为 `borderRadius` 属性（控制方块角的圆度）构建补间。

```dart
borderRadius = BorderRadiusTween(
  begin: BorderRadius.circular(4.0),
  end: BorderRadius.circular(75.0),
).animate(
  CurvedAnimation(
    parent: controller,
    curve: Interval(
      0.375, 0.500,
      curve: Curves.ease,
    ),
  ),
),
```


## 完成的交织动画
与所有交互式widget一样，完整动画由widget对组成：无状态widget和有状态widget。

无状态widget指定补间，定义Animation对象，并提供build()函数，负责构建widget树的动画部分。

有状态widget创建控制器，播放动画，并构建widget树的非动画部分。在屏幕中的任何位置检测到点击时，动画开始。

[basic_staggered_animation的main.dart的完整代码](https://github.com/flutter/website/tree/master/examples/_animation/basic_staggered_animation/main.dart)

## 无状态小部件：StaggerAnimation
在无状态widgetStaggerAnimation中，build()函数实例化AnimatedBuilder--一个用于构建动画的通用widget。 AnimatedBuilder构建一个widget并使用Tweens的当前值配置它。该示例创建一个名为`_buildAnimation()`的函数（执行实际的UI更新），并将其分配给其构建器属性。 AnimatedBuilder监听来自动画控制器的通知，在值发生变化时将widget树标记为脏。对于动画的每个刻度，值都会更新，从而调用`_buildAnimation()`。

```dart
class StaggerAnimation extends StatelessWidget {
  StaggerAnimation({ Key key, this.controller }) :

    // Each animation defined here transforms its value during the subset
    // of the controller's duration defined by the animation's interval.
    // For example the opacity animation transforms its value during
    // the first 10% of the controller's duration.

    opacity = Tween<double>(
      begin: 0.0,
      end: 1.0,
    ).animate(
      CurvedAnimation(
        parent: controller,
        curve: Interval(
          0.0, 0.100,
          curve: Curves.ease,
        ),
      ),
    ),

    // ... Other tween definitions ...

    super(key: key);

  final Animation<double> controller;
  final Animation<double> opacity;
  final Animation<double> width;
  final Animation<double> height;
  final Animation<EdgeInsets> padding;
  final Animation<BorderRadius> borderRadius;
  final Animation<Color> color;

  // This function is called each time the controller "ticks" a new frame.
  // When it runs, all of the animation's values will have been
  // updated to reflect the controller's current value.
  Widget _buildAnimation(BuildContext context, Widget child) {
    return Container(
      padding: padding.value,
      alignment: Alignment.bottomCenter,
      child: Opacity(
        opacity: opacity.value,
        child: Container(
          width: width.value,
          height: height.value,
          decoration: BoxDecoration(
            color: color.value,
            border: Border.all(
              color: Colors.indigo[300],
              width: 3.0,
            ),
            borderRadius: borderRadius.value,
          ),
        ),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      builder: _buildAnimation,
      animation: controller,
    );
  }
}
```


## 有状态widget：StaggerDemo
有状态widget StaggerDemo创建了AnimationController（统一它们的人），指定持续时间为2000毫秒。它播放动画，并构建widget树的非动画部分。在屏幕中检测到点击时动画开始。动画正向播放，然后倒放。

```dart
class StaggerDemo extends StatefulWidget {
  @override
  _StaggerDemoState createState() => _StaggerDemoState();
}

class _StaggerDemoState extends State<StaggerDemo> with TickerProviderStateMixin {
  AnimationController _controller;

  @override
  void initState() {
    super.initState();

    _controller = AnimationController(
      duration: const Duration(milliseconds: 2000),
      vsync: this
    );
  }

  // ...Boilerplate...

  Future<void> _playAnimation() async {
    try {
      await _controller.forward().orCancel;
      await _controller.reverse().orCancel;
    } on TickerCanceled {
      // the animation got canceled, probably because we were disposed
    }
  }

  @override
  Widget build(BuildContext context) {
    timeDilation = 10.0; // 1.0 is normal animation speed.
    return Scaffold(
      appBar: AppBar(
        title: const Text('Staggered Animation'),
      ),
      body: GestureDetector(
        behavior: HitTestBehavior.opaque,
        onTap: () {
          _playAnimation();
        },
        child: Center(
          child: Container(
            width: 300.0,
            height: 300.0,
            decoration: BoxDecoration(
              color: Colors.black.withOpacity(0.1),
              border: Border.all(
                color:  Colors.black.withOpacity(0.5),
              ),
            ),
            child: StaggerAnimation(
              controller: _controller.view
            ),
          ),
        ),
      ),
    );
  }
}
```
