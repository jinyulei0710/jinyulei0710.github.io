---
layout: post
title:  "(译)异步编程: Streams "
date:   2019-04-28 11:00:17 +0800
categories: 
   - Flutter 开发
tags:
   - Dart
   - Async
---

[原文链接](https://dart.dev/tutorials/language/streams)

> 关键点有哪些？
> * Streams提供了一个异步数据序列。
* 数据序列包括用户生成的事件和从文件读取的数据。
* 你可以使用来自 Stream API 的 **await for** 或 `listen()` 来处理流。
*  Streams 提供了一种响应错误的方法。
* 有两种类型的流：单一订阅流和广播流。

<!--more-->


[Future](https://api.dartlang.org/stable/dart-async/Future-class.html) 和 [Stream](https://api.dartlang.org/stable/dart-async/Stream-class.html) 类是 Dart 异步编程的特点。

Future 表示不立即完成的计算。在普通函数返回结果的地方，异步函数返回一个 Future ，这个 Future 最终将包含结果。Future 会在结果准备就绪的时候告诉你。

流是一系列异步事件。它就像一个异步的 Iterable ，流在准备好事件的时候告诉你,而不是在你要求它时获得下一个事件。

## 接收流事件

流可以通过多种方式创建，这是另一篇文章的主题，但它们都可以以相同的方式使用：异步 for 循环(通常称为 **await for**）迭代流的事件就像 **for 循环** 在一个 Iterable 上迭代。例如：

```dart
Future<int> sumStream(Stream<int> stream) async {
  var sum = 0;
  await for (var value in stream) {
    sum += value;
  }
  return sum;
}
```

此代码只接收整数事件流的每个事件，将它们相加，然后返回总和。循环体结束时，函数暂停，直到下一个事件到达或流完成。

该函数使用 `async` 关键字标记，这在使用 **await for** 循环时是必需的。

以下示例通过使用 一个 `async*` 函数生成简单的整数流来测试前面的代码：

```dart
// Copyright (c) 2015, the Dart project authors.
// Please see the AUTHORS file for details.
// All rights reserved. Use of this source code is governed
// by a BSD-style license that can be found in the LICENSE file.

import 'dart:async';

Future<int> sumStream(Stream<int> stream) async {
  var sum = 0;
  await for (var value in stream) {
    sum += value;
  }
  return sum;
}

Stream<int> countStream(int to) async* {
  for (int i = 1; i <= to; i++) {
    yield i;
  }
}

main() async {
  var stream = countStream(10);
  var sum = await sumStream(stream);
  print(sum); // 55
}

```

## 错误事件

当流中没有更多事件时流就完成了，并且就像通知新事件到达一样接收事件的代码会被通知。使用 一个 **await for** 循环读取事件时，循环在流完成时停止。

在某些情况下，在流完成之前会发生错误;也许从远程服务器获取文件时网络失败，或者创建事件的代码可能存在错误，但有人需要了解它。

Streams 还可以提供错误事件，例如它提供数据事件。大多数流将在第一个错误之后停止，但是可能有多个流提供多个错误，并且流可以在错误事件之后提供更多数据。在本文档中，我们仅讨论最多传递一个错误的流。

使用 await for 读取流时，循环语句抛出错误。这也结束了循环。你可以使用 try-catch 捕获错误。以下示例在循环迭代器等于4时抛出错误：

## 使用 streams

Stream 类包含许多辅助方法，可以为你在流上执行常见操作，类似于[Iterable](https://api.dartlang.org/stable/dart-core/Iterable-class.html)上的方法。例如，你可以使用 Stream API 中的 lastWhere() 在流中找到最后一个正整数。

```dart
Future<int> lastPositive(Stream<int> stream) =>
    stream.lastWhere((x) => x >= 0);
```

## 两种类型的流

存在两种类型的流。

### 单订阅流
最常见的流包含一系列事件，这些事件是更大整体的一部分。事件需要以正确的顺序传递，而不会遗漏任何事件。这是你在读取文件或接收 Web 请求时获得的流。

这样的流只能被监听一次。稍后再次监听可能意味着错过了初始事件，然后流的其余部分毫无意义。当你开始监听时，将获取数据并以块的形式提供。

### 广播流

另一种流用于可以一次处理一个的单个消息。例如，这种流可以用于浏览器中的鼠标事件。

你可以随时开始监听此类流，然后你可以获得在监听时触发的事件。多个监听者可以同时监听，您可以在取消之前的订阅后再次监听。

## 处理流的方法

Stream\<T>上的以下方法处理流并返回结果：

```dart
Future<T> get first;
Future<bool> get isEmpty;
Future<T> get last;
Future<int> get length;
Future<T> get single;
Future<bool> any(bool Function(T element) test);
Future<bool> contains(Object needle);
Future<E> drain<E>([E futureValue]);
Future<T> elementAt(int index);
Future<bool> every(bool Function(T element) test);
Future<T> firstWhere(bool Function(T element) test, {T Function() orElse});
Future<S> fold<S>(S initialValue, S Function(S previous, T element) combine);
Future forEach(void Function(T element) action);
Future<String> join([String separator = ""]);
Future<T> lastWhere(bool Function(T element) test, {T Function() orElse});
Future pipe(StreamConsumer<T> streamConsumer);
Future<T> reduce(T Function(T previous, T element) combine);
Future<T> singleWhere(bool Function(T element) test, {T Function() orElse});
Future<List<T>> toList();
Future<Set<T>> toSet();
```
除了 `drain()` 和 `pipe()` 之外的所有这些函数都对应于 [Iterable](https://api.dartlang.org/stable/dart-core/Iterable-class.html) 上的类似函数。通过使用具有 **await for** 循环 的异步函数（或仅使用其他方法之一）,可以轻松地编写每个函数。例如，一些实现可能是：

```dart
Future<bool> contains(Object needle) async {
  await for (var event in this) {
    if (event == needle) return true;
  }
  return false;
}

Future forEach(void Function(T element) action) async {
  await for (var event in this) {
    action(event);
  }
}

Future<List<T>> toList() async {
  final result = <T>[];
  await this.forEach(result.add);
  return result;
}

Future<String> join([String separator = ""]) async =>
    (await this.toList()).join(separator);
```

（实际实现稍微复杂一些，但主要是出于历史原因。）

## 修改流的方法

Stream上的以下方法基于原始流返回新流。每个人都等待，直到有人在监听原始流之前监听新流。

```dart
Stream<R> cast<R>();
Stream<S> expand<S>(Iterable<S> Function(T element) convert);
Stream<S> map<S>(S Function(T event) convert);
Stream<R> retype<R>();
Stream<T> skip(int count);
Stream<T> skipWhile(bool Function(T element) test);
Stream<T> take(int count);
Stream<T> takeWhile(bool Function(T element) test);
Stream<T> where(bool Function(T event) test);
```

前面的方法对应于 [Iterable](https://api.dartlang.org/stable/dart-core/Iterable-class.html) 上的类似方法，它将 iterable 转换为另一个 iterable 。所有这些都可以使用带有 **await for** 循环的异步函数轻松编写。

```dart
Stream<E> asyncExpand<E>(Stream<E> Function(T event) convert);
Stream<E> asyncMap<E>(FutureOr<E> Function(T event) convert);
Stream<T> distinct([bool Function(T previous, T next) equals]);
```

`asyncExpand()` 和 `asyncMap()` 函数类似于 `expand()` 和 `map()` ，但允许其函数参数为异步函数。 `distinct()` 函数在 `Iterable` 上不存在，但它可以在流上有。

```dart
Stream<T> handleError(Function onError, {bool test(error)});
Stream<T> timeout(Duration timeLimit,
    {void Function(EventSink<T> sink) onTimeout});
Stream<S> transform<S>(StreamTransformer<T, S> streamTransformer);
```

最后三个功能更加特殊。它们涉及到 **await for** 循环无法做到的错误处理， - 到达循环的第一个错误将结束循环及其在流上的订阅。并且没有恢复。在 **await for** 循环中使用它之前，你可以使用 `handleError()` 从流中删除错误。

### transform() 方法

`transform()` 函数不仅用于错误处理;它是流的更通用的 “map”。通常的map要求每个传入事件都有一个值。但是，特别是对于 I/O 流，可能需要多个传入事件才能生成输出事件。 [StreamTransformer](https://api.dartlang.org/stable/dart-async/StreamTransformer-class.html)可以使用它。例如，像[Utf8Decoder](https://api.dartlang.org/stable/dart-convert/Utf8Decoder-class.html)这样的解码器就是变换器。变换器只需要一个函数[bind()](https://api.dartlang.org/stable/dart-async/StreamTransformer/bind.html)，它可以通过异步函数轻松实现。

```dart
Stream<S> mapLogErrors<S, T>(
  Stream<T> stream,
  S Function(T event) convert,
) async* {
  var streamWithoutErrors = stream.handleError((e) => log(e));
  await for (var event in streamWithoutErrors) {
    yield convert(event);
  }
}
```

### 读取和解析一个文件

以下代码读取文件并在流上运行两个转换。它首先从UTF8转换数据，然后通过[LineSplitter](https://api.dartlang.org/stable/dart-convert/LineSplitter-class.html)运行它。打印所有行，除了以＃标签开头的任何行＃。

```dart
import 'dart:convert';
import 'dart:io';

Future<void> main(List<String> args) async {
  var file = File(args[0]);
  var lines = file
      .openRead()
      .transform(utf8.decoder)
      .transform(LineSplitter());
  await for (var line in lines) {
    if (!line.startsWith('#')) print(line);
  }
}

```

## 监听方法

Stream上的 final 方法是 `listen()`。这是一种“底层”方法 - 所有其他流函数都是根据`listen()` 定义的。

```dart
StreamSubscription<T> listen(void Function(T event) onData,
    {Function onError, void Function() onDone, bool cancelOnError});
```

要创建新的 Stream 类型，你可以扩展 `Stream` 类并实现 `listen()` 方法 - ` Stream` 所有其他方法为了能工作都要调用 `listen()`。

`listen()` 方法允许你开始监听流。在你这样做之前，流是一个惰性对象，描述你想要查看的事件。监听时，将返回 [StreamSubscription](https://api.dartlang.org/stable/dart-async/StreamSubscription-class.html) 对象，该对象表示生成事件的活动流。这类似于 `Iterable` 只是一个对象集合，但迭代器进行实际迭代。

流订阅允许你暂停订阅，暂停后恢复它，并完全取消订阅。你可以为每个数据事件或错误事件以及关闭流时设置回调。

## 其他资源

有关在 Dart 中使用流和异步编程的更多详细信息，请阅读以下文档。

* [在 Dart 中创建流](/2019/04/30/Dart/Async/Creating%20Streams%20in%20Dart/)，这是一篇关于创建自己的流的文章
* [ Futures 和错误处理](/2019/04/29/Dart/Async/Futures%20and%20Error%20Handling/)，这篇文章解释了如何使用 Future API 处理错误
* [异步支持](/2019/05/04/Dart/Async/Asynchrony%20support/)，[语言导览](https://www.dartlang.org/guides/language/language-tour)中的一个部分
* [流 API 参考](https://api.dartlang.org/stable/dart-async/Stream-class.html)
