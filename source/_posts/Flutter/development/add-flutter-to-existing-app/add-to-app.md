---
layout: post
title:  "(译)添加 Flutter 到已有的应用"
date:   2020-03-23 08:55:17 +0800
categories: 
   - Flutter 开发
tags:
    - 混合开发
---
[原文链接](https://flutter.dev/docs/development/add-to-app)

有时候,短时间内用 Flutter 重写你整个应用是不切实际的. 对于这些情况来说,Flutter 能够以库或者模块的形式嵌入到你已有的应用中.然后这个模块就能够导入到你的 Android 或 iOS应用中,使得你的应用的部分UI用 Flutter 进行渲染.或者,只是运行共享Dart的逻辑.

区区几步,你就能把 Flutter的生产力和表现力带到你的应用中.

在 Flutter 1.12 版本中,支持的基本场景是一个应用同一时刻支持集成一个全屏的 Flutter 实例. 它当前有着以下的限制:

* 运行多个Flutter实例或运行部分的界面视图可能会有未知的行为
* 在后台模式中使用 Flutter 仍然处于过程阶段
* 将Flutter库打包进另一个共享的库或者打包多个Flutter库到应用中目前是不支持的.
* 在向 Android 添加 flutter 中使用的插件应该经受[https://flutter.dev/go/android-plugin-migration](https://flutter.dev/go/android-plugin-migration)这一步,并使用基于 FlutterPlugin 的 API. 不支持 Flutter 插件 可能有未知的行为如果它们作出的假定在 add-to-app 中不成立的(例如假定 Flutter Activity 是一直在的)