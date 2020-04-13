---
layout: post
title:  "(译)使用命名过的路由进行导航"
date:   2019-04-24 15:16:17 +0800
categories: 
   - Flutter 开发
tags:
   - navigation
---

[原文链接](https://flutter.dev/docs/cookbook/navigation/named-routes)

在导航到导航到一个新页面和返回的方法中，我们学习了如何通过创建新路由并将其`push`导航器来导航到新路由。

但是，如果我们需要在应用程序的许多部分导航到同一路由，这可能会导致代码重复。 在这些情况下，定义“命名过的路由”，并使用“命名过的路由“进行导航，是非常方便的。

要使用“命名过的路由"，我们可以使用Navigator.pushNamed函数。 此示例将复制原始功能，演示如何使用“命名过的路由"。

<!--more-->


## 步骤
1. 创建两个路由
2. 定义路由表(routes)
3. 使用Navigator.pushNamed导航到第二个路由
4. 使用Navigator.pop返回第一个路由

## 1.创建两个路由
首先，我们需要使用两个路由。 第一个路由将包含一个导航到第二个路由的按钮。 第二个路由将包含一个导航回第一个路由的按钮。

```dart
class FirstScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('First Screen'),
      ),
      body: Center(
        child: RaisedButton(
          child: Text('Launch screen'),
          onPressed: () {
            // Navigate to second screen when tapped!
          },
        ),
      ),
    );
  }
}

class SecondScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("Second Screen"),
      ),
      body: Center(
        child: RaisedButton(
          onPressed: () {
            // Navigate back to first screen when tapped!
          },
          child: Text('Go back!'),
        ),
      ),
    );
  }
}
```

## 2.定义路由

接下来，我们需要通过为`MaterialAp`p构造函数提供其他属性来定义我们的路由：`initialRoute`和`routes`它们自己。

`initialRoute`属性定义了应用程序应该从哪个路由开始。`routes`属性定义可用的命名过的路由以及导航到这些路由时应构建的widget。

```dart
MaterialApp(
  // Start the app with the "/" named route. In our case, the app will start
  // on the FirstScreen Widget
  initialRoute: '/',
  routes: {
    // When we navigate to the "/" route, build the FirstScreen Widget
    '/': (context) => FirstScreen(),
    // When we navigate to the "/second" route, build the SecondScreen Widget
    '/second': (context) => SecondScreen(),
  },
);
```

注意：使用`initialRoute`时，请确保未定义`home`属性。

## 3.导航到第二个路由

通过我们的widget和路由，我们可以开始导航！在这种情况下，我们将使用Navigator.pushNamed函数。这告诉Flutter构建路由表中定义的Widget并启动路由。

在我们的FirstScreen widget 的build方法中，我们将修改`onPressed`回调：

```dart
// Within the `FirstScreen` Widget
onPressed: () {
  // Navigate to the second screen using a named route
  Navigator.pushNamed(context, '/second');
}
```

## 4.返回第一个路由

为了导航回第一页，我们可以使用Navigator.pop函数。

```dart
// Within the SecondScreen Widget
onPressed: () {
  // Navigate back to the first screen by popping the current route
  // off the stack
  Navigator.pop(context);
}
```

## 完整例子

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(MaterialApp(
    title: 'Named Routes Demo',
    // Start the app with the "/" named route. In our case, the app will start
    // on the FirstScreen Widget
    initialRoute: '/',
    routes: {
      // When we navigate to the "/" route, build the FirstScreen Widget
      '/': (context) => FirstScreen(),
      // When we navigate to the "/second" route, build the SecondScreen Widget
      '/second': (context) => SecondScreen(),
    },
  ));
}

class FirstScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('First Screen'),
      ),
      body: Center(
        child: RaisedButton(
          child: Text('Launch screen'),
          onPressed: () {
            // Navigate to the second screen using a named route
            Navigator.pushNamed(context, '/second');
          },
        ),
      ),
    );
  }
}

class SecondScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("Second Screen"),
      ),
      body: Center(
        child: RaisedButton(
          onPressed: () {
            // Navigate back to the first screen by popping the current route
            // off the stack
            Navigator.pop(context);
          },
          child: Text('Go back!'),
        ),
      ),
    );
  }
}
```
![img](https://flutter.dev/images/cookbook/navigation-basics.gif)