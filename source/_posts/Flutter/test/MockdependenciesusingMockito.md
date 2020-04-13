---
layout: post
title:  "(译)使用Mockito模拟外部依赖"
date:   2019-04-19 11:55:17 +0800
categories:   
   - Flutter 开发
tags:
    - 测试
---

[原文链接](https://flutter.dev/docs/cookbook/testing/unit/mocking)

在某些情况下，单元测试可能依赖于从实时Web服务或数据库获取数据的类。
这是很不方便的，以下给出了几个原因：

* 调用实时服务或数据库会降低测试执行速度。
* 如果Web服务或数据库返回意外结果，则目前的测试可能会失败。这被称为“薄片测试”。
* 使用实时Web服务或数据库很难测试所有可能的成功和失败场景。

因此，您可以“模拟”这些外部依赖，而不是依赖于实时Web服务或数据库。模拟允许我们模拟实时Web服务或数据库，并根据情况返回特定结果。

一般来说，您可以通过创建类的替代实现来模拟依赖项。您可以手动编写这些替代实现，也可以使用[Mockito包](https://pub.dartlang.org/packages/mockito)作为快捷方式。

本文演示了使用Mockito软件包进行模拟的基础知识。有关详细信息，请参阅[Mockito软件包文档](https://pub.dartlang.org/packages/mockito)。

<!--more-->


## 路线

1. 添加`mockito`以及`test`依赖
2. 创建一个用来测试的方法
3. 创建一个带有模拟`http.Client`的test文件
4. 为每种情况写一个test
5. 运行测试

## 1.添加mockito依赖
要使用`mockito`包，首先需要将其与dev_dependencies部分中的flutter_test依赖项一起添加到`pubspec.yaml`文件中。

您还将在此示例中使用`http`包，并将在dependencies部分中定义该依赖项。

```yaml
dependencies:
  http: <newest_version>
dev_dependencies:
  test: <newest_version>
  mockito: <newest_version>
```

## 2.创建一个用来测试的方法

在此示例中，您将需要从Internet配方的Fetch数据中对fetchPost函数进行单元测试。要测试此功能，您需要进行两项更改：

1. 为方法提供http.Client。这允许您基于不同情况提供正确的http.Client。对于Flutter和服务器端的项目，您可以提供http.IOClient。对于浏览器应用程序，您可以提供http.BrowserClient。对于测试，您提供模拟的http.Client。
2. 使用提供的客户端从Internet获取数据，而不是静态的http.get方法，因为这很难模拟。

该方法现在应该如下所示：

```dart
Future<Post> fetchPost(http.Client client) async {
  final response =
      await client.get('https://jsonplaceholder.typicode.com/posts/1');

  if (response.statusCode == 200) {
    // If the call to the server was successful, parse the JSON
    return Post.fromJson(json.decode(response.body));
  } else {
    // If that call was not successful, throw an error.
    throw Exception('Failed to load post');
  }
}
```
## 3.创建一个带有模拟的`http.client`的测试文件

接下来，创建一个测试文件以及`MockClient`类。按照[单元测试配方简介中](/flutter/test/2019/04/19/An-introduction-to-unit-testing.html)的建议，在根`test`文件夹中创建一个名为`fetch_post_test.dart`的文件。

MockClient类实现`http.Client`类。这允许您将`MockClient`传递给`fetchPost`函数，并允许您在每个测试中返回不同的http响应。

```dart
// Create a MockClient using the Mock class provided by the Mockito package.
// Create new instances of this class in each test.
class MockClient extends Mock implements http.Client {}

main() {
  // Tests go here
}
```

## 4.为每种情况写一个test

如果你考虑下fetchPost函数，它将做两件事中的一件：

1. 如果http调用成功，则返回一个`Post`
2. 如果http调用失败，则抛出异常

因此，你需要针对这两种情况进行测试。您可以使用MockClient类为成功测试返回“Ok”响应，并为不成功的测试返回错误响应。

为此，请使用Mockito提供的when方法。

```dart
// Create a MockClient using the Mock class provided by the Mockito package.
// Create new instances of this class in each test.
class MockClient extends Mock implements http.Client {}

main() {
  group('fetchPost', () {
    test('returns a Post if the http call completes successfully', () async {
      final client = MockClient();

      // Use Mockito to return a successful response when it calls the
      // provided http.Client.
      when(client.get('https://jsonplaceholder.typicode.com/posts/1'))
          .thenAnswer((_) async => http.Response('{"title": "Test"}', 200));

      expect(await fetchPost(client), isInstanceOf<Post>());
    });

    test('throws an exception if the http call completes with an error', () {
      final client = MockClient();

      // Use Mockito to return an unsuccessful response when it calls the
      // provided http.Client.
      when(client.get('https://jsonplaceholder.typicode.com/posts/1'))
          .thenAnswer((_) async => http.Response('Not Found', 404));

      expect(fetchPost(client), throwsException);
    });
  });
}
```

## 5.运行测试

现在你的fetchPost()方法和测试都就位了，运行测试。

```
$ dart test/fetch_post_test.dart
```
您还可以按照[单元测试配方简介](/flutter/test/2019/04/19/An-introduction-to-unit-testing.html)中的说明的那样，在你喜欢的编辑器中运行测试。

## 总结

在此示例中，您已经学习了如何使用Mockito来测试依赖于Web服务或数据库的函数或类。这只是对Mockito库和模拟概念的简短介绍。有关更多信息，请参阅[Mockito包](https://pub.dartlang.org/packages/mockito)提供的文档。