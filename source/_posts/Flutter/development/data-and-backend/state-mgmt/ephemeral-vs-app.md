---
layout: post
title:  "(译)短时状态和共享状态的区别"
date:   2019-04-16 16:36:17 +0800
categories: 
   - Flutter 开发
tags:
    - 状态管理
---

[原文链接](https://flutter.dev/docs/development/data-and-backend/state-mgmt/ephemeral-vs-app)

本文档介绍了应用状态，短时状态以及如何在 Flutter 应用中对这两种状态进行管理。

<!--more-->


在广义上，应用程序的状态是应用程序运行时在内存中存在的所有内容。这包括应用程序的`assets`，Flutter框架维护的关于UI，动画状态，纹理，字体等等的所有变量。虽然这种最广泛的状态定义是有效的，但它对于构建应用程序并不是很有用。

首先，你甚至不管理某些状态（如纹理）。 框架会为你处理这些问题。因此，更有用的状态定义是“在任何时刻重建UI所需的任何数据”。其次，你自己管理的状态可以分为两种概念类型：短时状态和应用状态。

## 短时状态

短暂状态（有时称为 UI 状态或本地状态）是您可以整齐地包含在单个 widget 中的状态。

这是一个模糊的定义，所以这里有一些例子。

* `pageView` 的当前页
*  复杂动画的当前状态
* `BottomNavigatorBar` 选中的当前标签

widget 树的其他部分很少需要存取这种状态。 没有必要对其进行序列化，并且它不会以复杂的方式发生变化。

换句话说，不需要在这种状态下使用状态管理技术（ScopedModel，Redux 等。你只需要一个StatefulWidget。

以下，你将看到底部导航栏中当前所选项目如何保存在 _MyHomepageState 类的 \_index 字段中。 在此例中，_index 是短时状态。

```dart
class MyHomepage extends StatefulWidget {
  @override
  _MyHomepageState createState() => _MyHomepageState();
}

class _MyHomepageState extends State<MyHomepage> {
  int _index = 0;

  @override
  Widget build(BuildContext context) {
    return BottomNavigationBar(
      currentIndex: _index,
      onTap: (newIndex) {
        setState(() {
          _index = newIndex;
        });
      },
      // ... items ...
    );
  }
}
```

在这里，使用 setState() 和 StatefulWidget 的 State 类中的字段是完全自然的。 你的应用没有其他任何部分需要存取 _index。 该变量仅在 MyHomepage widget 内更改。 而且，如果用户关闭并重新启动应用程序，你不介意 \_index 重置为零。

## 共享状态

那些不是短时的状态，你希望在应用程序的许多部分之间共享，并且你希望在用户会话之间保持 - 这就是我们所说的应用程序状态（有时也被称为共享状态）。

应用程序状态的示例：

* 用户偏好
* 登录信息
* 社交网络应用中的通知
* 电子商务应用中的购物车
* 新闻应用中的文章的已读/未读状态

对于管理应用状态，你会对你手里的选项做一番研究。您的选择取决于你的应用程序的复杂性和特性，团队以前的经验以及许多其他方面。继续看。

## 没有明确的规则

要清楚，你可以使用 State 和 setState() 来管理应用中的所有状态。 实际上，Flutter 团队在许多简单的应用程序示例中都会这样做（包括每次创建时都会获得的入门应用程序)。

它也有另一种方式。 例如，你可能决定 - 在特定应用程序的上下文中 - 底部导航栏中的选定选项卡不是短时状态。 你可能需要从类外部更改它，在会话之间保留它，依此类推。 在这种情况下，_index 变量是 app 状态。

没有明确的通用规则来区分特定变量是短时状态还是应用状态。 有时，你必须将一个重构为另一个。 例如，你会从一些清晰的短时状态开始，但随着你的应用程序功能的增加，它将需要变动到应用程序状态。

因此，让我们以下示意图持保留态度：

![img](https://flutter.cn/assets/development/data-and-backend/state-mgmt/ephemeral-vs-app-state-3137024aa509b4df5d20ed7ed30fb8a0f7cff54ebc8ab0d6e39794bced87e27c.png)

当被问及 React 的 setState 与 Redux 的 store 时，Redux 的作者 Dan Abramov 回答说：

经验法则是：哪个方便用哪个

总之，任何 Flutter 应用程序中都有两种概念类型的状态。 短时状态可以使用 State 和setState() 来实现，并且通常是 widget 本地状态。 剩下的就是你的应用状态。 这两种类型在任何 Flutter 应用程序中都占有一席之地，两者之间的分割取决于你自己的偏好和应用程序的复杂性。

