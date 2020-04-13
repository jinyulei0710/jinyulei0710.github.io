---
layout: post
title:  "(译) Google Flutter 中的简单状态管理"
date:   2019-04-18 09:55:17 +0800
categories: 
   - Flutter 开发
tags:
    - 状态管理
    - setState
---
[原文链接](https://medium.com/@agungsurya/basic-state-management-in-google-flutter-6ee73608f96d)

## 我是如何遇见 Google Flutter的

这对我来这是像往常一样的码代码的一天。我的一个朋友在我们的开发者群组中发了这么一个问题，是否有人尝试过Google Flutter。它想要知道React Native 和 Google Flutter之间的比较。这个问题让我种草了Google Flutter。我之前从没有听过Google Flutter。它是否值得与React Native进行比较，就像AngularJS相较于ReactJS?

我必须承认。我是一个React的迷弟。我已经使用ReactJS差不多两年了。我也写React Natice。但是不要对我有所误解。我之前也曾为AngularJS疯狂。有大约一年的时间我是一个AngularJS的开发者。然后我换了一家使用ReactJS的新公司。后来的事你们都知道了。

<!--more-->

##长话短说 我渴望尝试使用Flutter

Google Flutter是一个新的平台，在这个平台之上你可以用一份Dart代码同时开发Android和iOS应用。迁移到新的开发栈之后，我知道做一个简单的应用，至少要优先处理的是状态管理。它把我引向了三个问题：

1.如何在widget树种向下传递一个应用的状态

2.如何在更新应用的状态之后对widget进行重建

3.如何在在页面之间跳转的同时保持状态同步。

## 执行

默认情况下，flutter会创建main.dart。这是程序运行的地方。由于我要创建两个页面直接的跳转，我额外创建了两个文件：MyHomePage.dart以及MySecondPage.dart。

这个应用程序的目的是让用户做以下几件事：

* 使MyHomePage中的计数器自增
* 跳转到MySecondPage
* 使MySecondPage中的计数器递减

这虽然看起来很简单，但是我要找到一种方式保持MyHomePage和MySecondPage之间计数器的同步。

计数器的初始值是0。如果在MyHomePage中用户自增了两次，当用户跳转到MySecondPage之后，计数器必须展示位2。

与此同时，如果在MySecondPage中用户自减了两次，当用户跳转到MyHomePage，计数器必须展示为0。

这叫做状态管理。

我得知Google Flutter有着setState()的机制，而这个机制React也有。这使我能更快地找出解决方案。

在我的main.dart文件中：

```dart
import 'package:flutter/material.dart';
import 'package:flutter_redux_example/screens/MyHomePage.dart';
void main() => runApp(new MyApp());
class MyApp extends StatefulWidget {
  
  @override
  _MyAppState createState() => new _MyAppState();
}
class _MyAppState extends State<MyApp> {
  int counter;
  @override
  void initState() {
    super.initState();
    counter = counter ?? 0;
  }
  void _decrementCounter(_) {
    setState(() {
      counter--;
      print('decrement: $counter');
    });
  }
  
  void _incrementCounter(_) {
    setState(() {
      counter++;
      print('increment: $counter');
    });
  }
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      title: 'Flutter Demo',
      theme: new ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: new MyHomePage(
        title: 'My Home Page',
        counter: counter,
        decrementCounter: _decrementCounter,
        incrementCounter: _incrementCounter,
      ),
    );
  }
}

```

从以上的代码，你可以看到_decrementCounter()和_incrementCounter()。它们是负责操作counter的值的，然后我把它们传递给了MyHomePage的构造器。

在我的MyHomePage.dart文件中：

```dart

import 'package:flutter/material.dart';
import 'package:flutter_redux_example/screens/MySecondPage.dart';
class MyHomePage extends StatefulWidget {
  MyHomePage({
    Key key,
    this.title,
    this.counter,
    this.decrementCounter,
    this.incrementCounter
  }) : super(key: key);
  final String title;
  final int counter;
  final ValueChanged<void> decrementCounter;
  final ValueChanged<void> incrementCounter;
  @override
  _MyHomePageState createState() => new _MyHomePageState();
}
class _MyHomePageState extends State<MyHomePage> {
  void _onPressed() {
    widget.incrementCounter(null);
  }
  @override
  Widget build(BuildContext context) {
    return new Scaffold(
      appBar: new AppBar(
        title: new Text(widget.title),
      ),
      body: new Center(
        child: new Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            new Text('You have pushed the button this many times:'),
            new Text(
              widget.counter.toString(),
              style: Theme.of(context).textTheme.display1,
            ),
            new RaisedButton(
              child: new Text('Next Screen'),
              onPressed: () {
                Navigator.push(
                  context,
                  new MaterialPageRoute(
                    builder: (context) => new MySecondPage(
                      widget.decrementCounter,
                      title: 'My Second Page',
                      counter: widget.counter,
                    ),
                  ),
                );
              },
            )
          ],
        ),
      ),
      floatingActionButton: new FloatingActionButton(
        onPressed: _onPressed,
        tooltip: 'Increment',
        child: new Icon(Icons.add),
      ),
    );
  }
}
```

从上述代码，你可以看到我把decrementCounter()向下传递给了MySecondPage的构造器。

在我的MySecondPage.dart文件中：

```dart
import 'package:flutter/material.dart';

class MySecondPage extends StatefulWidget {
  MySecondPage(
    this.decrementCounter, 
    {Key key, this.title, this.counter}
  ): super(key: key);
  final String title;
  final int counter;
  final ValueChanged<void> decrementCounter;
  @override
  _MySecondPageState createState() => new _MySecondPageState();
}
class _MySecondPageState extends State<MySecondPage> {
  void onPressed() {
    widget.decrementCounter(null);
  }
  @override
  Widget build(BuildContext context) {
    return new Scaffold(
      appBar: new AppBar(
        title: new Text(widget.title),
      ),
      body: new Center(
        child: new Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            new Text('You have pushed the button this many times :'),
            new Text(
              super.widget.counter.toString(),
              style: Theme.of(context).textTheme.display1,
            ),
          ],
        ),
      ),
    floatingActionButton: new FloatingActionButton(
      onPressed: onPressed,
      tooltip: 'Decrement',
      child: new Icon(Icons.indeterminate_check_box),
      backgroundColor: Colors.red),
    );
  }
}
```
## 结论

对于有着React背景的的开发者来说，玩弄Google Flutter中的基础状态管理不是已经特别难得事情。方法看起来很类似：存在一个setState()机制来对视图进行更新。StatefulWidget以及StatelessWidget的概念，对于我来说就行Component和PureComponent。Flutter把它们叫做Widget而React把它叫做Component。

那么，基本上，主要的问题就是 Google Flutter 所使用的编程语言了。由于它使用了Dart,我适应了Dart所拥有的语法和范式。目前为止，不是特别困难。对于我来说，Dart就像Java和Javascript生了一个孩子。

在这篇文章中，我设法解决我之前提及到的三个问题：

1.如何在widget树种向下传递一个应用的状态 /通过

2.如何在更新应用的状态之后对widget进行重建 /通过

3.如何在在页面之间跳转的同时保持状态同步。/通过

记住，这只是一个很简单的应用。在大型应用上事情会变得复杂很多。想象你要在应用所有widget中向下传递变量。这会是一件令人苦恼的事情。

在React环境下，我使用Redux来管理应用状态。我得知Redux也适用于Google Flutter。事实上，我已经成功地在Google Flutter应用上实现了Redux。让我们拭目以待我是否能发布关于此内容的文章。
