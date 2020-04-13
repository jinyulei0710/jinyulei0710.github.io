---
layout: post
title:  "(译)发送数据到新路由"
date:   2019-04-24 17:00:17 +0800
categories: 
   - Flutter 开发
tags:
    - navigation
---
[原文地址](https://flutter.dev/docs/cookbook/navigation/passing-data)

通常，我们不仅要导航到新路由，还要将一些数据传递到路由。例如，我们经常希望传递有关我们所使用的项目的信息。

请记住：多个路由仅仅是多个Widget。在这个例子中，我们将创建一个Todos列表。当点击待办事项时，我们将导航到显示有关待办事项信息的新路由（widget）。

<!--more-->


## 步骤

1. 定义Todo类
2. 显示待办事项列表
3. 创建一个详细信息路由，可以显示有关待办事项的信息
4. 导航并将数据传递到详细信息路由

## 1.定义Todo类

首先，我们需要一种简单的方式来表示Todo。对于此示例，我们将创建一个包含两个数据的类：标题和描述。

```dart

class Todo {
  final String title;
  final String description;

  Todo(this.title, this.description);
}
```


## 2.创建待办事项列表

其次，我们要显示Todo列表。在这个例子中，我们将生成20个待办事项并使用ListView显示它们。有关使用列表的更多信息，请参阅[基础列表](https://flutter.dev/docs/cookbook/lists/basic-list/)配方。

### 生成Todo列表

```dart
final todos = List<Todo>.generate(
  20,
  (i) => Todo(
        'Todo $i',
        'A description of what needs to be done for Todo $i',
      ),
);
```

### 使用ListView显示待办事项列表
```dart
ListView.builder(
  itemCount: todos.length,
  itemBuilder: (context, index) {
    return ListTile(
      title: Text(todos[index].title),
    );
  },
);
```

到现在为止还挺好。我们将生成20个Todos并在ListView中显示它们！


## 3.创建一个详细信息屏幕，可以显示有关待办事项的信息

现在，我们将创建第二个路由。路由标题将包含待办事项的标题，路由正文将显示说明。

由于它是一个普通的StatelessWidget，我们只需要创建Screen的用户传入Todo！然后，我们将使用给定的Todo构建UI。

```dart
class DetailScreen extends StatelessWidget {
  // Declare a field that holds the Todo
  final Todo todo;

  // In the constructor, require a Todo
  DetailScreen({Key key, @required this.todo}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    // Use the Todo to create our UI
    return Scaffold(
      appBar: AppBar(
        title: Text(todo.title),
      ),
      body: Padding(
        padding: EdgeInsets.all(16.0),
        child: Text(todo.description),
      ),
    );
  }
}
```

## 4.导航并将数据传递到详细信息路由

随着我们的`DetailScreen`到位，我们已准备好执行导航！在我们的例子中，当用户点击列表中的Todo时，我们将要导航到`DetailScreen`。当我们这样做时，我们也想将Todo传递给DetailScreen。

为此，我们将为ListTile Widget编写onTap回调。在我们的onTap回调中，我们将再次使用Navigator.push方法。

```dart
ListView.builder(
  itemCount: todos.length,
  itemBuilder: (context, index) {
    return ListTile(
      title: Text(todos[index].title),
      // When a user taps on the ListTile, navigate to the DetailScreen.
      // Notice that we're not only creating a DetailScreen, we're
      // also passing the current todo to it!
      onTap: () {
        Navigator.push(
          context,
          MaterialPageRoute(
            builder: (context) => DetailScreen(todo: todos[index]),
          ),
        );
      },
    );
  },
);
```

## 完整例子

```dart
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';

class Todo {
  final String title;
  final String description;

  Todo(this.title, this.description);
}

void main() {
  runApp(MaterialApp(
    title: 'Passing Data',
    home: TodosScreen(
      todos: List.generate(
        20,
        (i) => Todo(
              'Todo $i',
              'A description of what needs to be done for Todo $i',
            ),
      ),
    ),
  ));
}

class TodosScreen extends StatelessWidget {
  final List<Todo> todos;

  TodosScreen({Key key, @required this.todos}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Todos'),
      ),
      body: ListView.builder(
        itemCount: todos.length,
        itemBuilder: (context, index) {
          return ListTile(
            title: Text(todos[index].title),
            // When a user taps on the ListTile, navigate to the DetailScreen.
            // Notice that we're not only creating a DetailScreen, we're
            // also passing the current todo through to it!
            onTap: () {
              Navigator.push(
                context,
                MaterialPageRoute(
                  builder: (context) => DetailScreen(todo: todos[index]),
                ),
              );
            },
          );
        },
      ),
    );
  }
}

class DetailScreen extends StatelessWidget {
  // Declare a field that holds the Todo
  final Todo todo;

  // In the constructor, require a Todo
  DetailScreen({Key key, @required this.todo}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    // Use the Todo to create our UI
    return Scaffold(
      appBar: AppBar(
        title: Text(todo.title),
      ),
      body: Padding(
        padding: EdgeInsets.all(16.0),
        child: Text(todo.description),
      ),
    );
  }
}
```
![img](https://flutter.dev/images/cookbook/passing-data.gif)
