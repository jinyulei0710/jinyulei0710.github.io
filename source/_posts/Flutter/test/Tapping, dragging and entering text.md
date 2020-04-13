---
layout: post
title:  "(译)点击，拖拽以及输入文本"
date:   2019-04-20 06:22:17 +0800
categories:
   - Flutter 开发
tags:
    - 测试
---

[原文链接](https://flutter.dev/docs/cookbook/testing/widget/tap-drag)

我们构建的许多widget不仅显示信息，还要响应用户交互。这包括用户可以点击的按钮，在屏幕上拖动条目或在TextField中输入文本。

为了测试这些交互，我们需要一种在测试环境中模拟它们的方法。为此，我们可以使用[flutter_test](https://api.flutter.dev/flutter/flutter_test/flutter_test-library.html)库提供的[WidgetTester](https://api.flutter.dev/flutter/flutter_test/flutter_test-library.html)类。

`WidgetTester`提供了输入文本，点击和拖动的方法。

<!--more-->


* enterText
* tap
* darg

在许多情况下，用户交互将更新我们的应用程序的状态。在测试环境中，当状态更改时，Flutter不会自动重建widget。为了确保在模拟用户交互之后重建我们的Widget树，我们必须调用`WidgetTester`提供的[pump](https://api.flutter.dev/flutter/flutter_test/WidgetTester/pump.html)或[pumpAndSettle](https://api.flutter.dev/flutter/flutter_test/WidgetTester/pumpAndSettle.html)方法。

## 路径

1. 创建一个要测试的widget
2. 在文本字段中输入文本
3. 确保点击按钮添加待办事项
4. 确保滑动删除会删除待办事项

## 1.创建一个要测试的widget

对于此示例，我们将创建一个基本的待办事项应用程序。它将有三个我们想要测试的主要功能：

1. 在TextField中输入文本
2. 点击FloatingActionButton会将文本添加到待办事项列表中
3. 滑动删除会从列表中删除该条目

为了将重点放在测试上，此配方将不提供有关如何构建待办事项应用程序的详细指南。要了解有关如何构建此应用程序的更多信息，请参阅相关配方：

* [创建文本字段并设置样式](https://flutter.dev/docs/cookbook/forms/text-input/)
* [处理点击事件](https://flutter.dev/docs/cookbook/gestures/handling-taps/)
* [创建一个基本列表](https://flutter.dev/docs/cookbook/lists/basic-list/)
* [实现滑动删除](https://flutter.dev/docs/cookbook/gestures/dismissible/)

```dart
class TodoList extends StatefulWidget {
  @override
  _TodoListState createState() => _TodoListState();
}

class _TodoListState extends State<TodoList> {
  static const _appTitle = 'Todo List';
  final todos = <String>[];
  final controller = TextEditingController();

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: _appTitle,
      home: Scaffold(
        appBar: AppBar(
          title: Text(_appTitle),
        ),
        body: Column(
          children: [
            TextField(
              controller: controller,
            ),
            Expanded(
              child: ListView.builder(
                itemCount: todos.length,
                itemBuilder: (BuildContext context, int index) {
                  final todo = todos[index];

                  return Dismissible(
                    key: Key('$todo$index'),
                    onDismissed: (direction) => todos.removeAt(index),
                    child: ListTile(title: Text(todo)),
                    background: Container(color: Colors.red),
                  );
                },
              ),
            ),
          ],
        ),
        floatingActionButton: FloatingActionButton(
          onPressed: () {
            setState(() {
              todos.add(controller.text);
              controller.clear();
            });
          },
          child: Icon(Icons.add),
        ),
      ),
    );
  }
}
```

## 2.在`text field`中输入文本
现在我们有了一个todo应用程序，我们就可以开始编写测试了！在这种情况下，我们首先在`TextField`中输入文本。

我们可以通过以下方式完成此任务

1. 在测试环境中构建Widget
2. 使用`WidgetTester`中的[enterText](https://api.flutter.dev/flutter/flutter_test/WidgetTester/enterText.html)方法

```dart
testWidgets('Add and remove a todo', (WidgetTester tester) async {
  // Build the Widget
  await tester.pumpWidget(TodoList());

  // Enter 'hi' into the TextField
  await tester.enterText(find.byType(TextField), 'hi');
});
```
注意：此配方基于之前的Widget测试配方。要了解Widget测试的核心概念，请参阅以下配方：

* [Widget测试简介](/flutter/test/2019/04/19/Anintroductiontowidgettesting/)
* [在widget测试中查找widget](/flutter/test/2019/04/19/Finding-widgets/)

## 3.确保点击按钮添加待办事项
在我们将文本输入TextField之后，我们要确保点击FloatingActionButton将该项添加到列表中。

这将涉及三个步骤：

1. 使用tap方法点击添加按钮
2. 使用pump方法更改状态后重建Widget
3. 确保列表项显示在页面上

```dart
testWidgets('Add and remove a todo', (WidgetTester tester) async {
  // Enter text code...

  // Tap the add button
  await tester.tap(find.byType(FloatingActionButton));

  // Rebuild the Widget after the state has changed
  await tester.pump();

  // Expect to find the item on screen
  expect(find.text('hi'), findsOneWidget);
});
```

## 4. 确保滑动删除会删除待办事项

最后，我们可以确保对todo项执行滑动删除操作会将其从列表中删除。这将涉及三个步骤：

* 使用[drag](https://api.flutter.dev/flutter/flutter_test/WidgetController/drag.html)方法执行滑动到解除操作。
* 使用[pumpAndSettle](https://api.flutter.dev/flutter/flutter_test/WidgetTester/pumpAndSettle.html)方法不断重建我们的Widget树，直到dismiss动画完成。
* 确保屏幕上不再显示该条目。

```dart
testWidgets('Add and remove a todo', (WidgetTester tester) async {
  // Enter text and add the item...

  // Swipe the item to dismiss it
  await tester.drag(find.byType(Dismissible), Offset(500.0, 0.0));

  // Build the Widget until the dismiss animation ends
  await tester.pumpAndSettle();

  // Ensure the item is no longer on screen
  expect(find.text('hi'), findsNothing);
});
```

## 完整代码

一旦我们完成了这些步骤，我们应该有了带有测试的一个工作的应用程序，其测试部分能确保它正常工作！

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets('Add and remove a todo', (WidgetTester tester) async {
    // Build the Widget
    await tester.pumpWidget(TodoList());

    // Enter 'hi' into the TextField
    await tester.enterText(find.byType(TextField), 'hi');

    // Tap the add button
    await tester.tap(find.byType(FloatingActionButton));

    // Rebuild the Widget with the new item
    await tester.pump();

    // Expect to find the item on screen
    expect(find.text('hi'), findsOneWidget);

    // Swipe the item to dismiss it
    await tester.drag(find.byType(Dismissible), Offset(500.0, 0.0));

    // Build the Widget until the dismiss animation ends
    await tester.pumpAndSettle();

    // Ensure the item is no longer on screen
    expect(find.text('hi'), findsNothing);
  });
}

class TodoList extends StatefulWidget {
  @override
  _TodoListState createState() => _TodoListState();
}

class _TodoListState extends State<TodoList> {
  static const _appTitle = 'Todo List';
  final todos = <String>[];
  final controller = TextEditingController();

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: _appTitle,
      home: Scaffold(
        appBar: AppBar(
          title: Text(_appTitle),
        ),
        body: Column(
          children: [
            TextField(
              controller: controller,
            ),
            Expanded(
              child: ListView.builder(
                itemCount: todos.length,
                itemBuilder: (BuildContext context, int index) {
                  final todo = todos[index];

                  return Dismissible(
                    key: Key('$todo$index'),
                    onDismissed: (direction) => todos.removeAt(index),
                    child: ListTile(title: Text(todo)),
                    background: Container(color: Colors.red),
                  );
                },
              ),
            ),
          ],
        ),
        floatingActionButton: FloatingActionButton(
          onPressed: () {
            setState(() {
              todos.add(controller.text);
              controller.clear();
            });
          },
          child: Icon(Icons.add),
        ),
      ),
    );
  }
}
```
