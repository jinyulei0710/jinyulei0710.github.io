---
layout: post
title:  "(译)Dart中适用于序列化的built_value"
date:   2019-04-15 14:20:17 +0800
categories: 
   - Flutter 开发
tags:
    - JSON序列化
    - built_value
---
[原文链接](https://medium.com/dartlang/darts-built-value-for-serialization-f5db9d0f4159)

上周，我介绍了[适用于不可修改对象模型的built_value](built_value-for-Immutable-Object-Models.html)。我们看到了如何在built_value中定义对象模型;它们是不变的，易于使用，如果你喜欢那种东西，那就很有趣了。

本文介绍了built_value包的其余部分。最重要的是，正如你可能从标题中猜到的那样，它们也是可序列化的。

<!--more-->


以下是built_value序列化的用法：

```dart
// Value type defined using built_value.
abstract class Login implements Built<Login, LoginBuilder> {
  // Add serialization support by defining this static getter.
  static Serializer<Login> get serializer => _$loginSerializer;
  ...
}
// Once per app, define a top level "Serializer" to gather together
// all the generated serializers.
Serializers serializers = _$serializers;
// Use it!
var login = new Login((b) => b
  ..username = 'johnsmith'
  ..password = '123456');
print(JSON.encode(serializers.serialize(login)));
-->
["Login", "username", "johnsmith", "password", "123456"]
```

注意到“JSON.encode”了吗？序列化程序实际上并不像序列化到String那样;而是它转换为Dart的内置JSON序列化知道如何处理的基础类型。因此，如果您愿意，可以使用JSON之外的其他东西。

您可能认为序列化是很简单的东西，但是这里涉及一些微妙的权衡。让我们深入研究built_value的序列化。

## 多态

built_value序列化的最重要的一个方面是它支持多态。具体来说，您可以拥有抽象类型的字段，以及

* 该抽象类型的任何可序列化实现都可以序列化;
* 将在线上写入足够的信息以反序列化为正确的类型。

最简单的例子是它可以序列化Object列表：

```
serializers.serialize(new BuiltList<Object>([1, 'two', 3]));
-->
['list', ['int', 1, 'string', 'two', 'int', 3]]
```


仅在需要消除反序列化时消除歧义时，才会在线路上添加额外信息。因此，如果你有一个类型为“BuiltList <int>”的字段，它将被序列化为“[1,2,3]”，而不是像“['int'，1，'int'，2，'int'， 3]”。

最重要的是，您可以根据需要定义对象模型，而built_value将序列化它。如果您想更详细地看到这一点，[map_serializer_test](https://github.com/google/built_value.dart/blob/master/built_value/test/built_map_serializer_test.dart)探索所有可能性。

## 多个实现
所有序列化机制必须面对的另一个问题是以某种方式定义可序列化类型的范围。在这里，通过允许一个“类型”的多个实现，built_json做了一些不寻常的事情。

这是有效的，因为类型仅通过其类名在线上定义。例如，没有尝试消除称为“登录”的不同类之间的歧义;假设发送方和接收方都有一个兼容的序列化程序，可用于名为“Login”的类。

这增加了有用的灵活性例如，如果您在服务器和客户端上使用Dart，则可以选择对象模型中的每个类：

* 您可以在客户端和服务器上使用相同的类。
* 或者，您可以使用不同的类。实现必须具有相同的名称和兼容的字段。

例如，您可以为客户端提供一个“登录”类来处理渲染和解析;以及处理身份验证和数据库的服务器的单独“登录”类。当然，仅服务器实现可以自由使用像“dart：io”这样的包，仅客户端可以使用的像“”dart:html"这样的包。

## 多种语言

因为built_value序列化仅通过类名识别类型，所以序列化数据很好地映射到任何面向对象的语言。计划通过[AutoValue](https://github.com/google/auto/tree/master/value)支持Java。

## 多种版本

序列化的built_value数据以非常简单的方式向后/向前兼容：它依赖于类名和字段名。类名更改和必填字段名称更改正在中断。

可空字段更灵活：在序列化时，只有非空时才会写入;在反序列化时，如果找不到它们，它们将默认为null。因此，可以添加，删除或重命名可空字段，这不是一个重大变化。

无法识别的字段将被忽略。

## 没有反射
最后，对性能至关重要的是，built_value不使用任何形式的反射。所有分析都在代码生成时完成，为你提供最小化的，高性能的序列化代码。

那是使用built_value完成的序列化。你可以坐下来编写一个对象模型，它可以直接序列化以用于RPC或长期存储。

## 枚举类

最后，built_value还有一个功能：EnumClass。 Dart枚举不是类，但强大的对象模型需要行为类似于类的枚举。显而易见的模式是创建一个带有“static const”字段的类，而EnumClass使这更容易做到。它提供了：

* 适用于“values”和“valueOf”的生成代码。
* 通过built_value序列化程序进行序列化。
* Angular或Angular2用户的额外奖励：codegen可以选择生成mixin来帮助您使用模板中的枚举。
在[示例](https://github.com/google/built_value.dart/blob/master/example/lib/enums.dart)中可以看到所有这些特性。

这就是本周的内容！已经涵盖了built_value的基础知识，我准备在下周详细介绍聊天示例。敬请关注！

