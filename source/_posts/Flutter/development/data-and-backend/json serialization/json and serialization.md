---
layout: post
title:  "(译) Flutter 中的 JSON 序列化"
date:   2019-04-15 10:40:17 +0800
categories: 
   - Flutter 开发
tags:
    - JSON序列化
---

[原文链接](https://flutter.dev/docs/development/data-and-backend/json)

很难想象，一个移动应用程序不需要与Web服务器通信，或者只是简单地在某一个位置存储结构化数据。制作与网络连接的应用程序的时候，迟早会需要使用一些美好的JSON。

本指南介绍了如何在Flutter中使用JSON。它涵盖了在不同场景中使用哪种JSON解决方案，以及原因。

<!--more-->

> 术语：编码和序列化是一回事 - 将数据结构转换为字符串。解码和反序列化是相反的过程 - 将字符串转换为数据结构。但是，序列化通常也指将数据结构转换为更易于阅读的格式的整个过程。

为避免混淆，本文档在引用整个过程时使用“序列化”，在专门引用这些过程时使用“编码”和“解码”。

## 哪种JSON序列化方法适合我？
本文介绍了使用JSON的两种常规策略：

* 手动序列化
* 使用代码生成进行自动序列化
不同的项目具有不同的复杂性和用例。对于较小的概念验证项目或快速原型，使用代码生成器可能有点杀鸡用牛刀。对于具有更多复杂性的多个JSON模型的应用程序，手动编码很快就会变得乏味，重复，并且会带来许多小错误。

## 对较小的项目使用手动序列化
手动JSON解码使用dart：convert中内置的JSON解码器。它涉及将原始JSON字符串传递给jsonDecode（）方法，然后在生成的Map <String，dynamic>中查找所需的值。它没有外部依赖或特别的设置过程，它有利于快速验证概念。

当项目变大时，手动解码的效果就变得不佳。手动编写解码逻辑可能变得难以管理且容易出错。如果在访问不存在的JSON字段时出现拼写错误，则代码会在运行时抛出错误。

如果您的项目中没有很多JSON模型，并且希望快速测试概念，那么手动序列化可能就是您想要的方式。有关手动编码的示例，请参阅使用dart：convert手动序列化JSON。

## 在大中型项目中使用代码生成

使用代码生成的JSON序列化意味着有一个外部库为您生成编码样板。进行一些初始设置后，您将运行一个文件观察者，它会依据您的模型类生成代码。例如，[json_serializable](https://pub.dartlang.org/packages/json_serializable)和[built_value](https://pub.dartlang.org/packages/built_value)就是这种类型的库。

这种方法适用于较大的项目。不需要手写的样板文件，并且在编译时捕获访问JSON字段时的拼写错误。代码生成的缺点是它需要一些初始设置。此外，生成的源文件可能会在项目导航器中产生视觉混乱。

当您拥有中型或大型项目时，您可能希望使用生成的代码进行JSON序列化。要查看基于JSON编码的代码生成示例，请参阅[使用代码生成库序列化JSON](https://flutter.dev/docs/development/data-and-backend/json#code-generation)。

## Flutter中是否有GSON / Jackson / Moshi等价物？

答案很简单，就是没有。

这样的库需要使用运行时反射，这在Flutter中被禁用。运行时反射会干扰[tree sharking](https://en.wikipedia.org/wiki/Tree_shaking)机制，Dart支持了这个机制已经很长时间。有了tree sharking](https://en.wikipedia.org/wiki/Tree_shaking)机制，您可以从发布版本中“摇掉”未使用的代码。这对应用大小的优化是显著的。

由于反射使得代码所有代码在默认情况下是隐式使用的，因此使[tree sharking](https://en.wikipedia.org/wiki/Tree_shaking)机制变得困难。这些工具无法知道运行时哪些部分未使用，因此冗余代码很难被删除。使用反射时，无法轻松优化应用程序大小。

dartson怎么样？ dartson库使用运行时反射，这使它与Flutter不兼容。

虽然您不能在Flutter中使用运行时反射，但是某些库基于代码为你提供了类似的易用API，[代码生成库部分](https://flutter.dev/docs/development/data-and-backend/json#code-generation)更详细地介绍了此方法。

## 使用dart：convert手动序列化JSON

Flutter中的基本JSON编码非常简单。 Flutter有一个内置的dart：转换库，包括一个简单的JSON编码器和解码器。

以下是简单用户模型的示例JSON。

```json
{
  "name": "John Smith",
  "email": "john@example.com"
}
```
使用dart：convert，您可以通过两种方式对此JSON模型进行编码。

### 以内联的方式序列化JSON

通过查看dart：convert文档，您将看到可以通过调用jsonDecode()方法来解码JSON，并使用JSON字符串作为方法参数。

```dart
Map<String, dynamic> user = jsonDecode(jsonString);

print('Howdy, ${user['name']}!');
print('We sent the verification link to ${user['email']}.');
```
不幸的是，jsonDecode（）返回一个Map <String，dynamic>，这意味着在运行时之前你不知道值的类型。使用这种方法，您将丧失大部分静态类型语言特性：类型安全，自动补全以及最重要的编译时异常。您的代码将立即变得更容易出错。

例如，当年使用名称或电子邮件字段的时候，都可能会很快引入拼写错误。由于JSON存在于map结构中，编译器对于拼写错误是不知道。

### 在模型类中序列化JSON

为了解决以上提及到的问题，我们引入了一个普通的模型类，在本例中叫做User。在User类中，您将找到：
* User.fromJson（）构造函数，用于从map结构构造新的User实例。
* 一个toJson（）方法，它将User实例转换为map。
使用这种方法，调用代码可以具有类型安全性，名称和电子邮件字段的自动补全以及编译时异常。如果您输入拼写错误或将字段视为int而不是字符串，则应用程序将无法编译，而不是在运行时崩溃。

user.dart

```dart
class User {
  final String name;
  final String email;

  User(this.name, this.email);

  User.fromJson(Map<String, dynamic> json)
      : name = json['name'],
        email = json['email'];

  Map<String, dynamic> toJson() =>
    {
      'name': name,
      'email': email,
    };
}
```

解码逻辑的责任现在在模型本身内部移动。使用这种新方法，您可以轻松解码User。

```dart
Map userMap = jsonDecode(jsonString);
var user = new User.fromJson(userMap);

print('Howdy, ${user.name}!');
print('We sent the verification link to ${user.email}.');
```

要对用户进行编码，请将User对象传递给jsonEncode（）函数。你不需要调用toJson（）方法，因为jsonEncode（）已经为你做了。

```dart
String json = jsonEncode(user);
```

使用这种方法，调用代码根本不必担心JSON序列化。但是，模型类仍然是必须。在生产应用程序中，您需要确保序列化正常工作。实际上，User.fromJson()和User.toJson()方法都需要进行单元测试以验证其行为的正确性。

但是，现实世界的场景通常不会那么简单。您不太可能使用如此小的JSON响应体。嵌套的JSON对象也是常有的。

如果有一些东西可以为你处理JSON编码和解码，那就太好了。幸运的是，有！

## 使用代码生成库序列化JSON
json_serializable包，这是一个自动生成的源代码生成器，可为您生成JSON序列化样板。

由于序列化代码不再是手动或手动维护的，因此可以最大限度地降低在运行时出现JSON序列化异常的风险。

## 在项目中设置json_serializable
要在项目中包含json_serializable，您需要一个常规依赖项和两个dev依赖项。简而言之，dev依赖项是我们的应用程序源代码中未包含的依赖项 - 它们仅在开发环境中使用。

可以通过以下JSON可序列化示例中的pubspec文件来查看这些必需依赖项的最新版本。

pubspec.yaml

```yaml
dependencies:
  # Your other regular dependencies here
  json_annotation: ^2.0.0

dev_dependencies:
  # Your other dev_dependencies here
  build_runner: ^1.0.0
  json_serializable: ^2.0.0
```

进入项目根文件夹运行flutter包，（或单击编辑器中的Packages Get）以在项目中使用这些新的依赖项。

## 以json_serializable方式创建模型类
下面显示了如何将User类转换为json_serializable类。为简单起见，此代码使用先前示例中的简化JSON模型。

user.dart

```dart
import 'package:json_annotation/json_annotation.dart';

/// This allows the `User` class to access private members in
/// the generated file. The value for this is *.g.dart, where
/// the star denotes the source file name.
part 'user.g.dart';

/// An annotation for the code generator to know that this class needs the
/// JSON serialization logic to be generated.
@JsonSerializable()

class User {
  User(this.name, this.email);

  String name;
  String email;

  /// A necessary factory constructor for creating a new User instance
  /// from a map. Pass the map to the generated `_$UserFromJson()` constructor.
  /// The constructor is named after the source class, in this case User.
  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);

  /// `toJson` is the convention for a class to declare support for serialization
  /// to JSON. The implementation simply calls the private, generated
  /// helper method `_$UserToJson`.
  Map<String, dynamic> toJson() => _$UserToJson(this);
}
```

通过此设置，源代码生成器生成用于编码和解码来自JSON的名称和电子邮件字段的代码。

如果需要，还可以轻松自定义命名策略。例如，如果API返回带有snake_case的对象，并且您希望在模型中使用lowerCamelCase，则可以将@JsonKey批注与name参数一起使用：

```dart
/// Tell json_serializable that "registration_date_millis" should be
/// mapped to this property.
@JsonKey(name: 'registration_date_millis')
final int registrationDateMillis;
```

## 运行代码生成实用程序

在第一次创建json_serializable类时，您将得到类似于下图所示的错误。

![img](https://flutter.dev/images/json/ide_warning.png)

这些错误完全正常，仅仅是因为模型类的生成代码尚不存在。要解决此问题，请运行生成序列化样板的代码生成器。

有两种运行代码生成器的方法。

### 一次性代码生成
通过运行flutter包，在项目根目录中运行build_runner build，可以在需要时为模型生成JSON序列化代码。这会触发一次性构建，该构建遍历源文件，选择相关文件，并为它们生成必要的序列化代码。

尽管这是很方便的，每次当你对你的模型类做了修改的时候，你没必要每次都去手动运行一次。

### 持续性地生成代码
观察者使我们的源代码生成过程更加方便。它会监听项目文件中的更改，并在需要时自动构建必要的文件。通过在项目根目录中运行flutter packages pub run build_runner watch来启动观察程序。

启动观察器一次，然后把它扔在后台运行是安全的。

### 使用json_serializable模型
要以json_serializable方式解码JSON字符串，您实际上没有对我们以前的代码进行任何更改。

```dart
Map userMap = jsonDecode(jsonString);
var user = User.fromJson(userMap);
```
编码也是如此。调用的API与以前相同。

使用json_serializable，您可以忘记User类中的任何手动JSON序列化。源代码生成器创建一个名为user.g.dart的文件，该文件具有所有必需的序列化逻辑。您不再需要编写自动化测试来确保序列化工作 - 现在确保序列化正常工作的职责落到了库上。
