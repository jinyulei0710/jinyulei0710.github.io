---
layout: post
title:  "(译)状态管理参考"
date:   2019-05-30 16:43:17 +0800
categories: 
   - Flutter 开发
tags:
   - 状态管理  
---
[原文链接](https://flutter.dev/docs/development/data-and-backend/state-mgmt/options)

> 译者序：官方关于状态管理，也就是架构的一份参考,并添加了一些前后相关的文章。从简单的 setState,到稍微复杂的 InheritedWidget 和 Scoped model，到足以应对大型项目的 Redux 和 Bloc，都给出了来自社区的技术分享博客。其中带有灰色背景的为中文翻译版本，没有灰色背景的为视频、github项目地址或者尚未翻译的博客。

状态管理是一个复杂的话题。 如果你认为某些问题没有得到解答，或者这些页面上描述的方法对你的用例不可行，那么你可能是对的。

通过以下链接了解更多信息，其中许多链接由 Flutter 社区提供：

<!--more-->


## 总体概述

[使用 Flutter 构建响应式应用](https://www.youtube.com/watch?v=RS36gBEp8OI&feature=youtu.be),一个来自 Google I/O 2018的视频，以及[一篇随附的文章](https://medium.com/flutter-io/build-reactive-mobile-apps-in-flutter-companion-article-13950959e381)。
[Flutter架构样例](http://fluttersamples.com/),作者 Brian Egan。

## setState

* [`添加交互性到你的 Flutter 应用中去`](/2019/04/18/Flutter/development/ui/interactive/)，一篇Flutter教程
* [`Google Flutter 中的基础状态管理`](/2019/04/18/Flutter/development/data-and-backend/state-mgmt/options/setState/Basic%20state%20management%20in%20Google%20Flutter/)，作者 Agung Surya


## InheritedWidget 和 Scoped model

* [`高效地使用 Flutter 的 InheritedWidget`](/2019/04/18/Flutter/development/data-and-backend/state-mgmt/options/InheritedWidget-Scopedmodel/Using%20Flutter%20Inherited%20Widgets%20Effectively/)，作者 Eric windmill
* [`你可能不需要 Redux：Flutter 版本`](/2019/04/18/Flutter/development/data-and-backend/state-mgmt/options/InheritedWidget-Scopedmodel/YouMightNotNeedReduxThe%20FlutterEdition/)，作者 Ryan Edge
* [在 Dart 的 Flutter 框架中使用 scoped model 模式管理状态](https://www.youtube.com/watch?v=-MCeWP3rgI0),一个来自 Tensor Programming 的视频
* [Flutter：InheritedWidget 和 Scoped model 解析，第一部分](https://www.youtube.com/watch?v=j-27MZwRbFw)，一个由 MtechViral 制作的视频
* [Flutter 状态管理-scoped model](https://www.youtube.com/watch?v=Oql5bU-Uvso)
* [Scoped model 包](https://pub.dartlang.org/packages/scoped_model)
* [`Widget-State-Context-InheritedWidget`](/2019/04/12/Flutter/development/data-and-backend/state-mgmt/options/InheritedWidget-Scopedmodel/Widget-State-BuildContext-InheritedWidget/),作者 Didier Bolelens

## Redux
* [使用 Redux和Flutter进行动画管理](https://www.youtube.com/watch?v=9ZkLtr0Fbgk),一个来自 DartConf 2018 的视频,[Medium上配套文章](https://medium.com/flutter/animation-management-with-flutter-and-flux-redux-94729e6585fa)。
* [发布网站](https://pub.dev/)上的[ Flutter Redux 包]()
* [`Flutter 中的 Redux 介绍`](/2019/04/20/Flutter/development/data-and-backend/state-mgmt/options/Redux/introduction%20to%20Redux%20in%20Flutter/),作者 Xavi Rigau
* [`Flutter + Redux-如何做出一个购物清单应用`](/2019/04/21/Flutter/development/data-and-backend/state-mgmt/options/Redux/Flutter%20+%20Redux%E2%80%8A%E2%80%94%E2%80%8AHow%20to%20make%20Shopping%20List%20App/)，一篇Paulina Szklarska在 Hackernoon上的文章。
* [在 Flutter 中用 Redux 构建一个TODO应用(CRUD)-第一部分](https://www.youtube.com/watch?v=Wj216eSBBWs)，一个来自 Tensor Programming 的视频
* [Flutter Redux Thunk的一个例子](https://medium.com/flutterpub/flutter-redux-thunk-27c2f2b80a3b),作者 Jack Wong
* [使用 Redux 构建一个大型应用](https://hillel.dev/2018/06/01/building-a-large-flutter-app-with-redux/),作者 Hillel Coren
* [基于 Redux 数据管理的组装式 flutter 应用框架](https://github.com/alibaba/fish-redux/),作者 Alibaba闲鱼团队

## BLoc/Rx

* [`使用 BLoc 模式架构你的 Flutter 应用`](/2019/04/15/Flutter/development/data-and-backend/state-mgmt/options/BloC/Architect%20your%20Flutter%20project%20using%20BLOC%20pattern/),作者Sagar Suri
* [`使用 BLoc 模式架构你的 Flutter 应用第二部分`](/2019/04/15/Flutter/development/data-and-backend/state-mgmt/options/BloC/Architect%20your%20Flutter%20project%20using%20BLOC%20pattern/),作者Sagar Suri
* [Bloc 库](https://felangel.github.io/bloc),作者Felix Angelov
* [`Flutter中的响应式编程(Reactive Programming)、流(Streams)以及业务逻辑组件(BloC)`](/2019/04/13/Flutter/development/data-and-backend/state-mgmt/options/BloC/ReactiveProgramming-Streams-BLoC/)，作者Didier Boelens
* [`响应式编程-流-Bloc-实例`](/2019/04/13/Flutter/development/data-and-backend/state-mgmt/options/BloC/ReactiveProgramming-Streams-BLoC-PracticalUseCases/)，作者Didier Boelens




