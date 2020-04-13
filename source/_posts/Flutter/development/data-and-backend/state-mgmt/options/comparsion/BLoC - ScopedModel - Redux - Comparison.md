---
layout: post
title:  "(译)BLoC-ScopedModel-Redux之间的比较"
date:   2019-04-17 23:13:17 +0800
categories: 
   - Flutter 开发
tags:
    - 状态管理
    - BLoC
    - ScopedModel
    - Redux
---
[原文链接](https://www.didierboelens.com/2019/04/bloc---scopedmodel---redux---comparison/)

BLoC-ScopedModel-Redux之间的比较，什么时候使用它们，以及原因是什么？

难度：初级

<!--more-->


## 介绍

BLoC,ScopedModel,Redux这三者之间差异，各自的使用场景以及各自的优缺点。

在网上可以找到许多关于这个话题的问题和和答案，但是其中有没有正确的选择呢？

为了给出我自己的分析，我考虑了两种不同类型的用例，使用3个框架覆盖这些用例，构建了出一个快速解决方案，并进行比较。

可以在GitHub上找到包含Redux，ScopedModel和BLoC解决方案的[完整源代码](https://github.com/boeledi/redux_scopedmodel_bloc)。

___

## 它们是什么？

为了更好地理解这些差异，我认为快速的过一下它们的主要原则是有所帮助的。

### Redux

介绍

Redux是一个应用程序状态管理框架。换句话说，它的主要目标是管理状态。

Redux的架构基于以下原则：

* 单向数据流

* 一个仓库(Store)
   仓库(Store)的行为就像是Redux的协调者。仓库(Store)：
   * 只储存一个状态
	* 公开叫做dispatch一个入口，它的入参只有Actions
	* 暴露一个getter来获取当前状态
	* 允许通过注册StreamSubscription，从而能接收到任何应用于状态的变化
	* 将actions和store分发到第一个MiddleWare
	* 将actions和当前状态分发给Reducer（可能是由多个reducer组成的外观）
	
* Actions

Actions是Store接入点唯一接受的数据类型。Actions结合当前的状态供中间件和Reducer用来处理一些功能，这可能导致状态的修改。

Actions只描述发生过的事情。

* 中间件

中间件通常是一种旨在基于Action异步(但不一定)运行的功能。中间件只使用状态(或一个Action作为触发器)但不会改变状态。

* Reducer

Reducer通常是一个同步函数，它根据Action-State的结合体进行一些处理。处理的结果可能会导致一个新的状态。

Reducer是唯一允许改变状态的对象。

值得注意的是，根据Redux的建议和良好实践，每个应用程序只有一个仓库。要拆分数据处理逻辑，建议使用`reducer composition`,而不是使用许多仓库。

它是如何工作的

如下的动图展示了Redux的工作原理：

![](https://www.didierboelens.com/images/models_redux_animation.gif)

说明：

* 当UI层发生某些事情(但实际上不限于UI)时，会创建一个Action并将其发送到Store(通过store.dispatch(action));
* 如果配置了一个或多个中间件，则按顺序调用它们，向它们传递Action和指向Store的引用（因此,也是State,间接);
* 中间件本身可以在处理过程中向仓库发送一个Action;
* 然后Action和当前状态也被发送到Reducer
* Reducer是唯一可能改变状态的对象
* 当状态发生改变，仓库会通知所有已注册的听众，告诉它们状态发生了变化
* 然后,UI(但不限于UI)可以采取与状态更改相关联的适当操作。

实现

Redux最初是为Javascript开发的，并以包的形式移植到Dart。

Flutter专用包(flutter_redux)提供了widget,例如：

* StoreProvider将Store传递给所有后代Widget
* StoreBuilder从StoreProvider获取Store并将其传递给Widget build方法
* StoreConnector从最近的StoreProvider祖先获取Store，将其转换为ViewModel并将其传递给一个build方法。

___

### ScopedModel

介绍

ScopedModel是一组实用工具，它允许从父Widget向下传递数据模型到它的后代。

ScopedModel约有3个类：

* Model

Model是一个包含与数据和数据相关业务逻辑的类。它被实现为可监听的，并且能在发生变化的时候通知那些对其感兴趣的。

* ScopedModel

ScopedModel是一个Widget，类似于Provider，它持有Model并允许：

* 通过常用的ScopedModel.of<Model>(context)调用来获取Model
* 在请求时，将context注册为底层InheritedWidget的依赖项。

ScopedModel基于一个AnimatedBuilder,它监听Model发送的通知，然后重建一个InheritedWidget，转而InheritedWidget将要求所有依赖项重建。

* ScopedModelDescendant

ScopedModelDescendant是一个Widget，它对模型的变化作出反应;当模型通知发生变化的时候，它就会重建。

它是如何工作的？

以下代码提取了使用Scoped Model模拟“计数器”应用程序的代码。

![](https://www.didierboelens.com/images/models_scopedmodel_code.png)

以下动画展示用户点击Button时会发生什么。

![](https://www.didierboelens.com/images/models_scopedmodel_animation.gif)

说明：

* 当用户点击RaisedButton时，将调用model.increment（）方法
* 此方法只是增加计数器值，然后调用notifyListeners（）API，因为Model实现了Listenable abstracts类。
* 这个notifyListeners（）被AnimatedBuilder截获，它重建了它的InheritedWidget子节点
* InheritedWidget插入Column及其子项
* 其中一个子节点是ScopedModelDescendant，它调用其构建器，该构建器引用现在拥有新计数器值的Model。

### BLoC

介绍

BLoC模式不需要任何外部库或包，因为它只依赖于Streams的使用。但是，对于更友好的功能（例如主题），它经常与RxDart包结合使用。

BLoC模式依赖于：

* StreamController

StreamController公开StreamSink，从而在Stream和Stream中注入数据，从而监听在Stream内部流动的数据。

* StreamBuilder

StreamBuilder是一个Widget，它监听流并在Stream发出新数据时重建。

* StreamSubscription

StreamSubscription允许监听流发出的数据并做出响应。

* BlocProvider

BlocProvider是一个实用的Widget，通常用于保存BLoC并使其可用于后代Widgets。

它是如何工作的？

以下动画显示了当我们将一些数据注入其中一个Sink或通过API时BLoC如何工作。
（我知道，BLoC只应该与Sinks和Streams一起使用......但是没有什么能真正阻止使用API​​ ......）

![](https://www.didierboelens.com/images/models_bloc_animation.gif)

说明：

* 一些数据被注入其中一个BLoC接收器
* 数据由BLoC处理，最终从一个输出流中发出一些数据
* 使用BLoC API时也同样适用...

有关BLoC概念的其他信息，请参阅我关于该话题的2篇文章：

2018年8月20日 - [Reactive Programming - Streams - BLoC](/flutter/state/bloc/2019/04/13/ReactiveProgramming-Streams-BLoC.html)

2018年12月1日 - [Reactive Programming - Streams - BLoC - Practical Use Cases](/flutter/state/bloc/2019/04/13/ReactiveProgramming-Streams-BLoC-PracticalUseCases.html)

## 第二部分

现在我们对于它们是什么以及它们是如何工作的有了更好的了解，让我们开始对它们进行比较。

为此，我将举两个例子来说明它们的差别，以及各自的优缺点。

### 例1：用户认证

这个常见的用例非常有趣，因为它涉及某种类型的共享状态。在此示例中，我希望页面的行为如下：

![](https://www.didierboelens.com/images/models_case_1.gif)

* 显示用户未经过身份验证的文本以及用于模拟身份验证的按钮;
* 模拟身份验证过程正在进行时的CircularProgressIndicator;
* 经过身份验证的用户的名字和姓氏，以及要注销的按钮。

### 代码比较
以下2张图片并排显示了应用程序初始化以及页面相关的代码。
![](https://www.didierboelens.com/images/models_comp_appl_case1.png)
例1-Application
![](https://www.didierboelens.com/images/models_comp_page_case1.png)
例1-page

我们可以看到，没有太大的区别。

缺乏重大差异可能是我做出的架构决策的结果，因为我要求我们能够从应用程序的任何地方“注销”。因此，我需要ScopedModel和BLoC解决方案将各自的模型和bloc注入到MaterialApp之上，以后可以在任何地方都可以使用。

但是，还是有区别的，让我们来看下。

___

### 差异和观察结果

#### 文件数量

Redux这个解决方案会导致更多文件(如果我们想要坚持“一个实体，一个文件”范式的话)。即使我们根据它们的职责重新组合实体(例如将所有操作放在一起),我们仍然有更多的文件。

ScopedModel解决方案需要更少的文件，因为模型同时包含数据和逻辑。

与ScopedModel相比,如果我们想要拆分模型和逻辑，BLoC解决方案需要一个额外的文件，但这不是强制性的。

#### 代码执行

在Redux中，由于它的运行方式，我们会无缘无故有着更多的代码被执行。实际上，编写reducer的方法是基于状态评估的，例如：“if action is...then”，同样适用于中间件。

此外，可能是由于flutter_redux包的实现原因，StoreConnector需要一个Conveter，尽管它有时这不是必需的。这个Converter旨在提供一种生成ViewModel的方法。

ScopedModel和BLoC解决方案似乎都需要较少的代码执行。

#### 代码复杂度

如果你记住了它总是由Action触发所有中间件，使其按顺序运行（直到它们要实现一些异步的事情),那么reducer需要基于action的类型的比较来做一些事情，代码还是相对简单的。但是,很快它将需要使用reducer组合的概念（请参阅combineReducers与TypedReducer）。

ScopedModel解决方案看起来像是代码最简单的解决方案:您调用了一个更新model的方法，model会通知监听者。但是，对于监听者来说，知道导致被通知的原因并不明显，因为对model的任何修改都会生成通知(即使它对该某个监听器不感兴趣）。

BLoC解决方案有点复杂，因为它涉及Streams的概念。

在背后，flutter_redux解决方案还依赖于Streams的使用，但从开发人员的角度来看这是隐藏的。

#### (re-)Build的数量

如果我们看一下应用程序重建部分的次数，就会变得有趣了。

在内部，flutter_redux使用Streams的概念来响应应用于State的更改，如果您不尝试通过StoreProvider.of(context)API访问Store，则只有StoreConnector将重建（如使用StreamBuilder)。这使得flutter_redux实现从重建角度来看很有趣。

ScopedModel解决方案是生成更多build的解决方案，因为每次Model通知其监听器时，它都会在ScopedModel()widget(实际上在底层AnimatedBuilder下)下重建整个树。

理想情况下，基于StreamBuilder响应变化的BLoC解决方案是flutter_redux，它导致较少的build（仅重建与StreamBuilder相关的部分）。

#### 代码隔离

在Redux中，reducer和middleware是“通常”但不一定是顶级函数/方法，意味着不属于类。因此，没有什么能阻止它们超出Redux Store的范围，这不是理想的。

ScopedModel和BLoC都倾向于易于代码隔离：对于model或bloc会有特定的类。
___

#### 例1：结论

对于这种特定情况，因为没有性能限制(重建方面的限制),我个人认为没有孰优孰劣之分。

我需要指出的Redux的一个优点，就是能够插入一个中间件来记录不同的Action:你只需要在初始化Store时添加中间件方法的引用。

对于ScopedModel和BLoC，这还有很大的提升空间。

### 例2：

针对这种情况，我们将模拟某种仪表板，用户可以在其中动态添加新面板。每个面板模拟一些实时数据演变(例如股票交易所)。用户可以单独为每个面板打开/关闭实时数据。

这是应用程序的样子：

![](https://www.didierboelens.com/images/models_case_2.gif)

#### 代码比较

由于我想坚持Redux主要原则（每个应用程序一个Store），因此在Redux中实现这一点更难，因为ApplicationState需要记住和处理每个单独的Pan​​el。

现在，如果你不想坚持这个原则，实际上没有任何东西阻止你使用多个仓库，每个面板一个。这么处理的话，代码会变得简单一点。

关于ScopedModel和BLoC版本，代码则非常相似。
___

### 差异和观察结果

#### 文件数量

同样，Redux比起其它2个解决方案需要更多文件(即使我们尽可能重新组合)。

对于这种情况，ScopedModel和BLoC解决方案都需要相同数量的文件。

#### 代码执行

与案例1的评论相同。

Redux执行的代码比ScopedModel和BLoC解决方案多得多，因为reducer基于状态评估，例如：“if action is ... then”，同样适用于中间件。此外，还需要3个StoreConnector实例：

* 在页面级别添加新的Panel
* 在Widget级别获取统计信息
* 在Widget级别处理打开/关闭统计数据的按钮。
* ScopedModel需要额外的代码执行而不是BLoC，因为ScopedModel依赖于Listenable / InheritedWidget来在每次模型更改时重建。

每个Panel，基于ScopedModel的解决方案需要：

* 一个ScopedModel（注入）
* 一个ScopedModelDescendant来处理统计信息的显示
* 一个ScopedModelDescendant处理打开/关闭统计数据的按钮

BLoC执行的代码代码。每个Panel,解决方案需要：

* 一个StreamBuilder来显示统计数据
* 一个StreamBuilder来处理打开/关闭统计数据的按钮

#### 代码复杂度

Redux解决方案更复杂，因为它需要在3个不同的地方调度Actions：

* 在页面级别，当我们需要实例化一个新的Panel时
* 在Widget级别，我们需要通过按钮打开/关闭定时器
* 在中间件级别，从服务器获取新的统计信息值

ScopedModel和BLoC解决方案的复杂度分别仅位于Model和BLoC级别。由于每个Panel都有自己的Model或BLoC，因此代码不那么复杂。

#### (re-)Build的数量

Redux解决方案是导致大部分重建的解决方案。

在实现中，基于“每个应用程序一个仓库”，每次更改应用于ApplicationState时，都会重建所有内容，这意味着：

* 当我们添加一个新的Panel
* 当我们打开/关闭统计数据集时，所有面板都会重建
* 将新值添加到一个Panel时，将重建所有Panel

如果我选择每个面板有一个Store，那么(重新)构建的数量会少很多。

关于ScopedModel解决方案，重建次数会少一些：

* 当我们添加一个新的Panel
* 限于一个小组，何时
  * 我们打开/关闭统计数据集
  * 为特定Panel收集新值

  
最后，BLoC解决方案需要更少的重建：

* 当我们添加一个新的Panel
* 当我们打开/关闭统计数据集时，只重建与特定面板相关的按钮
* 当为特定Panel收集新值时，仅重建Panel。

#### 例2：结论

对于这个特定情况，我个人发现BLoC解决方案是代码复杂性和重建方面的最佳选择。

次之是ScopedModel解决方案。

Redux架构不是此解决方案的最佳选择，但仍可以使用它。

### 其他包

在下结论之前，我想提一下现在围绕Redux的概念存在其余包，下面两个可能会引起倾向使用Redux的人的兴趣：

* [rebloc](https://pub.dartlang.org/packages/rebloc)，结合了Redux和BLoC的各个方面

这个包非常有趣，但仍然没有解决与reducer的概念相关联（如果action是......那么）的代码执行的开销问题。但是，它是值得一看的。

* [fish_redux](https://pub.dartlang.org/packages/fish_redux)，来自阿里巴巴闲鱼团队。

此包与其说是状态管理框架，还不如说是基于Redux的应用程序框架。这个包非常有趣，但需要完全改变开发应用程序的方式。此解决方案使得根据Actions和Reducers更容易构建代码。

### 结论

这个分析让我基于两个不同的用例，一览无余地比较了三个最常使用的框架。

这三个框架各有利弊，我会在下面一一列出，这当然只是我个人的观点。

Redux

* 优点
	* 由于Reducers是唯一可以执行从一个状态到另一个状态的过渡的对象，Redux允许集中管理一个状态。这使得状态转换完全可预测并且可以完全测试。
	* 在流程中插入中间件的便利性也是一个优点。例如，如果您需要不断验证与服务器的连接或跟踪活动，这是此类例程的完美占位符。
	* 它迫使开发人员根据“事件 ->操作 ->模型 ->视图模型 ->视图”来构建应用程序。

* 缺点
	* 一个Store和一个巨大的状态（如果你想坚持Redux良好实践）
	* 使用顶级功能/方法
	* reducers和middlewares级别的“如果......然后”比较太多了
	* 重建次数过多（每次状态发生变化）
	* 需要使用外部包，其中包含随着更改而发生的风险。

* 要使用
  * 当您需要处理全局应用程序状态时，我可以推荐Redux，例如用户身份验证，购物车，偏好(语言，货币)
  
* 不使用
	* 当你需要处理多个事件的实例时，我不会推荐Redux，每个实例都有自己的状态
	
Scoped Model

* 优点
	* ScopedModel使模型及其逻辑在一个位置重新组合变得非常容易。
	* ScopedModel不需要任何Streams概念的知识，这使得初学者更容易实现。
	* ScopedModel可用于全局和本地逻辑
	* ScopedModel不仅限于状态管理

* 缺点
	* ScopedModel没有提供任何方法让代码知道模型的哪个部分发生了变化并导致调用ScopedModelDescendant太多(重新)构建。
	* 每次模型通知其侦听器时，与该模型相关的所有内容都会重建（AnimatedBuilder，InheritedWidget ......）
   * 需要使用外部包，其中包含随着更改而发生的风险。

* 要使用
	* 当开发人员不熟悉Streams时
   * 当模型不是太复杂时

* 不使用
   * 当出于性能原因，应用程序需要减少build数量时。
   * 当应用程序需要准确知道模型的哪个部分已经改变时

BLoc

* 优点
	* BLoC可以轻松地将业务逻辑重新组合到一个位置
	* BLoC使得很容易精确地确定任何变化的性质（通过其基于Streams的输出接口）
	* 由于使用了StreamBuilder Widget，BLoC可以很容易地将(重新)构建的数量限制到严格的最小值
	* Streams的使用非常强大，为许多动作打开了大门（转换，分离，去抖......）
	* BLoC可用于全局和本地逻辑
	* BLoC不仅限于状态管理
	* 不需要使用任何外部包。

* 缺点
	* 如果你想坚持总体原则，我们就只能处理sinks和streams。
	* 就个人而言，我的BLoC还会暴露getter/setters/API，这会消除这种“缺点”。
	* 初学者很难从BLoC开始，因为它需要对Flutter实际工作方式有更多的了解

* 要使用的
  * 我没有看到任何限制。
  
* 不使用
  * 我没有看到任何建议不使用BLoC的情况，除非开发人员不熟悉Streams的概念。

因此，有没有一个完美的解决方案？

事实上，我会说没有“单一”的完美解决方案。这实际上取决于您的使用案例，而且，它实际上取决于你对框架的熟悉程度。

作为结论，我只谈论下自己的看法。

到目前为止，我从未在任何项目中使用过Redux，而且我从来没有感觉到我错过了什么。这同样适用于ScopedModel。

几个月前，我开始使用BLoC，我几乎可以在任何地方使用这个概念，它非常方便。它确实使我的代码更清晰，更容易测试，更加结构化和可重用。

我希望这篇文章能给你一些更多的见解。

请继续关注新文章，让我祝你编程愉快！









