---
layout: post
title: 为什么我觉得 Flutter 短期内不会流行但依然选择学习它
category: essay
tags: 随笔
---

Flutter 在去年小火了一把：连续两年在 Google IO 上亮相；1.0 正式版 Release；在闲鱼的大规模使用；各种教程文章的释出等等。我在去年 5 月份那样体验过一阵，觉得还挺不错的，但也没有进一步挖掘，感觉还尚早。我对跨平台框架有种抵触心理，因为它们通常打着提升开发效率的幌子，结果却是降低了效率，除了学习成本外，还有以下几个原因:

首先要抹平平台之间的差异这就不是一件小事，很容易出现各种吊诡的 bug，然后就要投入不少精力去找原因，还不一定能找到。而如果采用 Native 开发，相关的资料会多一些，出了问题找到解决方案的可能性也会大一些。

其次 Native 的沉淀会多很多，当你想要实现某个效果时，通常会有一些现成的（虽然不一定成熟）library 可供选择，即使不完全满足需求，也能从代码中找到思路，而跨平台框架的积累一定没有 Native 的多，因此这块也是个较大的劣势。

最后一定有一些场景是跨平台框架解决不了的，这时就需要求助于 Native。如果是多人，就涉及到了协作效率，如果是单人，那为什么不直接用 Native 开发呢？

因此当时虽然觉得 Flutter 不错，还是没有入坑。Flutter 除了要面对这三个问题外，它的学习成本还不低，使用 dart 语言开发，这本身就是一道足够高的槛; 使用声明式写法来表达 UI 也不一定能被接受; 那一大坨的 API 也着实让人发怵；再加上还有很多 issues 待解决。因此我认为短期内它不会流行。

那为什么我依然选择学习它呢，最重要的几个原因是：开发体验超预期；带来的副作用较小；插件机制弥补了局限性；活跃的社区。

### 开发体验

#### IDE

能够使用 VSCode 作为主力 IDE 这本身就有足够的吸引力，Debug、Widget Inspector、Hot Reload、Automatically get packages 等等一应俱全，就连被诟病的嵌套过深，VSCode 也提供了一些便利：在每个括号后面以注释形式标注（但不是真的注释）

![](/image/flutter-vscode-nested.jpg)

对某个类的参数不太清楚，光标移上去即可，想看下实现，Cmd+Click。

![](/image/flutter-vscode-hover.jpg)

想快速看下 Framework 里某个类的实现，Cmd+T

![](/image/flutter-vscode-cmd-t.jpg)

还有一些贴心的小功能

![](/image/flutter-vscode-widget.jpg)

当然也有改进空间，比如特别想要 auto import 功能。如果想要更完善的支持，可以使用 Android Studio，后者还提供了 Widget Tree、Performance Chart 等功能。

#### 开发语言

Dart 这门语言本身并不复杂，看着挺舒服的，没有新发明一些概念，尽量简单（有些地方感觉过于简单了，比如 class 可以同时表示 interface），对异步编程有着很好的支持，自带的标准库基本够用。如果真静下心来看的话，不出三天，语言方面应该不会有太大的障碍了。

Dart 是一门类型安全的语言，跟多数静态语言一样，也支持类型推导，写起来比较舒服。泛型、匿名函数等常见的语言特性都有，甚至支持 mixins。

#### 编写体验

得益于 Flutter 的设计，大多数情况下 UI 展示通过 Widget 的组合就基本搞定了，Widget 就是 Description 或者 Config，告诉框架这个 Widget 的一些信息，框架拿到后再构建一个真实的 View 出来。

状态管理和信息流处理也都有很好的支持，基本上可以用 GUI 编程的最佳体验来写。由于 Flutter 是基于最底层的 VSync 信号结合 Skia / Text 等引擎来构建视图，有时会遇到 Native 很方便地支持但 Flutter 不支持或者需要额外开发的场景，比如 TextFiled 的 Context Menu，Native 什么都不用做，这个 menu 就有了，而 Flutter 并没有，需要自己实现。

举一个点击事件的例子：

```dart
Container(
  child: GestureDetector(
      onTap: () {
        bloc.deleteHabit(habit, context);
        Navigator.pop(context);
      },
      child: Text(
        'delete',
        style: TextStyle(color: Colors.red),
      )),
),
```

是不是很直观：给 Text Widget 添加一个手势，当点击时，执行 `onTap` 里面的逻辑。

### 副作用

一般来说，引入了跨平台框架后会带来一些性能上的损失，App 的 Size 也会大一些，可能还会增加 Crash 率。那 Flutter 在这几块的表现如何呢？

#### 性能表现

我自己试过一个有点复杂的 Demo，Release 模式在 iOS 上跑起来很流畅，效果如下，在 Nexus 6P 上也大致如此

![](https://camo.githubusercontent.com/be369abc92c87ee76810c33719c35161c02b885d/68747470733a2f2f692e6c6f6c692e6e65742f323031382f31322f30362f356330393432383237653436332e676966)

不过网上有说官方的 Demo 在部分 Android 设备上(系统主要为 4.4)出现了卡顿（iOS 正常），可能在 Android 低端机上表现还不够理想。

#### 包大小

我的一个比较简单的 App，上架了之后，在 Google Play 的大小为 7.8M，在 AppStore 上是 15 M，所以大致是 Android 的比原生大 5 M，iOS 的比原生大 10 M 那样。

![](/image/flutter-android-size.jpg)

![](/image/flutter-ios-size.jpg)

#### Crash

目前还没有这方面的数据，因为量不大，不好下结论。从个人体验来说，遇到的概率比较少，在 Develop 模式下倒是遇到过开机 Crash，Release 模式下还没有遇到过。

### 插件机制

Flutter 提供了一套完善的插件机制方便与 Native 端进行数据传送、方法调用、流式处理。大致的实现是定义好两端都认识的基础类型，然后对消息进行编码和解码，再根据不同的消息使用目的（数据传送还是方法调用还是事件订阅）来执行不同的操作，这就给了 Flutter 很大的扩展空间。

有了这套插件机制，Native 和 Flutter 就可以各司其职。Flutter 负责展示相关的，Native 负责提供需要的数据，以及暴露 Native 的能力供调用。比如 App 需要实现跟服务端实时通信的功能，可以在 Native 端开发好功能，然后通过 EventChannel 把数据同步过去即可。

在性能这块，我记得看过闲鱼的一份报告，大概是 10K 以下的数据耗时不到 1ms，因此小数据的互传问题不大。

在 pub.dartlang.org 上，Flutter 相关的插件数量有 2k 多，评分在 90 及以上的差不多有 1k。不多，但也不算少了。

### 活跃的社区

Flutter 项目在 Github 上有近 5 万个关注； 在掘金上，Flutter 标签下有 800 多篇文章；闲鱼团队也在主推 Flutter；Reddit 上 FlutterDev 有近 1 万个关注者；StackOverflow 上也有近万个 Flutter 相关的提问。关注社区的活跃度其实就是想知道会不会在短期内挂掉，从目前的状况看，我觉得可能性比较小，而它被另一个同类产品 PK 掉的可能性则更小，因此值得投入时间去了解它。

### 小结

目前 Fluter 比较适合 Side Project 或探索性的项目，就我有限的开发经验来讲，还是挺舒服的，毕竟用优雅的姿势同时搞定两端还是有吸引力的，这也是我选择学习 Flutter 的主要原因。而公司的主打 App 引入 Flutter 则需要冒一定的风险，遇到问题不一定能够 hold 住，短时间内也不一定能带来多少效率上的提升，还要对支持体系进行改造，可能也就大厂玩得起吧。
