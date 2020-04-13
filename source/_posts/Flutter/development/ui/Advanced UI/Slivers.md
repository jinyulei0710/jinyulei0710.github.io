---
layout: post
title:  "(译) Slivers 揭秘"
date:   2019-05-07 12:30:17 +0800
categories: 
   - Flutter 开发
tags:
    - Slivers
---

如何在你的应用中使用Flutter去做出你想要的滑动效果。

你好，无畏的 Flutter 爱好者！今天我们要探索一个高度专业的话题-这是大部分应用开发者可以无忧无虑忽略的并且在无需知晓的情况下就能做出好看的应用。通常来说，你需要通过东西来进行滚动的话，[ListView](https://docs.flutter.io/flutter/widgets/ListView-class.html) 和 [GridView](https://docs.flutter.io/flutter/widgets/GridView-class.html) 就能胜任这份工作。这就完了。但是，如果你想要寻求更深入的知识并想增强你的滚动能力：

那就继续读下去

<!--more-->


或者等一下...如果你讨厌阅读，你可以快速的看下这两个视频，对本文的思想进行了总结，

<iframe width="700" height="393" src="https://www.youtube.com/embed/R9C5KMJKluE" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

<iframe width="700" height="393" src="https://www.youtube.com/embed/ORiTTaVY6mM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## 什么是 Sliver ,为什么要使用它？

在网上我可以看到大量的 FOS 。那些因为不了解Slivers而害怕 Slivers 的人。但是Sliver 只是可滑动区域的一部分。仅此而已！揭开面纱的话，所有你使用的可以滚动的视图，像 ListView 和 GridView ,实际上是使用 Slivers 实现的。你可以把 Sliver 当成较为底层的接口，在实现可滚动区域方面提供更加细腻化的控制。因为 Slivers 可以在每个条目被滚动到的时候对其进行懒构建，

slivers尤其对包含大量子视图的情形的高效滚动有用。

在滚动的时候你可能要以下的额外控制：

* 想要带有一个非标准行为的 app bar（在你滚动的时候），
* 想要列表和表格作为一个整体一起滚动（当然，你可以在listView中放一个表格但是效率上就差很多了，特别是当你有一个很大的表格的时候。）
* 做些怪异的事情，比如带有headers的可折叠列表（看下这页顶部右侧的gif图片）

## 我该如何使用它？

所有这些Sliver组件都是在[CustomScrollView]()中使用的，剩下的就是让你来决定如何组合你的sliver列表来组成你的自定义的可滚动区域。你可以通过把一个SliverList放入CustomScrollView然后不用做其他事情就可以完成对ListView的彻底改造。

## SliverList

SliverList把deledate作为参数，这个参数提供了在视图中滚动的列表中的条目。你可以用SliverChildListDelegate来指定实际的孩子列表，或者使用SliverChildBuilderDelegate对它们进行懒构建。

```dart
// 孩子的显式列表. 没有性能的节省
// 因为孩子已经构建完成了。
SliverList(
    delegate: SliverChildListDelegate(
      [
        Container(color: Colors.red, height: 150.0),
        Container(color: Colors.purple, height: 150.0),
        Container(color: Colors.green, height: 150.0),
      ],
    ),
);
// 无限滚动的不同颜色容器的列表
SliverList(
    delegate: SliverChildBuilderDelegate((BuildContext context, int index) {
      // 要把这个无限的列表转换为三个条目的列表
      // 对下面这行取消注释
      // if (index > 3) return null;
      return Container(color: getRandomColor(), height: 150.0);
    },
    // 或者对下面这行取消注释
    // childCount: 3,
  ),
);
```
## SliverGrid

SliverGrid就像SliverList也可以用deledate指定孩子。但是还有一个网格上cross-axis dimension的一些额外格式化。有三种方式能让你对你的表格布局进行指定：

1. Count 构造器来数有多少个条目在，在此例中，在横轴上：SliverGrid.count(children:scrollItem,crossAxisCount:4)
2. 对Constructor进行扩展指定条目宽度来适应网格。这在你的网格中大小不是固定的时候特别有用，你可以限制它们占据多大的空间（在此例中，是在水平方向上限制）。SliverGrid.extent(children: scrollItems, maxCrossAxisExtent: 90.0) // 90 logical pixels
3. 默认构造器，传入一个明确的gridDelegate参数：

```dart
// Re-implementing the above SliverGrid.count example:
SliverGrid(
  gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 4,
  ),
  delegate: SliverChildBuilderDelegate(
    (BuildContext context, int index) {
      return new Container(
        color: randomColor(),
        height: 150.0);
    }
);
```
## SliverAppBar

好了，好了，已经说的够多了。我知道你在等待什么了。我该如何却做出这些华丽伸缩的app-bar呢？密码就是同时设置flexibleSpace参数和expandedHeight参数。
当应用栏扩展到完整大小而不是“压缩”版本时，您可以为应用栏设置不同的高度和外观。

这是以上例子的代码：

```dart
CustomScrollView(
    slivers: <Widget>[
      SliverAppBar(
        title: Text('SliverAppBar'),
        backgroundColor: Colors.green,
        expandedHeight: 200.0,
        flexibleSpace: FlexibleSpaceBar(
          background: Image.asset('assets/forest.jpg', fit: BoxFit.cover),
        ),
      ),
      SliverFixedExtentList(
        itemExtent: 150.0,
        delegate: SliverChildListDelegate(
          [
            Container(color: Colors.red),
            Container(color: Colors.purple),
            Container(color: Colors.green),
            Container(color: Colors.orange),
            Container(color: Colors.yellow),
            Container(color: Colors.pink),
          ],
        ),
      ),
    ],
);
```

有一些额外的你可以加到 SliverAppBar 的定制。你可以把 floating 参数设置成为true ，使app bar在你往下滚动的时候再次出现，即使你还没有达到列表的顶部。

如果同时使用浮动参数添加 snap 参数，则可以在向下滚动时使应用栏完全捕捉回视图。

## 把所有这些放到一起：带有Sliver固定标题的可折叠滚动列表 

我试着想象一下我能想到的最不寻常但仍然可能有用的滚动行为。我想出了这个滚动的可折叠列表：

```dart
import 'package:flutter/material.dart';
import 'dart:math' as math;
void main() => runApp(MyApp());
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(title: Text('Collapsing List Demo')),
        body: CollapsingList(),
      ),
    );
  }
}
class _SliverAppBarDelegate extends SliverPersistentHeaderDelegate {
  _SliverAppBarDelegate({
    @required this.minHeight,
    @required this.maxHeight,
    @required this.child,
  });
  final double minHeight;
  final double maxHeight;
  final Widget child;
  @override
  double get minExtent => minHeight;
  @override
  double get maxExtent => math.max(maxHeight, minHeight);
  @override
  Widget build(
      BuildContext context, 
      double shrinkOffset, 
      bool overlapsContent) 
  {
    return new SizedBox.expand(child: child);
  }
  @override
  bool shouldRebuild(_SliverAppBarDelegate oldDelegate) {
    return maxHeight != oldDelegate.maxHeight ||
        minHeight != oldDelegate.minHeight ||
        child != oldDelegate.child;
  }
}
class CollapsingList extends StatelessWidget {
  SliverPersistentHeader makeHeader(String headerText) {
    return SliverPersistentHeader(
      pinned: true,
      delegate: _SliverAppBarDelegate(
        minHeight: 60.0,
        maxHeight: 200.0,
        child: Container(
            color: Colors.lightBlue, child: Center(child:
                Text(headerText))),
      ),
    );
  }
  @override
  Widget build(BuildContext context) {
    return CustomScrollView(
      slivers: <Widget>[
        makeHeader('Header Section 1'),
        SliverGrid.count(
          crossAxisCount: 3,
          children: [
            Container(color: Colors.red, height: 150.0),
            Container(color: Colors.purple, height: 150.0),
            Container(color: Colors.green, height: 150.0),
            Container(color: Colors.orange, height: 150.0),
            Container(color: Colors.yellow, height: 150.0),
            Container(color: Colors.pink, height: 150.0),
            Container(color: Colors.cyan, height: 150.0),
            Container(color: Colors.indigo, height: 150.0),
            Container(color: Colors.blue, height: 150.0),
          ],
        ),
        makeHeader('Header Section 2'),
        SliverFixedExtentList(
          itemExtent: 150.0,
          delegate: SliverChildListDelegate(
            [
              Container(color: Colors.red),
              Container(color: Colors.purple),
              Container(color: Colors.green),
              Container(color: Colors.orange),
              Container(color: Colors.yellow),
            ],
          ),
        ),
        makeHeader('Header Section 3'),
        SliverGrid(
          gridDelegate: 
              new SliverGridDelegateWithMaxCrossAxisExtent(
            maxCrossAxisExtent: 200.0,
            mainAxisSpacing: 10.0,
            crossAxisSpacing: 10.0,
            childAspectRatio: 4.0,
          ),
          delegate: new SliverChildBuilderDelegate(
            (BuildContext context, int index) {
              return new Container(
                alignment: Alignment.center,
                color: Colors.teal[100 * (index % 9)],
                child: new Text('grid item $index'),
              );
            },
            childCount: 20,
          ),
        ),
        makeHeader('Header Section 4'),
        // Yes, this could also be a SliverFixedExtentList. Writing 
        // this way just for an example of SliverList construction.
        SliverList(
          delegate: SliverChildListDelegate(
            [
              Container(color: Colors.pink, height: 150.0),
              Container(color: Colors.cyan, height: 150.0),
              Container(color: Colors.indigo, height: 150.0),
              Container(color: Colors.blue, height: 150.0),
            ],
          ),
        ),
      ],
    );
  }
}
```

最后一步留给读者，就是添加一个 GestureDetector, 那么当你点击标题的时候，就能允许你跳转到列表中的这部分。带着你新发现的关于 Slivers 的知识并把它运用到GestureDetection 上去来做出一个炫酷的可折叠列表吧。

