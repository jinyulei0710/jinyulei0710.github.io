---
layout: post
title:  "(译)在路由上设置widget动画"
date:   2019-04-24 17:40:17 +0800
categories: 
   - Flutter 开发
tags:
    - navigation
    - Animations
---

[原文链接](https://flutter.dev/docs/cookbook/navigation/hero-animations)

在路由之间导航时，引导用户浏览我们的应用通常很有帮助。引导用户浏览应用程序的常用技巧是将Widget以动画的形式从一个路由到下一个路由。这创建了连接两个路由的视觉锚点。

我们如何使用Flutter将Widget以动画的形式从一个路由到下一个路由？使用[Hero](https://api.flutter.dev/flutter/widgets/Hero-class.html) widget！

<!--more-->

## 步骤

1. 创建两个显示相同图像的路由
2. 将`Hero` Widget添加到第一个路由
3. 将`Hero` Widget添加到第二个路由

## 1.创建两个显示相同图像的路由

在此示例中，我们将在两个路由上显示相同的图像。 当用户点击图像时，我们希望将图像以动画的形式从一个路由到下一个路由。 现在，我们将创建视觉结构，并在接下来的步骤中处理动画！

注意：此示例基于[导航到新路由并返回](https://flutter.dev/docs/cookbook/navigation/navigation-basics)和[处理点击事件](https://flutter.dev/docs/cookbook/gestures/handling-taps)配方。

```dart
class MainScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Main Screen'),
      ),
      body: GestureDetector(
        onTap: () {
          Navigator.push(context, MaterialPageRoute(builder: (_) {
            return DetailScreen();
          }));
        },
        child: Image.network(
          'https://picsum.photos/250?image=9',
        ),
      ),
    );
  }
}

class DetailScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: GestureDetector(
        onTap: () {
          Navigator.pop(context);
        },
        child: Center(
          child: Image.network(
            'https://picsum.photos/250?image=9',
          ),
        ),
      ),
    );
  }
}
```

## 2.将Hero Widget添加到第一个路由

为了将两个屏幕与动画连接在一起，我们需要将Image Widget包装在Hero Widget的两个路由上。 Hero Widget需要两个参数：

1. tag：标识Hero的对象。两个路由上必须相同。
2. child：我们想要在路由之间实现动画效果的widget。

```dart
Hero(
  tag: 'imageHero',
  child: Image.network(
    'https://picsum.photos/250?image=9',
  ),
);
```

3.将Hero Widget添加到第二个路由

要完成与第一个路由的连接，我们需要使用Hero Widget将Image包装在第二个路由上！它必须使用与第一个路由相同的标签。

将Hero Widget应用到第二个路由后，路由之间的动画将起作用！

```dart
Hero(
  tag: 'imageHero',
  child: Image.network(
    'https://picsum.photos/250?image=9',
  ),
);
```

注意：此代码与我们在第一个路由上的代码相同！通常，您可以创建可重用的Widget而不是重复代码，但是对于此示例，我们将复制代码以用于演示目的。

完整例子

```dart
import 'package:flutter/material.dart';

void main() => runApp(HeroApp());

class HeroApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Transition Demo',
      home: MainScreen(),
    );
  }
}

class MainScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Main Screen'),
      ),
      body: GestureDetector(
        child: Hero(
          tag: 'imageHero',
          child: Image.network(
            'https://picsum.photos/250?image=9',
          ),
        ),
        onTap: () {
          Navigator.push(context, MaterialPageRoute(builder: (_) {
            return DetailScreen();
          }));
        },
      ),
    );
  }
}

class DetailScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: GestureDetector(
        child: Center(
          child: Hero(
            tag: 'imageHero',
            child: Image.network(
              'https://picsum.photos/250?image=9',
            ),
          ),
        ),
        onTap: () {
          Navigator.pop(context);
        },
      ),
    );
  }
}
```
![img](https://flutter.dev/images/cookbook/hero.gif)