---
layout: post
title:  "(转)导航到一个新页面和返回"
date:   2019-04-24 14:16:17 +0800
categories: 
   - Flutter 开发
tags:
    - navigation
---

[原文链接](https://flutter.cn/docs/cookbook/navigation/navigation-basics)

我们通常会用“屏”来表示应用的不同页面（界面），比如，某个应用有一“屏”展示商品列表，当用户点击某个商品的图片，会跳到新的一“屏”展示商品的详细信息。

术语: 在 Flutter 中，屏 (screen) 和 页面 (page) 都叫做 路由 (route)， 在下文中统称为“路由 (route)”。

在 Android 开发中，Activity 相当于“路由” , 在 iOS 开发中，ViewController 相当于“路由”。 在 Flutter 中，“路由”也是一个 Widget。

怎么样从一个“路由”跳转到新的“路由“呢？你需要使用 [Navigator](https://api.flutter.dev/flutter/widgets/Navigator-class.html) 类。

<!--more-->


## 步骤

下面来展示如何在两个路由间跳转，总共分三步：

1. 创建两个路由

2. 用 Navigator.push() 跳转到第二个路由

3. 用 Navigator.pop() 回退到第一个路由

## 1. 创建两个路由

首先，我们来创建两个路由。这是个最简单的例子，每个路由只包含一个按钮。 点击第一个路由上的按钮会跳转到第二个路由，点击第二个路由上的按钮，会回退到第一个路由。

首先来编写界面布局代码：

```dart
class FirstRoute extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('First Route'),
      ),
      body: Center(
        child: RaisedButton(
          child: Text('Open route'),
          onPressed: () {
            // Navigate to second route when tapped.
          },
        ),
      ),
    );
  }
}

class SecondRoute extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("Second Route"),
      ),
      body: Center(
        child: RaisedButton(
          onPressed: () {
            // Navigate back to first route when tapped.
          },
          child: Text('Go back!'),
        ),
      ),
    );
  }
}
```

## 2.用 Navigator.push() 跳转到第二个路由

使用 Navigator.push()方法跳转到新的路由。 `push()` 方法会添加一个 `Route` 对象到导航器的堆栈上。 那么这个 `Route` 对象是从哪里来的呢？ 你可以自己实现一个，或者直接使用 [MaterialPageRoute](https://api.flutter.dev/flutter/material/MaterialPageRoute-class.html)类。 使用 `MaterialPageRoute` 是非常方便的，框架已经为我们实现了和平台原生类似的切换动画。

在 `FirstRoute` widget 的 `build()` 方法中，我们来修改 onPressed() 回调函数：

```dart
// 位于 FirstRoute widget (Within the `FirstRoute` widget)
onPressed: () {
  Navigator.push(
    context,
    MaterialPageRoute(builder: (context) => SecondRoute()),
  );
}
```

## 3.用 Navigator.pop() 回退到第一个路由
怎么关闭第二个路由回退到第一个呢? 使用 [Navigator.pop()](https://api.flutter.dev/flutter/widgets/Navigator/pop.html) 方法，`pop()` 方法会从导航器堆栈上移除 `Route` 对象。

我们来修改 `SecondRoute` widget 的 `onPressed()` 回调函数，实现返回第一个路由的功能：

```dart
// 位于 SecondRoute widget (Within the SecondRoute widget)
onPressed: () {
  Navigator.pop(context);
}
```

## 完整的例子

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(MaterialApp(
    title: 'Navigation Basics',
    home: FirstRoute(),
  ));
}

class FirstRoute extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('First Route'),
      ),
      body: Center(
        child: RaisedButton(
          child: Text('Open route'),
          onPressed: () {
            Navigator.push(
              context,
              MaterialPageRoute(builder: (context) => SecondRoute()),
            );
          },
        ),
      ),
    );
  }
}

class SecondRoute extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("Second Route"),
      ),
      body: Center(
        child: RaisedButton(
          onPressed: () {
            Navigator.pop(context);
          },
          child: Text('Go back!'),
        ),
      ),
    );
  }
}
```
![](https://flutter.cn/images/cookbook/navigation-basics.gif)
