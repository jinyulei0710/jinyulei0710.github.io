---
layout: post
title:  "(译)查找widget"
date:   2019-04-19 17:55:17 +0800
categories:  
   - Flutter 开发
tags:
    - 测试
---

[原文链接](https://flutter.dev/docs/cookbook/testing/widget/finders)


为了在测试环境中定位Widget，我们需要使用`Finder`类。虽然可以编写我们自己的`Finder`类，但使用[flutter_test](https://api.flutter.dev/flutter/flutter_test/flutter_test-library.html)包提供的工具定位Widgets通常更方便。

在这个配方中，我们将查看`flutter_test`包提供的`find`常量，并演示如何使用它提供的一些Finder。有关可用查找器的完整列表，请参阅[CommonFinders文档](https://api.flutter.dev/flutter/flutter_driver/CommonFinders-class.html)。

如果您不熟悉Widget测试和`Finder`类的角色，请查看[Widget测试简介](/flutter/test/2019/04/19/Anintroductiontowidgettesting/)。

<!--more-->


## 路径

1. 查找一个文本Widget
2. 查找带有特定key的Widget
3. 查找一个特定的widget实例

## 1.查找一个文本widget

在我们的测试中，我们经常需要找到包含特定文本的widget。这正是find.text方法的用途。它将创建一个Finder，用于搜索显示特定文本字符串的widget。

```dart
testWidgets('finds a Text Widget', (WidgetTester tester) async {
  // Build an App with a Text Widget that displays the letter 'H'
  await tester.pumpWidget(MaterialApp(
    home: Scaffold(
      body: Text('H'),
    ),
  ));

  // Find a Widget that displays the letter 'H'
  expect(find.text('H'), findsOneWidget);
});
```

## 2.查找带有特定key的Widget

在某些情况下，我们可能希望根据已提供给它的Key找到一个Widget。如果我们显示相同Widget的多个实例，这可能很方便。例如，我们可能有一个ListView，它显示包含相同文本的多个Text Widgets。

在这种情况下，我们可以为列表中的每个Widget提供一个Key。这将允许我们唯一地标识特定的Widget，从而更容易在测试环境中找到Widget。

```dart
testWidgets('finds a Widget using a Key', (WidgetTester tester) async {
  // Define our test key
  final testKey = Key('K');

  // Build a MaterialApp with the testKey
  await tester.pumpWidget(MaterialApp(key: testKey, home: Container()));

  // Find the MaterialApp Widget using the testKey
  expect(find.byKey(testKey), findsOneWidget);
});
```

## 3.查找一个特定的widget实例

最后，我们可能对定位特定的Widget实例感兴趣。例如，在创建带有`child`属性的Widget时候，并且我们想要确保我们渲染了子Widget时，这可能很有用。

```dart
testWidgets('finds a specific instance', (WidgetTester tester) async {
  final childWidget = Padding(padding: EdgeInsets.zero);

  // Provide our childWidget to the Container
  await tester.pumpWidget(Container(child: childWidget));

  // Search for the childWidget in the tree and verify it exists
  expect(find.byWidget(childWidget), findsOneWidget);
});
```
## 总结

`flutter_test`包提供的`find`常量为我们提供了几种在测试环境中定位Widget的方法。该配方展示了这些方法中的三种，并且存在多种用于不同目的的方法中。

如果上述示例不适用于特定用例，请参阅[CommonFinders文档](https://api.flutter.dev/flutter/flutter_driver/CommonFinders-class.html)以查看所有可用方法。

## 完整代码

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets('finds a Text Widget', (WidgetTester tester) async {
    // Build an App with a Text Widget that displays the letter 'H'
    await tester.pumpWidget(MaterialApp(
      home: Scaffold(
        body: Text('H'),
      ),
    ));

    // Find a Widget that displays the letter 'H'
    expect(find.text('H'), findsOneWidget);
  });

  testWidgets('finds a Widget using a Key', (WidgetTester tester) async {
    // Define our test key
    final testKey = Key('K');

    // Build a MaterialApp with the testKey
    await tester.pumpWidget(MaterialApp(key: testKey, home: Container()));

    // Find the MaterialApp Widget using the testKey
    expect(find.byKey(testKey), findsOneWidget);
  });

  testWidgets('finds a specific instance', (WidgetTester tester) async {
    final childWidget = Padding(padding: EdgeInsets.zero);

    // Provide our childWidget to the Container
    await tester.pumpWidget(Container(child: childWidget));

    // Search for the childWidget in the tree and verify it exists
    expect(find.byWidget(childWidget), findsOneWidget);
  });
}

```
