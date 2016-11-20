---
layout: post
title: 蘑菇街 App 的组件化之路·续
category: tech
tag: 技术
---

前几天在「移动学习分享群」分享了关于蘑菇街组件化方面的一点经验，由于时间和文字描述方面的限制，很多东西表述的不是很清楚，让一些同学产生了疑惑，casatwy老师也写了篇[文章](http://casatwy.com/iOS-Modulization.html)来纠正其中的一些实现，看完之后确实有不少启发。

#### 统一的调用实现
将「URL 调用」和「组件间调用」通过 runtime 达到统一，通过 prefix 的方式来避免安全上的一些漏洞。看起来确实会舒服些，也比较灵活。

#### 通过 Category 来统一组件对外暴露的接口
支持 `openURL:` 但最终还是走的 target-action，跟内部调用无差别。
这也是我们目前有待提升的点，想知道某个组件支持哪些 URL 或 哪些 Protocol 不够方便，URL 的参数传递也是个问题，将来 URL 发生变动的话，调整起来也比较麻烦。后续会在这块再加强下。

当初决定使用 `openURL:` 来做页面间的跳转，而不是方法调用，主要是考虑到我们的大部分场景都可以通过这种方式解决，因此就这么定了。`openURL:` 更像 Android 里的 「隐式 Intent」，不关心谁来处理这个 URL，由系统（MGJRouter）来决定。而方法调用更像「显式 Intent」或者 RPC，明确地知道应该由谁来处理。前者的好处是可以更少地关心业务逻辑，后者的好处是可以很方便地完成参数传递。

### 更明确的表述

1. `openURL` 只是页面间的调用方式
2. 组件间的调用通过 protocol 来实现

每个组件都有一个 `Entry`，这个 `Entry`，主要做了三件事

* 注册这个组件关心的 URL
* 注册这个组件能够被调用的方法/属性
* 在 App 生命周期的不同阶段做不同的响应

#### 注册这个组件关心的 URL
![MGJRoute](/image/MGJRouter.png)


```objc
[MGJRouter registerURLPattern:@"mgj://detail?id=:id" toHandler:^(NSDictionary *routerParameters) {
    NSNumber *id = routerParameters[@"id"];
    // create view controller with id
    // push view controller
}];
```

URL 的注册会有对应的 block，拿到这个 URL 后，想怎么折腾就怎么折腾。

#### 注册这个组件能够被调用的方法/属性
当有一些场景不适合用 URL 的方式时，就可以通过注册 protocol 来实现

![ModuleManage](/image/ModuleManager.png)

`[ModuleManager registerClass:ClassA forProtocol:ProtocolA]` 的结果就是在 MM 内部维护的 dict 里新加了一个映射关系。

`[ModuleManager classForProtocol:ProtocolA]` 的返回结果就是之前在 MM 内部 dict 里 protocol 对应的 class，使用方不需要关心这个 class 是个什么东东，反正实现了 `ProtocolA` 协议，拿来用就行。

这里需要有一个公共的地方来容纳这些 public protocl，也就是图中的 `PublicProtocl.h`

#### 在 App 生命周期的不同阶段做不同的响应
上一篇文章中有提到，这里简单说下，`ModuleEntry`，实现某个特定的协议(该协议继承自 `UIApplicationDelegate` )，然后实现对应的方法即可。

### 针对casatwy那篇文章的一些回应

> 单纯以openURL的方式是无法胜任让一个App去实施组件化架构的

同意，所以我们并不只有 `openURL` 一种方式

> 根本无法表达非常规对象

单纯地通过 `openURL` 确实不太好表达，但我们并不只有 `openURL` 一种方式

> 注册URL的目的其实是一个服务发现的过程，在iOS领域中，服务发现的方式是不需要通过主动注册的，使用runtime就可以了。另外，注册部分的代码的维护是一个相对麻烦的事情，每一次支持新调用时，都要去维护一次注册列表。如果有调用被弃用了，是经常会忘记删项目的。runtime由于不存在注册过程，那就也不会产生维护的操作，维护成本就降低了。

> 由于通过runtime做到了服务的自动发现，拓展调用接口的任务就仅在于各自的模块，任何一次新接口添加，新业务添加，都不必去主工程做操作，十分透明。

尽管通过 runtime 可以做到这些，但最终还是要通过维护 `Category` 来暴露新增的 Target-Action，所以 runtime 虽然不存在注册过程，但实际使用过程中，还是会有注册过程，还是需要去维护。

> 没有拆分远程调用和本地间调用

从上面的图可以看到，我们其实是分为「组件间调用」和「页面间跳转」两个维度，只要 app 响应某个 URL，无论是 app 内还是 app 外都可以，而「组件间」调用走的完全是另一条路，所以也不会有安全上的问题。

> 以远程调用的方式为本地间调用提供服务

同上

> 本地间调用无法传递非常规参数，复杂参数的传递方式非常丑陋

同上，使用 Protocol

> 必须要在 app 启动时注册 URL 响应者

是的，就蘑菇街的方案来说，这步不可避免。

> 这个不必要的操作会导致不必要的维护成本

维护只是在组件内部做调整，并不需要在主工程里做修改。如果采用 Category 的方式，好处是不用在启动时注册，但当组件的接口有变动时，依然要维护 Category，这个成本是免不了的。

> 新增组件化的调用路径时，蘑菇街的操作相对复杂
> 在本文给出的组件化方案中，响应者唯一要做的事情就是提供Target和Action，并不需要再做其它的事情

提供了 Target-Action 之后，还是要在 Category 里添加一个 wrapper 的吧?

> 没有针对 target 层做封装
> 这种做法使得所有的跨组件调用请求直接hit到业务模块，业务模块必然因此变得臃肿难以维护，属于侵入式架构。应该将原本属于调用相应的部分拿出来放在target-action中，才能尽可能保证不将无关代码侵入到原有业务组件中，才能保证业务组件未来的迁移和修改不受组件调用的影响，以及降低为项目的组件化实施而带来的时间成本。

「将原本属于调用相应的部分拿出来放在target-action中」并不是唯一可行的方式，使用 Protocol/URL 注册也可以达到效果。

### 小结
casatwy 的一些思路和思考问题的角度挺不错的，也从他的文章中收获了不少，希望这篇文章能把之前模糊的一些观念说得足够清楚，还有问题的话欢迎继续交流：）



