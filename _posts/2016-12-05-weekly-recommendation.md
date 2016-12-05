---
layout: post
title: 每周推荐 (2016/12/05)
category: weekly-recommendation
tags: 周荐
---

### 文章

#### 费曼学习法
[Source](https://www.farnamstreetblog.com/2012/04/learn-anything-faster-with-the-feynman-technique/), 费曼是著名的物理学家、诺贝尔奖获得者，在量子力学领域有杰出贡献。他提出了快速学会任何技术的 4 步曲（虽然我有点怀疑这是不是他提出来的，但还是挺有效的）。

1. 挑一个概念
2. 对一个不了解这个概念的人阐述此概念
3. 找到说不清楚的地方重新整理
4. 回顾和简化

> The person who says he knows what he thinks but cannot express it usually does not know what he thinks.
> 
> -- Mortimer J. Adler

「我以为理解」和「我真的理解」之间存在着巨大的差异，跨过这个沟壑的最大障碍就是自己。如果找不到合适的人来阐述，写博客也是个很不错的途径，因为要把散落的知识点整理成有条理的文章绝不是件容易的事，这个过程也会强迫思考那些并没有彻底掌握的知识点。

#### People Don’t Buy Products, They Buy Better Versions of Themselves
[Source](https://stories.buffer.com/people-dont-buy-products-they-buy-better-versions-of-themselves-5d6552aad4c6#.xqdloqdpv), 这篇文章其实就一个观点：当我们推出产品时，多从使用者的角度去考虑，能为他们带来那些好处／便利，而不是自己做了什么。同时列举了几个例子：

![](/image/14808962699682.jpg)

如果从制作者的角度来看就是：一个容量很大体积却很小的音乐播放器。但这两句话对消费者的概念完全不一样。

![](/image/14808963568315.jpg)

> “Here’s what our product can do” and “Here’s what you can do with our product” sound similar, but they are completely different approaches.
> 
> — Jason Fried

#### How Uber, Airbnb, and Etsy Attracted Their First 1,000 Customers
[Source](http://hbswk.hbs.edu/item/how-uber-airbnb-and-etsy-attracted-their-first-1-000-customers), 本文的核心观点可以提炼为

1. 鸡和蛋相比，蛋更重要，初期要把更多的精力放在蛋上。
2. 扩张很重要，但不同时期有不同的策略。

以 Airbnb 为例，刚上线时没人知道它，为了「偷」用户，写脚本从 craglist 里扒了一群房东的信息，然后让他们把内容也同时发一份到 Airbnb 上，因为没什么损失，所以不少房东也同意了。这就解决了初期的「蛋」问题。同时要形成差异化竞争，用户体验也很重要，因此为了能够跟酒店媲美，请了专业的摄影师到房东的家里拍摄。然后找准供需比较大的城市开始发力，这样初期的用户就有了。

Uber 也差不多，在推出「Uber X」之前，先上线「Uber Black」，由专业的司机驾驶，并且找到用车高峰期的场合（比如演唱会结束），为这些人提供服务，通过良好的用户体验来赢得口碑，口口相传之后，第一批用户也有了。

但要赢取更多用户的话，比如 1 千万，则又得靠另外的途径，比如广告。所以不同阶段获取用户的方式各有不同。


### 编程

#### Diff.swift
[Source](https://github.com/wokalski/Diff.swift), Diff 是一个可以比较出两个 Collection 差异的类库，同时也对 UICollectionView/UITableView 添加了额外的支持。Instagram 在[重构他们的 Feed 流](https://realm.io/news/tryswift-ryan-nystrom-refactoring-at-scale-lessons-learned-rewriting-instagram-feed/)时也用到了类似的 Diff 技术，都是采用[寻找最长公共子序列](https://en.wikipedia.org/wiki/Longest_common_subsequence_problem)的方式。它可以和上一篇文章提到的 ReSwift 有效地结合，来发挥 ReSwift 更大的威力。

#### Generic protocols & their shortcomings
[Source](http://krakendev.io/blog/generic-protocols-and-their-shortcomings), 这篇文章讲的是 Swift 的 Generic protocols 的定义、它的问题以及绕过去的方式。

只要带有 `associatedtype` 或 `Self` 的都是 `Generic protocols`，更正常的 protocol 不一样的是，它不能作为变量的类型，不然会报错

![](/image/14808986323674.jpg)

文章里解释了出现这个 error 的原因，主要是编译器无法在编译期确保所有的调用不出问题，因为语法上不支持 `let myth: MythicalType<Hero>` 这样。变通方式是通过 `class` 中转下，因为 class 可以保证类型一致。

```swift
class Storybook<T: MythicalType> {}
```

但在处理带 generic property 的 protcol 时，还是会比较麻烦。

#### Mixins and Traits in Swift 2.0
[Source](http://matthijshollemans.com/2015/07/22/mixins-and-traits-in-swift-2/), 虽然是针对 Swift 2.0 来讲的，对 3.0 也同样适用。POP 能够帮助我们以 `Composition` 而不是 `Inheritance` 来考虑问题。

文中举了一个例子，塔防游戏中，闪电妖怪具有发射激光的能力，因此在 `ZapMonster` 这个类中加了这个方法，但后来要求城堡也要有这个能力，这时就有两种处理方式：

1. 在 `GameObject` (相当于 `NSObject`) 中添加此方法。
2. 把发射激光的能力抽出来作为单独的 helper。

采用方法 1 的问题在于 `GameObject` 很快就会变成 [God Object](https://en.wikipedia.org/wiki/God_object)，其他不需要此能力的子类也被迫具有了这些能力，同时一个类做的事情太多也不是件好事。

方法 2 比 1 会好些，但同样也有问题，比如无法直观地知道某个 Object 到底具备哪些能力；这个 helper 也需要初始化，也需要管理。

然后就引入了 [trait](https://en.wikipedia.org/wiki/Trait_(computer_programming))，trait 的意思通过抽离出来的一系列方法来扩展已有的类，比如把 shooting 抽出来作为一个 trait，move 也可以作为一个 trait，需要这些能力的类，直接拿来用即可。在 Swift 中可以通过 POP 来实现。

```swift
class AIPlayer: GameObject, AITrait, GunTrait, RenderTrait, HealthTrait {
  // ...
}
```

作者之后又举了个登录验证的例子（这个例子不是特别好，因为每个验证都抽出来作为 protocol 的话，就太多了），和模拟 Ruby Enumeratable 的例子。



