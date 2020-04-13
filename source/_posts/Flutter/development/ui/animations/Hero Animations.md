---
layout: post
title:  "(译)Hero 动画"
date:   2019-04-25 09:16:17 +0800
categories: 
   - Flutter 开发
tags:
    - Animations
    - navigation
---
[原文链接](https://flutter.dev/docs/development/ui/animations/hero-animations)

你会学到什么

* `hero`指的是在路由之间飞行的widget。
* 使用Flutter的`Hero`widget创建一个`hero`动画。
* 将`hero`从一个路由飞到另一个路由。
* 动画将`hero`的形状从圆形变为矩形，同时将其从一个路由飞到另一个路由。
* Flutter中的`Hero`widget 实现了一种动画风格，通常称为共享元素过渡或共享元素动画。

<!--more-->


你可能已经多次看过`Hero`动画了。例如，路由显示表示待售物品的缩略图列表。选择条目会将其飞到一个新路由，其中包含更多详细信息和“购买”按钮。将图像从一个路由飞到另一个路由在Flutter中称为`hero`动画，尽管相同的运动有时被称为共享元素过渡。

你可能想要观看介绍`Hero widget`的这一分钟视频：

<iframe width="560" height="315" src="https://www.youtube.com/embed/Be9UH1kXFDw" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

本指南演示了如何构建标准`Hero`动画，以及在飞行过程中将图像从圆形变换为方形的`Hero`动画。

示例：本指南提供以下链接中每种`hero`动画样式的示例。

* [Standard hero animation code](https://flutter.dev/docs/development/ui/animations/hero-animations#standard-hero-animation-code)
* [Radial hero animation code](https://flutter.dev/docs/development/ui/animations/hero-animations#radial-hero-animation-code)

Flutter新手？本页假定您知道如何使用Flutter的widget创建布局。有关更多信息，请参阅[在Flutter中构建布局](https://flutter.dev/docs/development/ui/layout)。

术语：[路由](/flutter/navigation/2019/04/24/Navigate-to-a-new-screen-and-back/)描述Flutter应用程序中的页面或屏幕。

你可以使用`Hero` widget在Flutter中创建此动画。当`hero`以动画形式从源路由到目标路由时，目标路由会淡入视图。通常，`hero`是UI的一小部分，如图像，两条路由都有共同之处。从用户的角度来看，`hero`在路由之间“飞翔”。本指南介绍如何创建以下`hero`动画：

`标准hero动画`

标准`hero`动画将`hero`从一个路由飞到一个新路由，通常降落在不同的位置并且具有不同的大小。

以下视频(以低速录制)展示了一个典型示例。在路由中心点击脚蹼将它们飞到新的蓝色路由的左上角，尺寸较小。在蓝色路由中点击脚蹼（或使用设备的回到前一个路由的手势）将脚蹼飞回原始路由。

<iframe width="910" height="512" src="https://www.youtube.com/embed/CEcFnqRDfgw" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

`径向hero动画`

在径向hero动画中，当`hero`在路由之间飞行时，其形状呈现为从圆形变为矩形。

以下视频(以低速录制)显示了径向`hero`动画的示例。开始时，路由底部会出现一排三个圆形图像。点击任何圆形图像会将图像飞到一个以正方形形状显示的新路由上。点击方形图像会使`hero`回到原始路由，显示为圆形。

<iframe width="910" height="512" src="https://www.youtube.com/embed/LWKENpwDKiM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


在进入到特定于标准或径向`hero`动画的部分之前，阅读`hero`动画的基本结构以学习如何构建`hero`动画代码，并在表象背后了解Flutter如何执行`hero`动画的。

## `hero`动画的基本结构

重点是什么？

* 在不同的路由中使用两个widget小部件，但使用匹配的标签来实现动画。
* Navigator管理着包含应用程序路由的堆栈。
* 从导航器的堆栈中`push`路由或`pop`路由会触发动画。
* Flutter框架计算一个[矩形补间](https://api.flutter.dev/flutter/animation/RectTween-class.html)，用于定义`hero`从源路由到目的地路由的边界。在飞行过程中，`hero`被移动到应用程序的覆盖层，以便它出现在两个路由的顶部。

术语：如果补间或补间的概念对你来说是新的，请参阅[Flutter中的动画教程](/flutter/animations/2019/04/24/Animations-tutorial/)。

使用两个Hero widget实现`hero`动画：一个描述源路由中的widget，另一个描述目标路由中的小widget。从用户的角度来看，`hero`似乎是共享的，只有程序员才需要了解这个实现细节。

关于对话框的注意事项：`hero`从一个`PageRoute`路由飞到另一个。对话框（例如，用showDialog()显示 ),使用的是PopupRoutes，它们不是`PageRoute`。至少目前，你无法在Dialog上实现hero动画。有关进一步的开发(以及可能的解决方法)，请[查看此问题](https://github.com/flutter/flutter/issues/10667)。

`hero`动画代码具有以下结构：

1. 定义一个起始`hero`小部件，称为源`hero`。`hero`指定其图形表示(通常是图像)和识别标记，并且在源路由定义的当前显示的widget树中。
2. 定义一个结束`hero`小部件，称为目标`hero`。此`hero`还指定其图形表示，以及与源`hero`相同的标记。两个hero widget都必须使用相同的标记创建，通常是表示基础数据的对象。为了获得最佳效果，`hero`应该拥有几乎相同的widget树。
3. 创建包含目标`hero`的路由。目标路由定义动画结尾处存在的widget树。
4. 通过在导航器堆栈上按下目标路由来触发动画。导航器的push和pop操作会触发每对`hero`的`hero`动画，并在源和目标路由中使用匹配的标记。
5. Flutter计算从起点到终点设置Hero边界的动画的补间(插值大小和位置),并在叠加层中执行动画。

下一节将更详细地介绍Flutter的处理过程。


## 表象背后

以下描述了Flutter如何执行从一个路由到另一个路由的过渡。

![](https://flutter.dev/assets/ui/animations/hero-transition-0-e129c393b824c1026897ff0051019e344062817525ea83babf3d8afb74e0234c.png)

在转换之前，源`hero`在源路由的widget树中等待。目标路由尚不存在，并且叠加层为空。

![](https://flutter.dev/assets/ui/animations/hero-transition-1-af93acb4da6e70ef6d902af4b64b2a9d1ab92c97ebd6a2580d259bdf058e36ad.png)

将路由`push`到导航器会触发动画。在t = 0.0时，Flutter执行以下操作：

在画面外，使用`Material`运动规范中描述的曲面运动计算目标`hero`的路径。Flutter现在知道`hero`最终的位置。

将目标`hero`放置在叠加层中，与源`hero`的位置和大小相同。向叠加层添加`hero`会更改其Z轴上顺序，以使其显示在所有路由的顶部。

将源`hero`移动到屏幕外。

![](https://flutter.dev/assets/ui/animations/hero-transition-2-fb104777adbaac9003639cab2b40aa4fae20033d522be4064b1790653e2fef29.png)


当`hero`飞行时，它的矩形边界使用在Hero的createRectTween属性中指定的[Tween \<Rect>](https://api.flutter.dev/flutter/animation/Tween-class.html)进行动画处理。默认情况下，Flutter使用[MaterialRectArcTween](https://api.flutter.dev/flutter/material/MaterialRectArcTween-class.html)的实例，该实例沿着弯曲路径设置矩形的对角线。(有关使用不同Tween动画的示例，请参阅径向`hero`动画。）

![](https://flutter.dev/assets/ui/animations/hero-transition-3-66ff086cd05a7ff64290c41634d6bbd63c025126427fc0a3459af8fd71b7cded.png)

飞行完成时：

Flutter将hero widget从叠加层移动到目标路由。叠加层现在为空。

目标`hero`出现在目的地路由的最终位置。

源`hero`将恢复到其路由。

pop路由执行相同的过程，将`hero`动画回原点和源路由中的位置。

### 必要的类

本指南中的示例使用以下类来实现`hero`动画：

`hero`

从源到目标路由的widget。为源路由定义一个`Hero`，为目标路由定义另一个`Hero`，并为每个分配相同的标记。flutter使用匹配的标签做出`hero`对的动画。

`Inkwell`

指定点击`hero`时会发生什么。InkWell的onTap()方法构建新路由并将其`push`到Navigator的堆栈。

`Navigator`

Navigator管理一堆路由。从导航器的堆栈中`push`路由或`pop`路由会触发动画。

`Route`

指定屏幕或页面。除最基本的应用程序之外，大多数应用程序都有个路由。


## 标准Hero动画

重点是什么？

* 使用`MaterialPageRoute`，`CupertinoPageRoute`指定路由，或使用`PageRouteBuilder`构建自定义路由。本节中的示例使用`MaterialPageRoute`。
* 通过将目标图像包装在SizedBox中，在过渡结束时更改图像的大小。
* 通过将目标图像放置在widget中来更改图像的位置。这些示例使用Container。


标准`hero`动画代码

以下每个示例都演示了将图像从一个路由飞到另一个路由。本指南介绍了第一个示例。

[hero_animation](https://github.com/flutter/website/tree/master/examples/_animation/hero_animation/)

将`hero`代码封装在自定义PhotoHero widget 中。沿着曲线路径动画`hero`的动作，如Material运动规范中所述。

[basic_hero_animation](https://github.com/flutter/website/tree/master/examples/_animation/basic_hero_animation/)

直接使用`hero`widget。本指南中未介绍此基本示例，供你参考。


### 这是怎么回事？
使用Flutter的`hero` widget可以轻松实现将图像从一个路由传输到另一个路由。使用`MaterialPageRoute`指定新路由时，图像沿着弯曲路径飞行，如“Material设计”运动规范所述。

[创建一个新的Flutter示例](https://flutter.dev/docs/get-started/test-drive)并使用[Gi​​tHub目录中](https://github.com/flutter/website/tree/master/examples/_animation/hero_animation/)的文件进行更新。

运行示例：

* 点击主路由的照片，将图像拖动到新的路由，在不同的位置，以不同比例显示相同的照片。
* 通过点击图像或使用设备的返回上一个路由手势返回上一个路由。
* 你可以使用`timeDilation`属性进一步减慢过渡。


### PhotoHero类

自定义PhotoHero类在点击时维护`hero`及其大小，图像和行为。PhotoHero构建了以下widget树：

![](https://flutter.dev/assets/ui/animations/photohero-class-ebc745117913037726a9660636a95d6b54fb59d077bceecef1546975722c4c1a.png)

这里是代码

```
class PhotoHero extends StatelessWidget {
  const PhotoHero({ Key key, this.photo, this.onTap, this.width }) : super(key: key);

  final String photo;
  final VoidCallback onTap;
  final double width;

  Widget build(BuildContext context) {
    return SizedBox(
      width: width,
      child: Hero(
        tag: photo,
        child: Material(
          color: Colors.transparent,
          child: InkWell(
            onTap: onTap,
            child: Image.asset(
              photo,
              fit: BoxFit.contain,
            ),
          ),
        ),
      ),
    );
  }
}
```

关键信息：

* 当HeroAnimation作为app的home属性提供时，MaterialApp会隐式push起始路由。
* InkWell包装图像，使得向源和目标`hero`添加点击手势变得微不足道。
* 使用透明颜色定义“Material”widget可使图像在飞往目标时“弹出”背景。
* SizedBox指定动画开始和结束时`hero`的大小。
* 将Image的fit属性设置为BoxFit.contain，可确保在过渡期间图像尽可能大，而不会更改其宽高比。


### HeroAnimation类
HeroAnimation类创建源和目标 `PhotoHeroes`，并设置过渡。

这是代码：

```dart
class HeroAnimation extends StatelessWidget {
  Widget build(BuildContext context) {
    timeDilation = 5.0; // 1.0 means normal animation speed.

    return Scaffold(
      appBar: AppBar(
        title: const Text('Basic Hero Animation'),
      ),
      body: Center(
        child: PhotoHero(
          photo: 'images/flippers-alpha.png',
          width: 300.0,
          onTap: () {
            Navigator.of(context).push(MaterialPageRoute<void>(
              builder: (BuildContext context) {
                return Scaffold(
                  appBar: AppBar(
                    title: const Text('Flippers Page'),
                  ),
                  body: Container(
                    // The blue background emphasizes that it's a new route.
                    color: Colors.lightBlueAccent,
                    padding: const EdgeInsets.all(16.0),
                    alignment: Alignment.topLeft,
                    child: PhotoHero(
                      photo: 'images/flippers-alpha.png',
                      width: 100.0,
                      onTap: () {
                        Navigator.of(context).pop();
                      },
                    ),
                  ),
                );
              }
            ));
          },
        ),
      ),
    );
  }
}
```


关键信息：

* 当用户点击包含源`hero`的InkWell时，代码使用`MaterialPageRoute`创建目标路由。将目标路由`push`到导航器的堆栈会触发动画。
* Container将PhotoHero定位在AppBar下方的目的路由的左上角。
* 目标PhotoHero的onTap()方法 `pop`导航器的堆栈，触发将Hero飞回原始路由的动画。
* 使用`timeDilation`属性可以在调试时减慢转换速度。

## 径向hero动画

重点是什么？

* 径向过渡将圆形动画变为方形。
* 径向`hero`动画在将`hero`从源路由飞到目标路由时执行径向过渡。
* `MaterialRectCenterArcTween` 定义 补间动画。
* 使用`PageRouteBuilder`构建目标路由。

将路由从一个路由飞向另一个路由，因为它从圆形过渡为矩形，这是一种光滑的效果，你可以使用`hero`widget来实现。为此，代码动画两个剪辑形状的交集：圆形和方形。在整个动画中，圆形剪辑（和图像）从minRadius缩放到maxRadius，而方形剪辑保持不变的大小。同时，图像从其在源路由中的位置飞到其在目的路由中的位置。有关此过渡的可视示例，请参阅材质运动规范中的[径向过渡](https://material.io/guidelines/motion/transforming-material.html#transforming-material-radial-transformation)。

此动画可能看起来很复杂(并且确实如此)，但你可以根据需要自定义提供的示例。繁重的工作已经为你完成了。


径向`hero`动画代码

以下每个示例都演示了径向`hero`动画。本指南介绍了第一个示例。

[radial_hero_animation](https://github.com/flutter/website/tree/master/examples/_animation/radial_hero_animation)

材质运动规范中描述的径向`hero`动画。

[basic_radial_hero_animation](https://github.com/flutter/website/tree/master/examples/_animation/basic_radial_hero_animation)

径向`hero`动画的最简单示例。目的路由没有`Scaffold`, `Card`, `Column`或`Text`。本指南中未介绍此基本示例，供你参考。

[radial_hero_animation_animate_rectclip](https://github.com/flutter/website/tree/master/examples/_animation/radial_hero_animation_animate_rectclip)

通过动画化矩形剪辑的大小来扩展radial_hero_animaton。本指南中未介绍此高级示例，供你参考。

专业提示：径向`hero`动画涉及将圆形与正方形相交。即使使用timeDilation减慢动画速度，也很难看到这一点，因此你可以考虑在开发过程中启用Flutter的可视化调试模式。


### 这是怎么回事？

下图显示了动画开头(t = 0.0)和结束(t = 1.0)的剪裁图像。

![](https://flutter.dev/assets/ui/animations/radial-hero-animation-cf2651ecb9c60cd5001451fb534e90621774cc186adebd519b056eaa92dec33f.png)

蓝色渐变(表示图像)表示剪辑形状相交的位置。在转换开始时，交集的结果是一个圆形剪辑（[ClipOval](https://api.flutter.dev/flutter/widgets/ClipOval-class.html)）。在转换期间，ClipOval从minRadius缩放到maxRadius，而[ClipRect](https://api.flutter.dev/flutter/widgets/ClipRect-class.html)保持恒定大小。在过渡结束时，圆形和矩形剪辑的交叉产生一个与`hero`widget大小相同的矩形。换句话说，在转换结束时，图像不再被剪裁。

[创建一个新的Flutter示例](https://flutter.dev/docs/get-started/test-drive)并使用[Gi​​tHub目录](https://github.com/flutter/website/tree/master/examples/_animation/radial_hero_animation)中的文件进行更新。

运行示例：

* 点击三个圆形缩略图中的一个，将图像设置为一个较大的正方形，该正方形位于新路由的中间，遮挡了原始路由。
* 通过点击图像或使用设备的返回上一个路由手势返回上一个路由。
* 你可以使用timeDilation属性进一步减慢过渡。


### Photo类

Photo类构建保存图像的widget树：
```
class Photo extends StatelessWidget {
  Photo({ Key key, this.photo, this.color, this.onTap }) : super(key: key);

  final String photo;
  final Color color;
  final VoidCallback onTap;

  Widget build(BuildContext context) {
    return Material(
      // Slightly opaque color appears where the image has transparency.
      color: Theme.of(context).primaryColor.withOpacity(0.25),
      child: InkWell(
        onTap: onTap,
        child: Image.asset(
            photo,
            fit: BoxFit.contain,
          )
      ),
    );
  }
}
```

关键信息：

* Inkwell捕获轻击手势。调用函数将onTap()函数传递给Photo的构造函数。
* 在飞行过程中，InkWell在它的第一个`Material`祖先上散开。
* `Material`”widget具有略微不透明的颜色，因此图像的透明部分将使用颜色进行渲染。这确保了即使对于具有透明度的图像也容易看到圆到方的过渡。
* Photo类在其widget树中不包含Hero。为了使动画起作用，`hero`包装了RadialExpansion widget。


### RadialExpansion类
RadialExpansion widget 是演示的核心，它构建了在过渡期间剪切图像的widget树。剪裁的形状来自圆形剪辑(在飞行期间生长)与矩形剪辑(整个过程中保持恒定大小)的交叉。

为此，它构建了以下widget树：

![](https://flutter.dev/assets/ui/animations/radial-expansion-class-057f907d3b6ff22ac2c857a2e739436dd36eedd3ef0ed9dd73839d93d026f0d7.png)

这里是代码:

```dart
class RadialExpansion extends StatelessWidget {
  RadialExpansion({
    Key key,
    this.maxRadius,
    this.child,
  }) : clipRectSize = 2.0 * (maxRadius / math.sqrt2),
       super(key: key);

  final double maxRadius;
  final clipRectSize;
  final Widget child;

  @override
  Widget build(BuildContext context) 
    return ClipOval(
      child: Center(
        child: SizedBox(
          width: clipRectSize,
          height: clipRectSize,
          child: ClipRect(
            child: child,  // Photo
          ),
        ),
      ),
    );
  }
}
```

关键信息：

* `hero`包装了RadialExpansion widget。
* 当`hero`飞行时，它的大小会发生变化，并且由于它限制了孩子的大小，因此`RadialExpansion` widget会更改大小以匹配。
* `RadialExpansion`动画由两个重叠的剪辑创建。
* 该示例使用`MaterialRectCenterArcTween`定义补间插值。`hero`动画的默认飞行路径使用`hero`的角插入补间。此方法会影响径向变换期间`hero`的纵横比，因此新的飞行路由使用`MaterialRectCenterArcTween`使用每个`hero`的中心点插补补间。

这是代码：

```dart
static RectTween _createRectTween(Rect begin, Rect end) {
  return MaterialRectCenterArcTween(begin: begin, end: end);
}
```

`hero`的飞行路径仍遵循弧线，但图像的纵横比保持不变。



