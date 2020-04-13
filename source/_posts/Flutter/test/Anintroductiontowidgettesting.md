---
layout: post
title:  "(译)widget测试简介"
date:   2019-04-19 17:55:17 +0800
categories:  
   - Flutter 开发
tags:
    - 测试
---
[原文链接](https://flutter.dev/docs/cookbook/testing/widget/introduction)

在[单元测试介绍](/flutter/test/2019/04/19/An-introduction-to-unit-testing)中,我们学习了如何使用test包测试Dart类。为了测试Widget类，我们需要一些flutter测试包提供的附加工具，它是随FlutterSDK一起提供的。

flutter_test包提供了以下用于测试widget的工具：

* WidgetTester，它允许我们在测试环境中构建Widgets并与之交互。
* testWidgets方法。此方法将自动为每个测试用例创建一个新的WidgetTester，并用于代替正常的测试功能。
* Finder类。这些类允许我们在测试环境中查找widget。
* 特定于 widget Matcher常量，可帮助我们验证Finder是否在测试环境中找到widget或多个widget。

<!--more-->


如果这听起来很难对付，请不要担心！我们将在整个配方中看到所有这些零件是如何组合在一起的。

## 路径

1. 添加`flutter_test`依赖项
2. 创建一个要测试的小部件
3. 创建一个`testWidgets`测试
4. 使用`WidgetTester`构建Widget
4. 使用`Finder`查找我们的小部件
5. 使用`Matcher`验证我们的Widget是否正常工作

## 1. 添加flutter_test依赖

在我们开始编写测试之前，我们需要在pubspec.yaml文件的`dev_dependencies`部分中包含`flutter_test`依赖项。如果使用命令行工具或代码编辑器创建新的Flutter项目，则此依赖项应该已经到位！

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
```

## 2.创建一个要测试的Widget

接下来，我们需要创建一个我们可以测试的Widget！对于这个配方，我们将创建一个显示标题和消息的widget。

```dart
class MyWidget extends StatelessWidget {
  final String title;
  final String message;

  const MyWidget({
    Key key,
    @required this.title,
    @required this.message,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      home: Scaffold(
        appBar: AppBar(
          title: Text(title),
        ),
        body: Center(
          child: Text(message),
        ),
      ),
    );
  }
}
```
## 3.创建一个testWidget测试

现在我们有一个Widget来测试，我们可以开始编写我们的第一个测试了！首先，我们将使用flutter_test包提供的testWidgets方法来定义测试。 testWidgets方法将允许我们定义一个Widget测试，并将创建一个WidgetTester供我们使用。

我们的测试将验证MyWidget是否显示给定的标题和消息。

```dart
void main() {
  // Define a test. The TestWidgets function will also provide a WidgetTester
  // for us to work with. The WidgetTester will allow us to build and interact
  // with Widgets in the test environment.
  testWidgets('MyWidget has a title and message', (WidgetTester tester) async {
    // Test code will go here!
  });
}
```

## 4.使用WidgetTester构建Widget
接下来，我们将要在测试环境中构建MyWidget。为此，我们可以使用WidgetTester提供的pumpWidget方法。 pumpWidget方法将构建并渲染我们提供的Widget。

在这种情况下，我们将创建一个MyWidget实例，将“T”显示为标题，将“M”显示为消息。

```dart
void main() {
  testWidgets('MyWidget has a title and message', (WidgetTester tester) async {
    // Create the Widget tell the tester to build it
    await tester.pumpWidget(MyWidget(title: 'T', message: 'M'));
  });
}
```
注意
在初始调用`pumpWidget`之后，`WidgetTester`提供了重建相同Widget的其他方法。如果您正在使用StatefulWidget或动画，这将非常有用。

例如，如果我们点击一​​个按钮，并且此按钮调用setState，Flutter将不会在测试环境中自动重建您的Widget。我们需要使用以下方法之一来让Flutter再次构建我们的Widget。

* tester.pump（）
在给定的持续时间后触发Widget的重建。
* tester.pumpAndSettle（）
在给定的持续时间内反复呼叫泵，直到不再安排任何帧。这基本上要等待所有动画完成。

这些方法提供了对构建生命周期的细粒度控制，这在测试时特别有用。

##  5.使用`Finder`查找我的Widget

现在我们已经在测试环境中构建了Widget，我们想要使用Finder在Widget树中查找标题和消息Text Widgets。这将允许我们验证我们是否正确显示这些widget！

在这种情况下，我们将使用flutter_test包提供的顶级find方法来创建Finders。由于我们知道我们正在查找Textwidget，我们可以使用find.text方法。

有关Finder类的更多信息，请参阅[widget测试配方中的查找widget](https://flutter.dev/docs/cookbook/testing/widget/finders)。

```dart
void main() {
  testWidgets('MyWidget has a title and message', (WidgetTester tester) async {
    await tester.pumpWidget(MyWidget(title: 'T', message: 'M'));

    // Create our Finders
    final titleFinder = find.text('T');
    final messageFinder = find.text('M');
  });
}
```

## 6. 使用Matcher验证我们的Widget是否正常工作
最后，我们可以使用flutter_test提供的Matcher常量验证标题和消息Text Widgets在屏幕上显示。 Matcher类是测试包的核心部分，并提供了一种通用方法来验证给定值是否符合我们的期望。

在这种情况下，我们希望确保我们的widget只出现在屏幕上一次。因此，我们可以使用findsOneWidget Matcher。

```dart
void main() {
  testWidgets('MyWidget has a title and message', (WidgetTester tester) async {
    await tester.pumpWidget(MyWidget(title: 'T', message: 'M'));
    final titleFinder = find.text('T');
    final messageFinder = find.text('M');

    // Use the `findsOneWidget` matcher provided by flutter_test to verify our
    // Text Widgets appear exactly once in the Widget tree
    expect(titleFinder, findsOneWidget);
    expect(messageFinder, findsOneWidget);
  });
}
```

## 其他Matcher

除了findsOneWidget之外，flutter_test还为常见案例提供了其他的Matcher。

* findsNothing
验证没有找到Widgets
* findsWidgets
验证是否找到一个或多个小部件
* findsNWidgets
验证是否找到特定数量的小部件

## 完整代码

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  // Define a test. The TestWidgets function will also provide a WidgetTester
  // for us to work with. The WidgetTester will allow us to build and interact
  // with Widgets in the test environment.
  testWidgets('MyWidget has a title and message', (WidgetTester tester) async {
    // Create the Widget tell the tester to build it
    await tester.pumpWidget(MyWidget(title: 'T', message: 'M'));

    // Create our Finders
    final titleFinder = find.text('T');
    final messageFinder = find.text('M');

    // Use the `findsOneWidget` matcher provided by flutter_test to verify our
    // Text Widgets appear exactly once in the Widget tree
    expect(titleFinder, findsOneWidget);
    expect(messageFinder, findsOneWidget);
  });
}

class MyWidget extends StatelessWidget {
  final String title;
  final String message;

  const MyWidget({
    Key key,
    @required this.title,
    @required this.message,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      home: Scaffold(
        appBar: AppBar(
          title: Text(title),
        ),
        body: Center(
          child: Text(message),
        ),
      ),
    );
  }
}
```
