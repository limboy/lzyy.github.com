---
layout: post
title: ReSwift 介绍
category: tech
tags: 技术
---

### 什么是 ReSwift

[ReSwift](https://github.com/ReSwift/ReSwift) 是基于 [Redux](http://redux.js.org/) 思想实现的 Swift 类库。基本的流程如下

![](/image/14808452245719.jpg)

当用户点击了视图上的某个元素时，会发出一个 `Action`，这个 `Action` 包含了两个基本元素：`Action Type` 和 `Action Payload`，比如「点击收藏按钮」这个 `Action`，可能会被描述为：`Action("CollectButtonTapped", ["itemID": 189])`。然后这个 `Action` 就会到达 `Store`，`Store` 也很简单，只做两件事：1. 接收 `Action`；2. 将 `Action` 和 `State` 发送给 `Reducer`。`Reducer` 做的事情就更简单了，接收 `Store` 发出的 `Action` 和 `State`，内部运算之后，返回一个新的 `State`。`Store` 拿到了新的  `State` 后，再把 `State` 发送给 `View`。`View` 渲染新的 `State`。

简单描述下各个模块的职责：

#### View
`View` 可以理解为一个「壳」，所有的数据都由 `State` 提供，这样就把表现层和数据层分开了。

```
view = f(state)
```

#### Action
`Action` 用来描述发生了什么事情，比如不小心用脚踢到了椅子，神经系统就会把这个信息传递给大脑，这个信息就是 `Action`，而大脑就是之后要讲到的 `Store`。

#### Store
这是核心模块，就像大脑会不停地接受到各种 `Action`，并作出反应，只不过在这里 `Store` 并不具备「做决定」的能力，而是把这个 `Action` 交给了所有可能关心它的 `Reducers`。

ReSwift 推荐一个 App 只有一个 `Store`，在实际情况中，如果这么做的话，会带来不少的副作用，比如所有的模块都需要依赖 `Store`，这个 `State` 会很庞大，不可避免的会影响性能。所以，单个页面或模块有一个 `Store` 会比较合适。

#### State
`State` 是一个隐形的杀手，因为使用它极其方便，而它的危害也不会瞬间爆发，就像温水煮青蛙一样，等发现问题越来越多、被各种多线程问题困扰时，就会感受到它的威力了。

所以把 `State` 单独拎出来，并且使用 [Value Types](https://developer.apple.com/videos/play/wwdc2015/414/) 来解决各种多线程或变量被修改导致的问题。

WWDC 的 [Protocol and Value Oriented Programming in UIKit Apps](https://developer.apple.com/videos/play/wwdc2016/419/) 中也推荐使用 Value Composition，而不是继承，同时把 State 集中到一个地方处理，也有助于 Local Reasoning。

### 为什么要使用 ReSwift
确切说来是为什么要使用「单向数据流」的架构模式，主要有这么几个好处：

1. 数据单向流动容易让结构变得清晰，出问题时也更容易排查。
2. 使用了 「Value Types」作为流动的数据，避免各种诡异的「不小心被篡改」或多线程 bug。
3. 在统一的入口处理数据（State），比起散落在各处更加容易控制。

[Readme](https://github.com/ReSwift/ReSwift) 里带了一个简单的 Demo，可以感受下。

### 源码一瞥
ReSwift (3.0.0) 的源码很精简，对 Swift 熟悉的话，很快就能看完。说下我自己在看源码的过程中学到的一些 tips 吧。

#### Reduce 的使用
`reduce` 在函数式编程的领域里会经常被用到，甚至可以实现 `map` / `filter` 等功能，足见其强大。它的运行规则是以函数的处理结果作为初始值，再结合数组中的元素返回处理结果，不断循环，直到数组中的元素全部处理完成。

![](/image/14808586998591.png)

在 Swift 中，它是 `Sequence` 协议扩展的一个方法，签名如下

```swift
public func reduce<Result>(_ initialResult: Result, _ nextPartialResult: (Result, Self.Iterator.Element) throws -> Result) rethrows -> Result
```

在 ReSwift 中有好几个地方都用到了 `reduce`，比如通过它来达到 `combineReducer` 的效果

```swift
public struct CombinedReducer: AnyReducer {
	  // self.reducers 包含了 AnyReducer 的实例
    public func _handleAction(action: Action, state: StateType?) -> StateType {
        return reducers.reduce(state) { (currentState, reducer) -> StateType in
            return reducer._handleAction(action: action, state: currentState)
        }!
    }
}
```

按照入队列的先后，reducer 被依次执行，并且把生成的新的 `State` 作为下一个循环的初始值传递给下一个 reducer。

在处理 `middleware` 时，也有用到类似的技术，不过那个更加复杂些，涉及到[高阶函数](https://zh.wikipedia.org/zh-hans/%E9%AB%98%E9%98%B6%E5%87%BD%E6%95%B0)。

#### 装饰器模式
装饰器模式简单来说就是在不改变类／方法原有功能的前提下，提供了一些额外的能力。比较常见的有 validator，客户端提交的数据要入库前需要做一下校验，不通过的话直接返回。在 python 里装饰器非常常见，比如在一个方法上加一个 `@cached` 或者 `@validate` 等 annotation。

在实现 Reducer 时，有用到这个模式：

```swift
public protocol AnyReducer {
    func _handleAction(action: Action, state: StateType?) -> StateType
}

public protocol Reducer: AnyReducer {
    associatedtype ReducerStateType

    func handleAction(action: Action, state: ReducerStateType?) -> ReducerStateType
}

extension Reducer {
    public func _handleAction(action: Action, state: StateType?) -> StateType {
        return withSpecificTypes(action, state: state, function: handleAction)
    }
}
```

`_handleAction` 对 `handleAction` 做了个校验，（`withSpecificTypes` 函数里如果校验不通过，`handleAction` 不会被执行），这样对于使用者，只需继承 Reducer 实现 `handleAction` 方法，ReSwift 内部调用时会使用 `_handleAction` 来做一些校验。

在 `StoreSubscriber` 里也有用到类似的技术。

#### associatedtype 的使用
通过 `associatedtype`，可以让 protocol 使用 `generic`, Natasha 还写过一篇关于 [PAT 使用的文章](https://www.natashatherobot.com/swift-what-are-protocols-with-associated-types/)，里面以宠物小精灵为例，通过 PAT 让不同的小精灵具备了不同的能力。不过使用了 `associatedtype` 或 `Self` 后，就不能作为变量的类型来声明了，比如 `var something: AProtoclWithAssociatedType` 这样编译器会报错，具体原因可以参考[这篇文章](http://krakendev.io/blog/generic-protocols-and-their-shortcomings)，主要是因为无法指定 Generic 的类型，导致编译器无法在编译期间就确定具体的类型，对于强类型语言来说，这是不能接受的。

ReSwift 中，在定义 StoreType 时，有用到 `associatedtype`

```swift
public protocol StoreType {

    associatedtype State: StateType

    /// Initializes the store with a reducer and an intial state.
    init(reducer: AnyReducer, state: State?)
    
    //...
}
```

在定义 reducer protocol 时，也有用到（也是关联了 StateType）。

#### 对外只读，对内可读写
在 OC 时代，通常的做法是在 .h 里声明为 `readonly`，然后在 .m 的 class extension 里，将同名的属性声明为 `readwrite`。

Swift 没有头文件的概念，直接一句话搞定 `private(set)`

```swift
struct Subscription<State: StateType> {
    private(set) weak var subscriber: AnyStoreSubscriber? = nil
    let selector: ((State) -> Any)?
}
```

subscription 希望外部可以拿到 subscriber，但不要修改它，于是在前面加了 `private(set)`，也就是把 `set` 方法标记为 private。

### 小结
ReSwift 还是挺值的一试的，一方面是因为单向数据流确实对程序的清晰度有帮助，另一方面 ReSwift 的代码很简洁，内部实现比较容易搞明白，这样即使出问题也比较容易定位。[Realm](https://realm.io/news/benji-encz-unidirectional-data-flow-swift/) 上有作者分享的案例，可以参考下。不足嘛肯定也有，比如功能比较简单，只是做了数据流，缺少 Diff 支持，在做列表更新／删除时会比较痛苦；如何与 MVVM 等比较成熟的架构有效地结合起来等。

除此之外，由于数据都通过 State 来传递，可以在出 bug 时，上传当时的 state 内容方便定位；还可以基于 State 来做[时光机](https://github.com/ReSwift/ReSwift#demo)。不妨在 Side Project 中尝试下。

