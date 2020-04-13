---
layout: post
title:  "(译)Dart中适用于不可修改集合的built_collection"
date:   2019-04-15 11:20:17 +0800
categories: 
   - Flutter 开发
tags:
    - JSON序列化
    - built_value
---
[原文链接](https://medium.com/dartlang/darts-built-collection-for-immutable-collections-db662f705eff)

在2016年的Dart开发者大会上，我谈论了关于[使用source_gen来生成对象模型的内容](https://www.youtube.com/watch?v=TMeJxWltoVo)。我提到了一些非常值得仔细研究的软件包和技术，所以在这里我要讲完整个故事。

<!--more-->


首先:[built_collection](https://github.com/google/built_collection.dart)

built_collection包提供类似SDK的具有不可变性的集合。不可变性的优点在于:

* 简单，方便。可以传递一个不可变的集合，而不必担心谁可能会修改它。
* 性能。可变性导致昂贵的模式，如防御性复制和变化检测。

build_collection应该很方便的这个事实对API提出了一些挑战。让我们来看看。首先，一个不可变的列表：

```dart
var list = new BuiltList<int>([1, 2, 3]);
```

现在我想为它添加一个值。当然，它是不可改变的，所以我不能把值添加上去;我真正想要的是创建一个在最后添加一个额外的值的新的列表。

这里我们使用到了builder模式，也是此包命名的由来。通过转换为builder，在builder上进行修改，然后构建从而获得新列表：

```dart
var builder = list.toBuilder();
builder.add(4);
var newList = builder.build();
```
Buidlers通常是内联使用的，让我们这么做：

```dart
var newList = (list.toBuilder()..add(4)).build()
```
这就是构建器模式通常所做的，但是Dart有lambdas，所以我们可以做得更好：

```dart
var newList = list.rebuild((b) => b.add(4));
```

我们发现在Dart中级联的方法使得构建器模式非常强大。你几乎总能做你想要的内联：

```dart
var newList = list.rebuild((b) => b
    ..add(4)
    ..addAll([7, 6, 5])
    ..sort()
    ..remove(1));
```

所以，正如所希望的，built_collection的使用时很方便的。

___


让我们谈论下性能。

Built Collections目前针对具有更新/发布周期的代码进行了优化。这意味着：集合将转换为构建器，更新，构建，然后发布 - 传递引用 - 以供其他代码使用。

这在Web应用程序是很典型的，例如Angular应用程序。在应用中，用户交互或RPC响应会触发更新。数据模型在发布以进行渲染之前会重建。

Built Collections尚未针对非常频繁的重建进行优化，这需要在精心选择的数据结构之上实现。如果你有对此有使用实例，请联系我 - 这是Built Collections设计的一部分，这将在需要时实现。

___

除了不变性之外，built_collection还可以帮助您编写正确的代码。有着一整套的帮助属性在这里：

* 必须使用显式类型参数创建Built Collections;没有BuiltList <dynamic>这样的东西。

```dart
new BuiltList([1, 2, 3]);     // Throws an exception!
new BuiltList<int>([1, 2, 3]); // Better.
```
* 值必须为非空; 空无法添加到Built Collection。

```dart
new BuiltList([1, 2, null]);  // Throws an exception!
```
* Built Collections是可比的和可哈希的。这意味着您可以将它们放在maps，sets和multimaps中，创建与你的数据匹配的集合。

这些属性特别适合于业务对象的集合：例如，在购物Web应用程序中，用于保存用户购物车的当前内容。

___

这是对build_collection提供的内容和原因的快速概述; [github页面](https://github.com/google/built_collection.dart)上提供了更多信息。但是，为了充分利用不可变集合，您需要不可变类。接下来：[built_value](http://github.com/google/built_value.dart)！