---
layout: post
title:  "(译)你可能不需要Redux：Flutter版"
date:   2019-04-18 14:33:17 +0800
categories: 
   - Flutter 开发
tags:
    - 状态管理
    - ScopedModel
---

![](https://cdn-images-1.medium.com/max/800/1*D1hjP93cVtw5v2x-4rpztw.png)

长话短说：SetState->[scoped_model](https://pub.dartlang.org/packages/scoped_model)->[redux](https://pub.dartlang.org/packages/redux).

如果你是从事移动开发相关工作的，你可能听过[Flutter](https://flutter.io/)这个小玩意。这是一个非常棒的移动UI框架，您无法想象应用程序开发可以犹如游戏开发，运行时随应用程序一起提供，可以在不放弃性能的情况下实现完全可定制的体验。

<!--more-->

当你研究如何使用Flutter构建应用的时候，你可能遇到`setState`和`redux`这两种最流行的管理应用状态的模式。虽然它们彼此都有一席之地，但是`setState`的简易性和`redux`的灵活性之间存在着鸿沟。

## setState有什么问题？

要清楚，使用setState没有任何问题。这种更新小部件状态的方法是Flutter和React最强大的功能之一。您可以在此处阅读有关如何使用自上而下的数据流有效管理React应用程序中的状态的信息。该模式同样适用于Flutter。

也就是说，如果你熟悉React，那么你就会遇到“螺旋钻”的痛苦。对于初学者来说，这是通过不关心数据的组件传递道具的过程，这样您就可以将它发送给那些关心的组件。重构组件时，问题被放大，使维护成为一场噩梦，并且您还可能通过为一个窗口小部件重新呈现窗口小部件树的部分而体验性能下降。

## Redux有什么问题?

与setState一样，redux没有任何问题。这是一个完美的模式，可以根据需要管理应用程序的状态。

也就是说，使用redux并非没有缺点，其主要原因是它可能没有必要时增加的复杂性。作为好公民，我们

不应该过度优化。


不应该使用模式或库，因为它们很受欢迎。

Redux，无论它对于前端开发有多么棒，也不例外。

##在状态管理上获取setState般的简易性

作为React的重要支持者，我非常喜欢社区中库日益增长的趋势，他们利用其新的Context API在状态管理中获得了一席之地。在搜索类似的模式时，我遇到了scoped_model。

如果您对在flutter应用程序中管理状态感兴趣，我强烈建议您使用Scoped Models。
请注意，目前这不是在flutter应用程序中管理状态的“常规”方式，但我们发现它非常有用。
就个人而言，我认为你应该尽可能少地使用StatefulWidgets。 Scoped Models非常有用。

就像[unstated](https://github.com/jamiebuilds/unstated)利用React和上下文中已存在的模式一样，scoped_model构建在您已经熟悉的模式上（如果使用setState）。


## 总览

scoped_model是基于三个主要的概念：

### 1.model

从概念上讲，Model与redux store 非常相似。它是一个类似于widget的类，只包含与状态相关的部分。

![](https://cdn-images-1.medium.com/max/800/1*R1Dw1D5iEf_XB1cILeQG6A.png)
扩展Model的CounterModel

### 2.ScopedModel

ScopedModel包含状态模型，将其提供给请求它的所有widget。这类似于unstated的Provider。
![](https://cdn-images-1.medium.com/max/800/1*TKhNRYBHTSE1VDJtqi0Hmw.png)
在应用顶层创建ScopedModel

### 3. ScopedModelDescendant

ScopedModelDescendant小部件允许您将状态从Model传递到窗口小部件。状态更改将触发重新呈现，您可以在模型上调用方法。这与unstated的Subscribe组件非常相似。

![](https://cdn-images-1.medium.com/max/800/1*gDBbNOWQ4-crhkySIFETxw.png)
ScopedModelDescendant接受传入模型的build()函数。

### 添加多个Container到widget

在查看API之后，我遇到的第一个问题是如何向widget添加多个模型？

使用mixins这是相当简单的。让我们重写前面的例子，增加递减和重置计数的能力：

![](https://cdn-images-1.medium.com/max/800/1*LZAdc8a3pl1wyL80-hyGrg.png)
然后，我们可以在应用程序的根下用MainModel替换我们的CounterModel创建
![](https://cdn-images-1.medium.com/max/800/1*oymne0NLUoZk8VFIPsBVsQ.png)
将CounterModel替换为MainModel

最后，我们可以使用多个模型完成我们的示例
![](https://cdn-images-1.medium.com/max/800/1*5B58dyaB_6jwDd19v_nsoQ.png)

build()方法可以访问所有MainModel道具


你可能在想

为什么他不把所有这些方法放在一起形成一个逻辑模型？
你的评估是正确的，这个例子分离没有多大意义。将此视为更多的能力演示。

实际上，您应该组合代表不同类型状态的模型（CounterState，AuthState，ProfileState等）。

## 异步操作


由于模型的简单性，异步操作变成了贪睡的盛会。一旦状态改变，触发notifyListeners基本上就像setState一样运行。

## 结论

在本文开头，我提到使用setState或redux进行状态管理没有任何问题，我想重申这一说法。本文更多的是关于填补两者之间空白而作出的努力。

类似于React中状态管理演变的unstated，scoped_model是Flutter中的演变。与unstated一样，它降低了复杂性并保持在Flutter的范围内。

在考虑状态管理时，scoped_model值得与setState和redux放在一起，特别是当你需要比setState更强大但又不需要太厚重的时候。

如果您想查看本文的完整工作演示，可以在此处找到[它](https://github.com/chimon2000/hello_scoped_model)。
