---
layout: post
title:  "(译)从路由返回数据"
date:   2019-04-24 16:40:17 +0800
categories: 
   - Flutter 开发
tags:
    - navigation
---

[原文链接](https://flutter.dev/docs/cookbook/navigation/returning-data)

在某些情况下，我们可能希望从新路由返回数据。例如，假设我们`push`一个向用户提供两个选项的新路由。当用户点击选项时，我们想要通知我们的第一个路由用户的选择，以便它可以对该信息采取行动！

我们怎样才能做到这一点？使用[Navigator.pop](https://api.flutter.dev/flutter/widgets/Navigator/pop.html)！

<!--more-->


## 路线

1. 定义主路由
2. 添加一个启动选择路由的按钮
3. 使用两个按钮显示选择路由
4. 点击按钮时，关闭选择路由
5. 在主屏幕上用snackbar显示选择的那项

## 1.定义主屏幕

主路由将显示一个按钮。点击时，它将启动选择路由！

```dart
class HomeScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Returning Data Demo'),
      ),
      // We'll create the SelectionButton Widget in the next step
      body: Center(child: SelectionButton()),
    );
  }
}
```

## 2.添加一个启动选择路由的按钮

现在，我们将创建SelectionButton。我们的选择按钮将：

1. 点击后启动`SelectionScreen`
2. 等待`SelectionScreen`返回结果

```dart
class SelectionButton extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return RaisedButton(
      onPressed: () {
        _navigateAndDisplaySelection(context);
      },
      child: Text('Pick an option, any option!'),
    );
  }

  // A method that launches the SelectionScreen and awaits the result from
  // Navigator.pop
  _navigateAndDisplaySelection(BuildContext context) async {
    // Navigator.push returns a Future that will complete after we call
    // Navigator.pop on the Selection Screen!
    final result = await Navigator.push(
      context,
      // We'll create the SelectionScreen in the next step!
      MaterialPageRoute(builder: (context) => SelectionScreen()),
    );
  }
}
```

## 3.使用两个按钮显示选择路由

现在，我们需要建立一个选择路由！它将包含两个按钮。当用户点击按钮时，它应该关闭选择路由，并让主路由知道点击了哪个按钮！

现在，我们将定义UI，并确定如何在下一步中返回数据。

```dart
class SelectionScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Pick an option'),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Padding(
              padding: const EdgeInsets.all(8.0),
              child: RaisedButton(
                onPressed: () {
                  // Pop here with "Yep"...
                },
                child: Text('Yep!'),
              ),
            ),
            Padding(
              padding: const EdgeInsets.all(8.0),
              child: RaisedButton(
                onPressed: () {
                  // Pop here with "Nope"
                },
                child: Text('Nope.'),
              ),
            )
          ],
        ),
      ),
    );
  }
}
```

## 4.点击按钮时，关闭选择路由
现在，我们要修改两个按钮的onPressed回调！为了将数据返回到第一个路由，我们需要使用[Navigator.pop](https://api.flutter.dev/flutter/widgets/Navigator/pop.html)方法。

`Navigator.pop`接收一个名为result的可选的第二个参数。如果我们提供结果，它将在SelectionButton中返回到`Future`中！

## 是的按钮
```dart
RaisedButton(
  onPressed: () {
    // Our Yep button will return "Yep!" as the result
    Navigator.pop(context, 'Yep!');
  },
  child: Text('Yep!'),
);
```

## 否的按钮

```dart
RaisedButton(
  onPressed: () {
    // Our Nope button will return "Nope!" as the result
    Navigator.pop(context, 'Nope!');
  },
  child: Text('Nope!'),
);
```


## 5.在主路由用snarkback显示选择的那项

既然我们正在启动选择路由并等待结果，我们将要对所返回的信息做些什么！

在这种情况下，我们将显示一个显示结果的Snackbar。为此，我们将在SelectionButton中修改_navigateAndDisplaySelection方法。

```dart
_navigateAndDisplaySelection(BuildContext context) async {
  final result = await Navigator.push(
    context,
    MaterialPageRoute(builder: (context) => SelectionScreen()),
  );

  // After the Selection Screen returns a result, hide any previous snackbars
  // and show the new result!
  Scaffold.of(context)
    ..removeCurrentSnackBar()
    ..showSnackBar(SnackBar(content: Text("$result")));
}
```

## 完整例子

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(MaterialApp(
    title: 'Returning Data',
    home: HomeScreen(),
  ));
}

class HomeScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Returning Data Demo'),
      ),
      body: Center(child: SelectionButton()),
    );
  }
}

class SelectionButton extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return RaisedButton(
      onPressed: () {
        _navigateAndDisplaySelection(context);
      },
      child: Text('Pick an option, any option!'),
    );
  }

  // A method that launches the SelectionScreen and awaits the result from
  // Navigator.pop!
  _navigateAndDisplaySelection(BuildContext context) async {
    // Navigator.push returns a Future that will complete after we call
    // Navigator.pop on the Selection Screen!
    final result = await Navigator.push(
      context,
      MaterialPageRoute(builder: (context) => SelectionScreen()),
    );

    // After the Selection Screen returns a result, hide any previous snackbars
    // and show the new result!
    Scaffold.of(context)
      ..removeCurrentSnackBar()
      ..showSnackBar(SnackBar(content: Text("$result")));
  }
}

class SelectionScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Pick an option'),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Padding(
              padding: const EdgeInsets.all(8.0),
              child: RaisedButton(
                onPressed: () {
                  // Close the screen and return "Yep!" as the result
                  Navigator.pop(context, 'Yep!');
                },
                child: Text('Yep!'),
              ),
            ),
            Padding(
              padding: const EdgeInsets.all(8.0),
              child: RaisedButton(
                onPressed: () {
                  // Close the screen and return "Nope!" as the result
                  Navigator.pop(context, 'Nope.');
                },
                child: Text('Nope.'),
              ),
            )
          ],
        ),
      ),
    );
  }
}
```

![img](https://flutter.dev/images/cookbook/returning-data.gif)
