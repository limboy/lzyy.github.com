---
layout: post
title: 不可变对象的魅力
category: tech
tag: 技术
---

> 10x Engineer: a developer who incurs technical debt so fast he appears more productive than the 10 developers tasked w/ cleaning his mess up

我们都知道，全局变量应该尽量少用或不用，因为它会带来两个明显的问题：耦合和不确定性。有了它，单元测试就不好进行，即使通过了测试，也不能确保这个全局变量变了之后是否能通过测试。 我们经常使用的单例就有全局变量的意味：外部可以直接拿来用，并且可以在任何地方被修改。

为了加快开发速度，往往会以功能实现优先，其中的一个「方法」就是提供可变对象，比如像 OC 里的 `NSMutableDictionary`。前两天正好遇到一个与此有关的 case，可以拿出来说一下。

我们的网络层发送请求时，默认会带上一些系统参数，比如 iOS 系统版本，app 版本等。同时如果用户已经登录了，也会带上一些用户信息，比如 `token`。为了方便复用，我们每次请求都会把已有的参数放在一个自定义的网络请求类，假设这个类的名字叫`APIClient`。同时又允许外部动态添加一些请求参数，比如用户信息，是否启用调试等。

出于方便考虑，我们给 `APIClient` 类加了一个 `NSMutableDictionary` 属性 `builtinParameters`，这样外部只要拿到 `APIClient` 的单例，然后往这个属性里面添加额外的参数就可以了。`APIClient` 里会把这些参数组装成 `querystring` 发送给服务端。

就这样正常运行了一段时间，忽然有一天发现用户登出后，Ta原先的一些登录信息还是被发送给了服务端。因为这个网络请求类并没有做过改动，所以排查起来没什么头绪。经过多次抓包和跟踪后，终于定位到了问题的原因：`builtinParameters` 这个属性在外部被改变了。更细致的原因跟一次重构有关，这里就不展开了。

所以可变对象会给调试和维护带来麻烦，尤其是这些对象多起来后，更是不好处理。

「可变对象」就像男人的承诺：不可信，不知道什么时候会因为什么原因发生改变。

「不可变对象」就不一样了，拿到的是什么，就是什么，不会改变，除非被换成了一个新的。

但「这世界唯一不变的就是变化」，不可变对象如何来应对这个充满变数的环境呢？

先来看一下这个「动画」

![](http://31.media.tumblr.com/fe521bb54c25c173355632a3f5e029fe/tumblr_nmobaa6IQa1ruhxczo1_500.gif)

通过连续快速地翻页来形成动画的假象，这主要是利用了人眼的[视觉停留](https://www.wikiwand.com/zh-hant/%E8%A6%96%E8%A6%BA%E6%9A%AB%E7%95%99)。

有点扯远了，但这跟「不可变对象」可变化，还挺像的，这些图像是静态的，不变的，但这本书让这些图像变了起来。这本书可以是一个类，其中的图片可以是一个 ivar，外部可以给这个 ivar 设置新的 value，这样对于 class 来说，就可以放心地使用这个 ivar，不用担心什么时候这个 ivar 自身会发生变化，比如 `[dict addObject:]`。

再来看看 ReactJS+Flux 是如何使用 Immutable Objects 的。

先来说说 Flux，用一张图就能差不多描述清楚了

![](http://facebook.github.io/flux/img/flux-simple-f8-diagram-with-client-action-1300w.png)

Flux 的一个特点是，数据是单向流动的，就像漏斗一样。

`Dispatcher` 是一个「分发器」，它的职责是接受所有的 Action，简单组装后，扔给 Store，其他的事情就不管了。

`Store` 是一个数据中心，当 Store 接收到 Dispatcher 过来的 Action 时，会根据这些 Action，生成新的 States，然后再把它传给 View。

`View` 拿到这些新的 States 后，会有选择的进行组件的更新。

这里的 States 就是一个不可变对象，Store 不会去修改 States 的某个属性，而是生成一个新的。但是生成一个新的成本不是会很大？是的，所以可以利用 [Copy on Write](https://www.wikiwand.com/en/Copy-on-write) 等技术进行优化。

接下来看看 ReactJS 拿到这个新的 property 后会如何处理，先来看一张图

![](/image/should-component-update.png)

View 会对新的 property 和当前的 property 做比较，如果数据是一致的，那就什么也不做（就像 C2 一样），它下面的节点也不用比较了；如果数据不一致，再往下找，一直找到那[几]个需要更新的节点。

这整个过程没有使用到 Mutable Objects，但照样 Getting Things Done。

### 小结
Immutable Objects 和 Mutable Objects 有各自的使用场景，后者可以作为前者的容器。比如 Facebook 在[他们的架构文章](http://www.infoq.com/news/2014/10/Facebook-ios-architecture)中提到，他们的 Model 类是只读的，但 Model 寄生的对象可以更新 Model。我们可能习惯了使用可变对象，因为各种教程/编程书籍上都是这么写的，但合理地使用「不可变对象」有时会带来更好的效果。

### References

* [How Immutable State Helped Facebook to Improve Its iOS App Architecture](http://www.infoq.com/news/2014/10/Facebook-ios-architecture)
* [Controlling Complexity in Swift](https://realm.io/news/andy-matuschak-controlling-complexity/)
* [Simple Made Easy](http://www.infoq.com/presentations/Simple-Made-Easy)
* [Domain Driven Design](http://www.infoq.com/minibooks/domain-driven-design-quickly)
