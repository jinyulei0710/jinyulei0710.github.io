---
layout: post
title:  "(译)传递参数到命名过的路由"
date:   2019-04-24 15:16:17 +0800
categories: 
   - Flutter 开发
tags:
    - navigation
---

[原文链接](https://flutter.dev/docs/cookbook/navigation/navigate-with-arguments)

导航器提供了使用公共标识符从应用程序的任何部分导航到命名过的路由的功能。在某些情况下，你可能还需要将参数传递给命名过的路由。例如，你可能希望导航到`/user`路由并将有关用户的信息传递给该路由。

在Flutter中，你可以通过为Navigator.pushNamed方法提供其他参数来完成此任务。你可以使用ModalRoute.of方法或在提供给MaterialApp或CupertinoApp构造函数的onGenerateRoute函数内提取参数。

此配方演示了如何将参数传递给命名过的路由并使用`ModelRoute.of`和`onGenerateRoute`读取参数。

<!--more-->


## 步骤

1. 定义你需要传递的参数
2. 创建一个提取参数的widget
3. 在路由表中注册widget
4. 导航到widget

## 1. 定义你需要传递的参数

首先，定义传递给新路由所需的参数。在此示例中，传递两个数据：路由标题以及一个消息。

要传递两个数据，请创建一个存储此信息的类。

```dart
// You can pass any object to the arguments parameter. In this example, create a
// class that contains both a customizable title and message.
class ScreenArguments {
  final String title;
  final String message;

  ScreenArguments(this.title, this.message);
}
```

## 2. 创建一个提取参数的widget

接下来，创建一个widget，从ScreenArguments中提取并显示标题和消息。要访问ScreenArguments，请使用ModalRoute.of方法。此方法返回带有参数的当前路由。

```dart
// A Widget that extracts the necessary arguments from the ModalRoute.
class ExtractArgumentsScreen extends StatelessWidget {
  static const routeName = '/extractArguments';

  @override
  Widget build(BuildContext context) {
    // Extract the arguments from the current ModalRoute settings and cast
    // them as ScreenArguments.
    final ScreenArguments args = ModalRoute.of(context).settings.arguments;

    return Scaffold(
      appBar: AppBar(
        title: Text(args.title),
      ),
      body: Center(
        child: Text(args.message),
      ),
    );
  }
}
```

## 3.在路由表中注册widget

接下来，在提供给`MaterialApp` Widget的路径中添加一个条目。路由根据路由名称定义应创建哪个widget。

```dart
MaterialApp(
  routes: {
    ExtractArgumentsScreen.routeName: (context) => ExtractArgumentsScreen(),
  },     
);
```

## 4.导航到widget

最后，当用户使用Navigator.pushNamed点击按钮时，导航到ExtractArgumentsScreen。通过arguments属性为路由提供参数。 ExtractArgumentsScreen从这些参数中提取标题和消息。

```dart
// A button that navigates to a named route that. The named route
// extracts the arguments by itself.
RaisedButton(
  child: Text("Navigate to screen that extracts arguments"),
  onPressed: () {
    // When the user taps the button, navigate to the specific rout
    // and provide the arguments as part of the RouteSettings.
    Navigator.pushNamed(
      context,
      ExtractArgumentsScreen.routeName,
      arguments: ScreenArguments(
        'Extract Arguments Screen',
        'This message is extracted in the build method.',
      ),
    );
  },
);
```

## 或者，使用onGenerateRoute提取参数
你也可以在onGenerateRoute函数中提取参数并将它们传递给widget，而不是直接在widget中提取参数。

onGenerateRoute函数根据给定的`RouteSettings`创建正确的路由。

```dart
MaterialApp(
  // Provide a function to handle named routes. Use this function to
  // identify the named route being pushed and create the correct
  // Screen.
  onGenerateRoute: (settings) {
    // If you push the PassArguments route
    if (settings.name == PassArgumentsScreen.routeName) {
      // Cast the arguments to the correct type: ScreenArguments.
      final ScreenArguments args = settings.arguments;

      // Then, extract the required data from the arguments and
      // pass the data to the correct screen.
      return MaterialPageRoute(
        builder: (context) {
          return PassArgumentsScreen(
            title: args.title,
            message: args.message,
          );
        },
      );
    }
  },
);
```

## 完整例子

```dart
import 'package:flutter/material.dart';

void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      // Provide a function to handle named routes. Use this function to
      // identify the named route being pushed and create the correct
      // Screen.
      onGenerateRoute: (settings) {
        // If you push the PassArguments route
        if (settings.name == PassArgumentsScreen.routeName) {
          // Cast the arguments to the correct type: ScreenArguments.
          final ScreenArguments args = settings.arguments;

          // Then, extract the required data from the arguments and
          // pass the data to the correct screen.
          return MaterialPageRoute(
            builder: (context) {
              return PassArgumentsScreen(
                title: args.title,
                message: args.message,
              );
            },
          );
        }
      },
      title: 'Navigation with Arguments',
      home: HomeScreen(),
    );
  }
}

class HomeScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Home Screen'),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            // A button that navigates to a named route that. The named route
            // extracts the arguments by itself.
            RaisedButton(
              child: Text("Navigate to screen that extracts arguments"),
              onPressed: () {
                // When the user taps the button, navigate to the specific route
                // and provide the arguments as part of the RouteSettings.
                Navigator.push(
                  context,
                  MaterialPageRoute(
                    builder: (context) => ExtractArgumentsScreen(),
                    // Pass the arguments as part of the RouteSettings. The
                    // ExtractArgumentScreen reads the arguments from these
                    // settings.
                    settings: RouteSettings(
                      arguments: ScreenArguments(
                        'Extract Arguments Screen',
                        'This message is extracted in the build method.',
                      ),
                    ),
                  ),
                );
              },
            ),
            // A button that navigates to a named route. For this route, extract
            // the arguments in the onGenerateRoute function and pass them
            // to the screen.
            RaisedButton(
              child: Text("Navigate to a named that accepts arguments"),
              onPressed: () {
                // When the user taps the button, navigate to a named route
                // and provide the arguments as an optional parameter.
                Navigator.pushNamed(
                  context,
                  PassArgumentsScreen.routeName,
                  arguments: ScreenArguments(
                    'Accept Arguments Screen',
                    'This message is extracted in the onGenerateRoute function.',
                  ),
                );
              },
            ),
          ],
        ),
      ),
    );
  }
}

// A Widget that extracts the necessary arguments from the ModalRoute.
class ExtractArgumentsScreen extends StatelessWidget {
  static const routeName = '/extractArguments';

  @override
  Widget build(BuildContext context) {
    // Extract the arguments from the current ModalRoute settings and cast
    // them as ScreenArguments.
    final ScreenArguments args = ModalRoute.of(context).settings.arguments;

    return Scaffold(
      appBar: AppBar(
        title: Text(args.title),
      ),
      body: Center(
        child: Text(args.message),
      ),
    );
  }
}

// A Widget that accepts the necessary arguments via the constructor.
class PassArgumentsScreen extends StatelessWidget {
  static const routeName = '/passArguments';

  final String title;
  final String message;

  // This Widget accepts the arguments as constructor parameters. It does not
  // extract the arguments from the ModalRoute.
  //
  // The arguments are extracted by the onGenerateRoute function provided to the
  // MaterialApp widget.
  const PassArgumentsScreen({
    Key key,
    @required this.title,
    @required this.message,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(title),
      ),
      body: Center(
        child: Text(message),
      ),
    );
  }
}

// You can pass any object to the arguments parameter. In this example, create a
// class that contains both a customizable title and message.
class ScreenArguments {
  final String title;
  final String message;

  ScreenArguments(this.title, this.message);
}
```
![](https://flutter.dev/images/cookbook/navigate-with-arguments.gif)