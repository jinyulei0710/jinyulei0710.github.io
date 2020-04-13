---
layout: post
title:  "(译)状态管理中的声明式编程思维"
date:   2019-04-16 16:20:17 +0800
categories: 
   - Flutter 开发
tags:
    - 状态管理
---
[原文链接](https://flutter.dev/docs/development/data-and-backend/state-mgmt/declarative)

如果你是从一个命令式的框架（例如 Android SDK 或是 iOS UIKit ）来到 Flutter 的，你需要从一个新的角度开始思考应用开发。

<!--more-->


许多你可能有的假定在 Flutter 上不再适用。例如，在 Flutter 中从头开始重新构建你的部分 UI 而不去修改它们是没有问题的。Flutter 足够快以至于能这么做，如果需要的话甚至可以在每一帧上。

Flutter 是声明式的。这就意味着 Flutter 构建它的用户界面对你应用的当前状态进行响应：

![img](https://flutter.cn/assets/development/data-and-backend/state-mgmt/ui-equals-function-of-state-54b01b000694caf9da439bd3f774ef22b00e92a62d3b2ade4f2e95c8555b8ca7.png)

当你的应用的状态发生变化(例如，用户点了下设置界面的一个开关按钮)，你改变了状态，并且状态触发了用户界面的重新绘制。不存在命令式地改变UI本身的方式(比如 widget.setText )-你改变了状态，然后 UI 就从头开始构建。

想要知道更多关于响应式 UI 编程，就看下这篇[使用指南](Introduction-to-declarative-UI.html)。

响应式的 UI 编程有许多优点。最显著的优点就是，对于UI的所有状态都只有一条代码路径。你对于所有状态下 UI 应该长什么样描述一次就够了。


起初，这种编程风格可能看起来不像命令式风格那么直观。这就是本文存在的原因。继续看下去。