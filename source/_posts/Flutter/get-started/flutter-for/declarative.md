---
layout: post
title:  "(译)声明式 UI 介绍"
date:   2019-04-16 16:29:17 +0800
categories: 
   - Flutter 开发
tags:
    - declarative
---
[原文链接](https://flutter.dev/docs/get-started/flutter-for/declarative)

这篇介绍描述了 Flutter 使用的声明样式与许多其他 UI 框架使用的命令式样式之间的概念差异。

<!--more-->


## 为什么是声明式 UI?

从 Win32 到 Web 到 Android 和 iOS 的框架通常使用命令式的 UI 编程风格。这可能是您最熟悉的样式 - 您手动构建全功能UI实体（如UIView或等效实体），然后在 UI 更改时使用方法和 setter 对其进行变更。

为了减轻开发人员的负担，无需编写如何在不同的 UI 状态之间进行切换的代码，相比之下，Flutter 让开发人员描述当前的 UI 状态，并把状态切换的的工作留给了框架。

然而，这需要在思考如何操纵UI方面略有转变。

## 如何在声明式框架中改变 UI

思考以下一个简单的例子：

![img](https://flutter.cn/images/declarativeUIchanges.png)


在命令式风格中，你通常会转到 ViewB 的所有者并使用选择器或 findViewById 或类似函数获得实例 b，并在其上调用修改（并隐式对其刷新）。例如：

```dart
// Imperative style
b.setColor(red)
b.clearChildren()
ViewC c3 = new ViewC(...)
b.add(c3)
```

你可能还需要在 ViewB 的构造函数中复制此配置，因为 UI 的数据源可能比实例 b 本身活的更长。

在声明式样式中，视图配置（例如Flutter的小部件）是不可变的，并且只是轻量级的“蓝图”。要更改 UI，Widget 会在自身上触发重建（最常见的是通过在 Flutter 中的 StatefulWidget 上调用 setState() ）并构造一个新的 Widget 子树。

```dart
// Declarative style
return ViewB(
  color: red,
  child: ViewC(...),
)
```

这里，Flutter 构建新的 Widget 实例，而不是在 UI 更改时改变旧实例 b。该框架使用RenderObject 管理传统 UI 对象的许多职责（例如维护布局的状态）。 RenderObject 在帧之间保持不变，Flutter 的轻量级 Widget 告诉框架在状态之间改变 RenderObject。 Flutter框架处理其余部分。