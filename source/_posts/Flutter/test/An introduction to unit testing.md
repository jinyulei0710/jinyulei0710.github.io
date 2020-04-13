---
layout: post
title:  "(译)单元测试介绍"
date:   2019-04-19 10:55:17 +0800
categories: 
   - Flutter 开发
tags:
    - 测试
---

[原文链接](https://flutter.dev/docs/cookbook/testing/unit/introduction)

如何确保在添加更多功能或更改现有功能时，应用程序继续正常工作？通过编写测试。

单元测试可以方便地验证单个函数，方法或类的行为。[test](https://pub.dartlang.org/packages/test)包提供了编写单元测试的核心框架，[flutter_test](https://api.flutter.dev/flutter/flutter_test/flutter_test-library.html)包提供了用于测试Widgets的其他实用工具。

此方法演示了`test`包提供的核心功能。有关test包的更多信息，请参阅[test包文档](https://github.com/dart-lang/test/blob/master/README.md)。

<!--more-->


## 路线
1.添加test或flutter_test依赖项

2.创建一个测试文件

3.创建一个要测试的类

4.为我们的类写一个测试

5.在一个组中组合多个测试

6.运行测试

## 1. 添加测试依赖

如果您正在处理不依赖于Flutter的Dart包，则可以导入测试包。测试包提供了在Dart中编写测试的核心功能。在编写将由Web，服务器和Flutter应用程序使用的包时，这是最好的方法。

```dart
dev_dependencies:
  test: <latest_version>
```
## 2.创建一个测试文件

在此示例中，创建两个文件：`counter.dart`和`counter_test.dart`。

`counter.dart`文件将包含您要测试的类，并归属于lib文件夹。`counter_test.dart`文件将包含多个测试并存在于`test`文件夹中。

通常，测试文件应该归属于于Flutter应用程序或程序包根目录的test文件夹中。

完成后，文件夹结构应如下所示：

```yaml
counter_app/
  lib/
    counter.dart
  test/
    counter_test.dart
```

## 3. 创建一个类来对它进行测试

接下来，您需要一个“单位”来测试。请记住：“unit”是函数，方法或类的设想名称。在此示例中，在lib/counter.dart文件中创建Counter类。它将负责递增和递减从0开始的值。

```dart
class Counter {
  int value = 0;

  void increment() => value++;

  void decrement() => value--;
}
```

注意：为简单起见，本教程不遵循“测试驱动开发”方法。如果你对这种开发方式更加满意，你可以随时走这条路。

## 4. 为我们的类写一个测试

在counter_test.dart文件中，编写第一个单元测试。测试是由顶级测试功能定义的，您可以使用顶级expect功能检查结果是否正确。这两个功能都来自`test`包。

```dart
// Import the test package and Counter class
import 'package:test/test.dart';
import 'package:counter_app/counter.dart';

void main() {
  test('Counter value should be incremented', () {
    final counter = Counter();

    counter.increment();

    expect(counter.value, 1);
  });
}
```

## 5. 在一个组中组合多个测试
如果您有多个彼此相关的测试，请使用`test`包提供的组功能将它们组合在一起。

```dart
import 'package:test/test.dart';
import 'package:counter_app/counter.dart';

void main() {
  group('Counter', () {
    test('value should start at 0', () {
      expect(Counter().value, 0);
    });

    test('value should be incremented', () {
      final counter = Counter();

      counter.increment();

      expect(counter.value, 1);
    });

    test('value should be decremented', () {
      final counter = Counter();

      counter.decrement();

      expect(counter.value, -1);
    });
  });
}
```

## 6.运行测试

现在Counter类和test已经就位了，你可以运行测试了。

### 使用IntelliJ或VSCode运行测试
IntelliJ和VSCode的Flutter插件支持运行测试。这通常是编写测试时的最佳选择，因为它提供了最快的反馈循环以及设置断点的能力。

* IntelliJ

1. 打开`counter_test.dart`文件
2. 选择`Run`菜单
3. 单击`Run'test in counter_test.dart'`选项
4. 或者，针对你的平台使用适当的快捷键。

* VSCode
1. 打开`counter_test.dart`文件
2. 选择`Debug`菜单
3. 单击`start Debugging`选项
4. 或者，针对你的平台使用适当的快捷键。

### 在终端中运行测试
您还可以使用终端通过从项目的根目录执行以下命令来运行测试：

```
flutter test test/counter_test.dart
```
