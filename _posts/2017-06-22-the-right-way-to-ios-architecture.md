---
layout: post
title: The Right Way to Architect iOS App with Swift
category: tech
tags: 技术
---

关于 iOS 架构的文章感觉已经泛滥了，前一阵正好 Android 官方推了一套  [App Architecture](https://developer.android.com/topic/libraries/architecture/guide.html) ，于是就在想，对于 iOS 来说，怎样的架构才是最适合的。带着这个问题，我开始了探索。

## Why Architecture Matters?
这是第一个也是最重要的问题，为什么会出现各种 Architecture Pattern？真的那么重要么？

我们来想一下，无论是做一个 App 还是搭一套后台系统，如果是一次性的，今天用完明天就可以扔掉，那么怎么快怎么来，代码重复、代码逻辑、代码格式统统不重要。

这种场景比较适合黑客马拉松，而真实情况往往是我们的代码需要上线，要对用户负责，而一套好的架构会让这些事情变得更加容易。

### 好的架构简洁且整洁

![](http://s3.mogucdn.com/mlcdn/c45406/170619_4e7gif674kdad56l6iek5lj8i7dl9_984x329.png)

说到架构，往往会想到建筑，软件架构跟建筑不同的点是软件架构会随着时间的推移进行演进，而实体建筑则没这个特性。抛开时间维度，这二者还是有一定的相似性的。

好的架构容易催生好的代码，就像住在干净整洁的房子里，会下意识地让其中的家具、电器、摆饰等也井井有条。

### 好的架构让代码更加容易维护

不容易维护的代码往往有这么几个特点：

1. 抽象程度低
2. 职责不明确
3. 喜欢走捷径

好的架构能对 2 和 3 有一定的作用，对于第 1 点还是要看程序员的能力和经验。

#### 抽象程度低
这样的代码往往是命令式编程产生的，也就是像 CPU 那样的思考方式，把产品经理的需求直观地翻译成代码，而不对其中的共性、本质进行抽离和抽象，时间一长就容易看不懂其中的逻辑，需求一变就要改核心代码。

比如下面这段代码，不知道具体要完成什么任务。

![](http://ww1.sinaimg.cn/large/afe37136gy1fgtcs43v88j20nk0damzy.jpg)

#### 职责不明确
这也是产生「一大坨代码」的原因之一，就像 MVC 模式里，没有说明用户的操作应该在哪里处理，业务逻辑放在什么地方，这样就容易走捷径，怎么方便怎么来，而越是方便到后来就越容易出问题。

#### 喜欢走捷径
这是我们的天性，毕竟能够更快更方便地达到目标，为什么不做呢？

比如我们都知道「通知」用起来很方便，所有涉及到单向数据传递的地方都可以使用，比如 Cell 通过通知向 VC 传递点击事件信息、Model 通过通知向 VC 传递数据信息、VC 之间通过通知进行解耦等等。 

又比如可以很方便地在 VC 存储状态信息，慢慢地 VC 里这些状态变量就多了起来，到后来要维护这些变量就变得非常困难，出了问题也不好排查。

Clojure 的作者 Rich Hickey 有一个非常著名的 [Simple Made Easy](https://www.infoq.com/presentations/Simple-Made-Easy) 分享

> Simple is often erroneously mistaken for easy. "Easy" means "to be at hand", "to be approachable". "Simple" is the opposite of "complex" which means "being intertwined", "being tied together". Simple != easy.

Simple 是我们所追求的，而 Easy 往往会让事情往反方向发展。

### 好的架构能够覆盖大多数场景
产品经理：老板说要做一个插座，具体怎么实现我不管，下周一就要。拿到这个需求之后，你觉得很简单，完美符合需求，就像这样：

![](http://ww1.sinaimg.cn/large/afe37136gy1fgtctgc56dj206u06ujrp.jpg)

可是好景不长，老板新买了一个电脑，只支持两相的插座，而且现在就要，作为工程师，你不能被这么简单朴实的需求难倒，于是稍微动了下脑筋，就出了一个解决方案：

![](http://ww1.sinaimg.cn/large/afe37136gy1fgtctvnyggj206c065dhe.jpg)

虽然丑陋，但是可以工作。但我们的目标不只是可以工作（紧急情况除外），更要优雅地工作。

举一个现实的例子，比如页面间支持通过 Router 进行跳转，但有一天发现有页面间通信的需求，然后就会出来一些 trick 的解决方案，比如发通知或者给 Router 加一个 `- (id)objectForURL:` 的方法，本质上跟上图的解决方案没什么区别。

### 好的架构能够提升开发效率，方便定位问题
好的架构能够支持多人并行开发、一定程度的代码复用、单元测试，出了问题能比较方便地找到原因。这几点是架构要解决的主要问题。

## 当前的状态
目前主流的主要有 MVC 和 MVVM，VIPER 用的会少一些，它们之间的优劣对比这里就不展开了，可以查看这篇文章来了解：[iOS 架构模式 - 简述 MVC, MVP, MVVM 和 VIPER (译) - Coding 博客](https://blog.coding.net/blog/ios-architecture-patterns)

简单总结下：

* MVC 模式过于简单，定的标准过于粗放， 容易滋生捷径。
* MVVM 会好很多，但场景的覆盖还不够全，比如缺少页面间跳转／通信、数据获取等。
* VIPER 更加细致，但有点臃肿。

## How to Define “Right”
每种架构都有自己的特点，如果要定义「Right」的话，至少要符合一些标准，以下是我整理的觉得比较重要的几条：

* 尽量简单
* 结构清晰
* 职责明确
* 符合 GUI 编程的特点

### 尽量简单
简单的事物容易理解，也比较容易接受，用爱因斯坦的话来说「尽量简单，但不要过于简单」。VIPER 其实已经挺完善的了，但就是有点复杂，可以看[这篇文章](https://www.objc.io/issues/13-architecture/viper/)感受下。

### 结构清晰
清晰的结构让外人也能很快地知道每个目录是做什么的，里面的文件起着怎样的作用，自己维护起来也方便。

### 职责明确
也就是 `Separation of Concern` ，每个单元只需要关心自己的事情，跟外部尽量解耦，这样无论是对代码复用和测试都会很有帮助。

### 符合 GUI 编程的特点
GUI 编程和其他的非界面编程还是有差异的，对 GUI 编程的特点进行合适地抽象，并在此基础上形成的架构才更有「对」的感觉。

我比较认同 `view = render(state) + handle(event)` 这个定义，view 本身只做两件事，给 state 包一层漂亮的外衣，同时对用户的操作做出响应。

## Inspiring
差不多心里有谱了，现在来看看相关领域的架构大概是怎样的，找点启发。

### Android Architecture
Android 最近出了一套官方推荐的[架构](https://developer.android.com/topic/libraries/architecture/index.html)，挺细致的，主要的流程如下图所示

![](https://developer.android.com/topic/libraries/architecture/images/final-architecture.png)

大意就是 `ViewModel` 通过调用 `Repository` 从 `Model` 或 `Remote` 中获取数据，然后放到内置的 `LiveData` 里，而 `LiveData` 在 `Activity` 初始化时即被绑定，因此当  `LiveData` 变化时，可以马上反馈到界面。

当用户操作界面时，`Activity` 会捕获到这些事件，然后调用 `ViewModel` 的特定方法，这些方法最终会导致 `LiveData` 发生改变，再次反馈到界面。

整体也是 MVVM 的模式，但也有自己的特点：

* 通过 `LiveData` 来做单向绑定。
* 使用 `Repository` 来统一数据的交互。
* 内置  `Room` 作为持久层。
* 内置 `ViewModel` 供使用。
* 内置 `LifeCycle` 来简化跟生命周期相关的对象的操作，避免内存泄漏。（比如 ViewModel）
* 使用 `Dagger2` 这个依赖注入工具来避免依赖。

### Elm Architecture
> Elm is a functional language that compiles to JavaScript. It competes with projects like React as a tool for creating websites and web apps. Elm has a very strong emphasis on simplicity, ease-of-use, and quality tooling.

Elm 是一个主打函数式编程，同时通过强大的编译器来尽量确保没有 runtime error 的编程语言，著名的 Redux 就是受它启发。来感受下它的代码：

```elm
import Html exposing (Html, button, div, text)
import Html.Events exposing (onClick)

main =
  Html.beginnerProgram { model = 0, view = view, update = update }

type Msg = Increment | Decrement

update msg model =
  case msg of
    Increment ->
      model + 1

    Decrement ->
      model - 1

view model =
  div []
    [ button [ onClick Decrement ] [ text "-" ]
    , div [] [ text (toString model) ]
    , button [ onClick Increment ] [ text "+" ]
    ]
```

主要分为 4 块，`model` , `view` , `update` , `message` 

* view 展示 model 数据，同时将用户的操作作为 message 抛出。
* model 包含了页面所需的所有信息。
* 当 message 被抛出时，会自动进入到 update 方法，update 返回的新 model 自动进入到 view 里被展示。

![](http://ww1.sinaimg.cn/large/afe37136gy1fgtcussf1ij20e00983yw.jpg)

跟其他的前端框架不同，Elm 不喜欢 parent-child communication, 也不提倡 components，作为函数式编程语言，它在乎的就是创建 function，通过 [helper function](https://guide.elm-lang.org/reuse/) 来达到类似的效果。

### Vue Architecture
![](http://ww1.sinaimg.cn/large/afe37136gy1fgtcv8rgczj20io09wq3l.jpg)

Vue 也是采用的 MVVM 模式，把数据绑定在内部处理了，对外部来说只要在 `data` 里声明特定的 key，在 `view` 里就可以直接使用，并且实时响应。对于 `view`  的事件，也会映射到 `ViewModel` 的特定方法。

Vue 的  `Router` 是把 path 映射到 component 上，看着也比较清晰。

```
const routes = [
  { path: '/foo', component: Foo },
  { path: '/bar', component: Bar },
	{ path: '/user/:id', component: User }
]
```

## The Right Way (IMO)

### 目录结构

![](http://ww1.sinaimg.cn/large/afe37136gy1fgtcvlcjgyj20dw0xg76t.jpg)

目录结构需要能够让不同职责的文件找到自己的归属，同时尽量清晰。这个是我目前觉得还不错的分类

* `External` ：一些第三方的 framework。
* `Extensions` : 针对当前 App 做的一些针对性扩展。
* `Infrastructure` : 比较重要的基础组件，在前期就要管控起来。
* `Models` : 对应服务端的 Objects。
* `Views` : 页面。
* `Shared` : 会在 App 内部被公用的部分，方便统一管控。
* `Utilities` : 一些帮助类。

### Architecture
![](http://ww1.sinaimg.cn/large/afe37136gy1fgtcxzpz4yj218m0mw0vs.jpg)

本质上跟 MVVM 差不多，只是多补充了些细节。之前也有考虑过采用 ReSwift + RxSwift 的方式，也就是 Redux，后来写下来发现还是有点复杂：比如下拉刷新的 3 个 state （ loading / loaded / failed），action 要定义（毕竟获取数据的逻辑写在 Action 中），state 中也要定义（视图最终关心的是 state 的变化）；没有很方便的 diff 支持等。于是就回归到了 MVVM 模式。

#### ViewModel
ViewModel 主要有 3 个职责：

* 通过 Repository 获取/修改数据。
* 提供 `Observable Properties` 供 View 使用。
* 提供 `Functions` 供 View 调用，通常会导致 `Observable Properties` 的改变。

这块也算是常规手法，需要注意的一点是 Repository 的初始化，如果要方便测试的话，最好提供注入点（比如初始化时注入或提供 set 方法注入）。

#### Repository
Repository 的职责就是跟数据打交道，获取远程／本地数据，并将其转换成 Model 返回给 ViewModel。

#### 页面间跳转和通信
使用 Router 即可，如果是内部的 VC 之间跳转，还可以携带 model 信息。

#### 通用的小模块( Components )
我发现前端开发里，`Components` 用得还蛮多的，客户端开发倒不那么常见。这些小模块其实就是一些可在多个页面复用的业务相关的视图（Widget），可能带有业务逻辑，方便复用，比如「赞」按钮。

#### 服务调用
比如在详情页要使用购物车的「加购」功能，通常做法是采用 `Register Procotol` 方式，维护一个 Protocol 和 Class 的注册表，并且在 App 启动时进行注册。我发现使用 Swift 的 POP 就不需要这么麻烦了，具体怎么做，我们后面讲。

### Demo

这个 Demo 演示了知乎日报的列表和详情页：

![](http://ww1.sinaimg.cn/large/afe37136gy1fgtcx5hukwj20lt0ijq3x.jpg)

看起来蛮简单的，不过事实可能并非如此，我们来慢慢捋一下。

#### 初始页

![](http://ww1.sinaimg.cn/large/afe37136gy1fgtcxfxauej20af0ijwek.jpg)

刚进来时，会处于原始的 loading 状态，这个状态不同于下拉刷新，可能是一个萌萌的 loading 图。

首先这个页面属于 `NewsFeed` 页，因此在该目录下新建 3 个文件

```
|- NewsFeedViewModel.swift
|- NewsFeedViewController.swift
|- NewsFeedRepository.swift
```

本着 view 只是展示 state 的原则，我们首先要处理的就是 state，那么怎么处理？ 这个 Event 是从 View 那边触发的，触发之后，对于 View 来说只能求助于 ViewModel，于是 VM 就提供了一个 `initialLoading` 方法。

那这个 `initialLoading` 里该做些什么呢？其实也就是根据 repository 的不同结果，设置不同的 state，然后 view 来响应这些 state。同时考虑到之后的「下拉刷新」和「加载更多」，顺便分离出一个通用的 `loadData:` 方法

#### ViewModel

```swift
class NewsFeedViewModel {
	func initialLoading() {
        loadData(.initial)
    }

	func loadData(_ loadingType: LoadingType, offset: String = "") {
		// todo
	}
}
```

那么 `Observable Properties` 应该是怎样的呢？在 OC 时代，只要简单的暴露 readonly 的 property，外部无论是 KVO 还是 RAC 都能很方便地进行绑定，到了 swift 时代，如果要做 KVO 就要继承 `NSObject`，还要加一个 `@dynamic`  前缀，不优雅。比较理想的状态是使用 RxSwift 的 `Observable` 作为属性，外部只要 `subscribe` 就行了。不过在内部如何给这个 `Observable`  塞数据又有点小问题。最终决定使用 `Variable` 作为暴露的属性，它的好处是内部不需要再新建一个变量，直接设置这个 `Variable` 的 `value` 即可，弊端就是对于使用方需要先通过 `asObservable()` 转一下再进行 subscribe，并且只要愿意，也可以设置 `value` 值，存在误操作的风险。在这里我们先简单起见用 `Variable` 来做。

接下来的问题就是这个 `Variable` 里应该放什么？肯定要放一些当前的 loading 状态，比如 loaded，failed，loading 这些，那么要不要带上 data？如果不一起带上 data，那么状态的改变和数据的改变就不是一个原子操作，有可能会带来一些异常（比如 view 发现 loading 状态变为 loaded，自动去取最新的 data，但此时 data 可能还没有改变）。因此，我把它们都放到了一起，首先来看一下 `ResultModel` 

#### Model
这是一个通用的数据结构

```swift
// ResultModel.swift

enum LoadingType {
    case initial, refresh, more
}

enum LoadingStatus: Equatable {
    case none
    case loading
    case loaded
    case failure(Error)
    
    static func ==(lhs: LoadingStatus, rhs: LoadingStatus) -> Bool {
        switch (lhs, rhs) {
        case (.none, .none):
            return true
        case (.loading, .loading):
            return true
        case (.loaded, .loaded):
            return true
        default:
            return false
        }
    }
}

struct ResultModel<T> {
    var loadingStatus: LoadingStatus = .none
    var loadingType: LoadingType = .initial
    
    var previousItems = [T]()
    var currentItems = [T]()
}
```

```swift
// NewsModel.swift

class NewsFeedViewModel {
    // 1
	  static var news:Variable<ResultModel<NewsItem>> = Variable(ResultModel())

    func initialLoading() {
        loadData(.initial)
    }

    func loadData(_ loadingType: LoadingType, offset: String = "") {
        // 2 如果当前处于 loading 状态，就不继续处理了
        if (NewsFeedViewModel.news.value.loadingStatus == .loading) {
            return
        }

        // 3 设置新的 loading 类型和状态
        var value = NewsFeedViewModel.news.value
        value.loadingStatus = .loading
        value.loadingType = loadingType
        NewsFeedViewModel.news.value = value
        
        // 4 接下来就是发网络请求，根据不同的请求结果设置 state
    }
}
```

1. 这里使用 `static` 主要是出于方便。
2. 这里纠结了一段时间，之前是新建了 3 个 loading status（initial, refresh, loadmore），然后每个 status 再细分为 3 种状态(loading, loaded, error)，后来发现这样的话，「当前是哪个 loading status，该 status 目前处于什么状态」判断起来会比较麻烦。于是就按照现在这样进行了拆分。
3. 在这里对状态进行更改之后，UI 那边可以自动收到更新。
4. 这里会调用 Repository 来获取数据。

#### Repository
Repository 这块由于是异步交互，因此直接就上 RxSwift 了，返回一个 `Observable` ，VM 作为消费方来订阅。

```swift
import Foundation
import RxSwift

class NewsFeedRepository {
    static func news(_ offset: String = "") -> Observable<[String:Any]?> {
        return Observable.create({ observer in
            let path = offset.characters.count > 0 ? "/api/4/news/before/\(offset)" : "/api/4/news/latest"
            let resource = Resource(path: path, method: .GET, requestBody: nil, headers: ["Content-Type": "application/json"], parse: decodeJSON)
            
            // 这个用的是 chris 开源的简单的 API 请求封装 http://chris.eidhof.nl/posts/tiny-networking-in-swift.html
            apiRequest(baseURL: URL(string: "https://news-at.zhihu.com")!, resource: resource, failure: { (reason, result) in
                observer.on(.error(reason))
            }, success: { result in
                observer.on(.next(result))
                observer.on(.completed)
            })
            
            return Disposables.create()
        })
    }
}
```

也可以在这里直接返回解析后的 Model，这样 VM 那边就不用处理了。

#### ViewModel 调用 Repository

```swift
class NewsFeedViewModel {
    // 4
    NewsFeedRepository.news(offset).asObservable().subscribe(onNext: {[unowned self] (result) in
        // 把 json 转换为 model
        let parsedResult = self._parseResult(result: result)
        var value = NewsFeedViewModel.news.value
        value.previousItems = NewsFeedViewModel.news.value.currentItems
        
        // 设置对应的 value
        if value.loadingType == .more {
            value.currentItems = value.previousItems + (parsedResult?.news ?? [])
        } else {
            value.currentItems = parsedResult?.news ?? []
        }
            
        value.loadingStatus = .loaded
        NewsFeedViewModel.news.value = value
        self.offset = parsedResult?.date ?? ""
        value.loadingStatus = .none
        
        // 统一设置 value，对外部 subscriber 来说就是原子操作
        NewsFeedViewModel.news.value = value
    }, onError: { (error) in
        NewsFeedViewModel.news.value.loadingStatus = .failure(error)
    }, onCompleted: {  
    }) {
    }.addDisposableTo(disposeBag)
}
```

这里你会注意到有一个 `previousItems` 和 `currentItems` ，这个主要是提供灵活性，避免暴力的 `reloadData()` ，比如获取到了更多的数据之后，可以只 reload 新的数据。

#### View

```swift
// NewsFeedViewController.swift
class NewsFeedViewController: UITableViewController {
    override func viewDidLoad() {
        handleDataChange()
        viewModel.initialLoading()
    }

    func handleDataChange() {
        NewsFeedViewModel.news.asObservable()
            .observeOn(MainScheduler.instance)
            .subscribe(onNext: {[unowned self] item in
                if item.loadingStatus != .loading {
                    self.initialLoadingIndicator.stopAnimating()
                }
                if item.loadingStatus == .loaded {
                    // 这里调用 Diff 这个 framework 提供的 extension
                    self.tableView.animateRowChanges(oldData: item.previousItems, newData: item.currentItems)
                }
                if item.loadingType == .initial && item.loadingStatus == .loading {
                    self.initialLoadingIndicator.startAnimating()
                }
            }).addDisposableTo(disposeBag)
    }
}
```

「正在加载」和「已经加载」的场景已经处理完了，「加载失败」的处理也类似，比如失败之后显示一个 reload button，点击 reload button 之后，再调用一下 `viewModel.initialLoading()`

#### TableView
接下来就来看看如何处理 TableView 的数据展示，其实就是消费 VM 的 property

```swift
extension NewsFeedViewController {
    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return NewsFeedViewModel.news.value.currentItems.count
    }
    
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "Cell") as! NewsCell
        let newsItem: NewsItem = NewsFeedViewModel.news.value.currentItems[indexPath.row]
        cell.configure(newsItem)
        return cell
    }
}
```

到这里最基本的首页数据展示就基本完成了。

### 加载更多
之前一直在纠结这块到底该怎么做才比较合适，如果直接把 newItems append 到原有的 items 列表，形成新的列表，UI 那边拿到之后就只能 `reloadData()` 了，最好能让 UI 那边知道新的和旧的之间发生了哪些变化，于是就找到了 [Diff](https://github.com/wokalski/Diff.swift) 这个 framework，它能够定位出两个 collection 之间的差异，但前提是 collection item 要实现 `Equatable` 协议。于是就有了 `previousItems` 和 `currentItems` 的设计。

### 喜欢功能
喜欢功能本质上是修改 NewsItem 的 `hasFaved` 属性，然后让 UI 可以感知到这个变化。这里问题就来了：如何对列表中的一个 `struct` 进行调整？我们知道 `struct` 是值拷贝的，只要发生赋值行为，拿到的就不再是原先的那个 struct 了（比如把 items 通过参数传递，要修改的话就要进行拷贝，除非设置为 `inout`）。

这个问题本质上是如何操作 Immutable Objects，然后就想到了 [Immutable.js](https://facebook.github.io/immutable-js/)，它也提供了一些修改 List 的方法，只不过都是返回一个新的：

```js
const { List } = require('immutable');
const list = List([ 0, 1, 2, List([ 3, 4 ])])
list.setIn([3, 0], 999);
// List [ 0, 1, 2, List [ 999, 4 ] ]
```

因此，这里简单的处理方式就是通过传进来的 `newsItem` 找到它在 list 中的 index（`newsItem` 已经实现了 `Equatable` 协议），然后把修改过 `hasFaved` 属性的新的 `newsItem` 放到 index 位置来达到替换的效果。

```swift
class NewsFeedViewModel {
    func toggleFav(_ newsItem: NewsItem) {
        if let newsIndex = NewsFeedViewModel.news.value.currentItems.index(of: newsItem) {
            var _newsItem = NewsFeedViewModel.news.value.currentItems[newsIndex]
            _newsItem.hasFaved = !_newsItem.hasFaved

            var value = NewsFeedViewModel.news.value
            value.currentItems[newsIndex] = _newsItem

            NewsFeedViewModel.news.value = value
        }
    }
}
```

#### Components
由于新闻列表和喜欢的新闻列表表现上一致，那么就可以进行一些复用，比如可以把 Cell 作为 Component。

那对于一个 Component 来说，需要具备哪些特性呢？这个并没有什么约定，本质上就是一个或几个函数，外部调用后会返回一个 view，或者提供一些 block 回调，仅此而已。

#### Truth and Computed Properties 
这里的 `Truth` 是指最源头的数据，比如一个数组，`Computed Properties` 是指对源头数据进行消费可以得到的结果，比如数组的长度，或数组中的正数等。

在这个例子中，`Truth` 就是 `newsItems` 列表，而喜欢的 `newsItems` 就是 `Computed Properties` 。因此只要 newsItems 发生变化，就重新计算喜欢的 NewsItems。

```swift
NewsFeedViewModel.news.asObservable().subscribe(onNext: { item in
    NewsFeedViewModel.favedNews.value = NewsFeedViewModel.news.value.currentItems.filter { (item) -> Bool in
         return item.hasFaved
    }
}).addDisposableTo(disposeBag)
```

#### 喜欢功能的 View
主要就是两件事：

1. 点击 Fav 按钮时，调用 VM 的 `toggleFav` 方法。
2. 当 Fav 列表更新时，刷新 TableView。

```swift
extension FavedViewController {
    func handleDataChange() {
        NewsFeedViewModel.favedNews.asObservable().subscribe(onNext:{[unowned self] item in
            self.tableView.reloadData()
        }).addDisposableTo(disposeBag)
    }
}

extension FavedViewController {
    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return NewsFeedViewModel.favedNews.value.count
    }
    
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "Cell") as! NewsCell
        var newsItem: NewsItem = NewsFeedViewModel.favedNews.value[indexPath.row]
        
        cell.configure(newsItem) { [unowned self] (button) in
            if button.tag == 0 {
                button.tag = 1
                button.setTitle("♥︎", for: .normal)
            } else {
                button.tag = 0
                button.setTitle("♡", for: .normal)
            }
            self.viewModel.toggleFav(newsItem)
            self.tableView.reloadData()
        }
        
        return cell
    }
}
```

### 页面跳转
页面间的跳转用到了 `Router` ，也就是 open 一个 url 就能到达特定的页面，这么做的好处是可以和外部跳转进来的情况统一处理（因为从外部跳到某个 app 只能通过 openURL）。

但在内部直接输入 URL 总觉得不优雅，而且容易出错，将来如果要修改 URL 也不方便。因此做了一个简单的 `Router` 来达到这个效果：

```swift

import Foundation
import UIKit

// 1
enum RouterTable: String {
    case home = "home"
    case detail = "detail/:id"
    
    func asController() -> UIViewController.Type {
        switch self {
        case .home:
            return NewsFeedViewController.self
        case .detail:
            return NewsDetailViewController.self
        }
    }
}

// 2
class Router {
    static func to(_ route: RouterTable, parameters: Dictionary<String, Any>?) -> Void {
        let viewController = route.asController().init()

        // 2.1
        if let parameters = parameters {
            for (key, value) in parameters {
                viewController.putExtra(key, value)
            }
        }

        //TODO: 添加 shouldBePushed 调用，比如有些页面需要先登录
        DispatchQueue.main.async {
            UINavigationController.current().pushViewController(viewController, animated: true)
        }
    }
}

// 3
extension Router {
    func parseURL(_ url: String) -> (RouterTable, Dictionary<String, String>?) {
        //TODO: add implementation
        return (.home, nil)
    }
}
```

主要分为 3 部分：

1. 这个跟 vue-router 里定义 url 和 components 的关系一样，主要是为了方便统一管理。
2. 这里主要是把 enum 转换为对应的 Controller，因为限制了类型，也就不会出现找不到 VC 的情况。
3. 这个是用来应对外部跳转进来的 URL，把它解析成 `RouterTable`，统一逻辑。

针对 2 重点说一下，这个是最简实现，真实场景会比这复杂得多，比如有些页面是 present 出来的，有些页面 push 前需要先判断是否登录等等。

注意到 `2.1` 的部分，这里有一个 `putExtra` 方法，这是新添加的一个扩展，参考了 Android 的 `Intent`  `putExtra` 。实现如下：

```swift
protocol ViewCotrollerIntent {
    func putExtra(_ key: String, _ value: Any)
    func getExtra(_ key: String) -> Any?
}

extension UIViewController: ViewCotrollerIntent {
    
    private struct IntentStorage {
        static var extra: [String:Any] = [:]
    }
    
    func putExtra(_ key: String, _ value: Any) {
        IntentStorage.extra[key] = value
    }
    
    func getExtra(_ key: String) -> Any? {
        return IntentStorage.extra[key]
    }
}
``` 

由于 extension 不支持 associated properties，因此用 struct 做了个中转。这样，VC 之间的跳转如果要带上额外的参数，只要放到 extra 里即可。

### 详情页
详情页比较简单，只是展示一个 webview，这里比较棘手的问题是 model 数据的同步。由于详情页也可以修改 `NewsItem` 的 `hasFaved` 属性，这个改变需要能够实时同步到列表页，不然就会出现状态不同步的情况。

这块的设计也想了一段时间，Pinterest 采用的是[通知的方式](https://medium.com/@Pinterest_Engineering/immutable-models-and-data-consistency-in-our-ios-app-d10e248bfef8)，并且额外开发了一个用来支持这种方式的[库](https://github.com/pinterest/plank)，不想整的这么麻烦。本质需求是：当传过去的 model 发生变化时通知我。而 RxSwift 里的 `Variable` 不是正好可以达到这个效果么？于是就有了基于 `Variable` 的解决方案。

```swift
extension NewsFeedViewController {
    override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let newsItem: NewsItem = NewsFeedViewModel.news.value.currentItems[indexPath.row]
        let newsItemVariable = Variable<NewsItem>(newsItem)

        // 详情页可能会对这个 newsItemVariable 进行调整
        newsItemVariable.asObservable().subscribe(onNext: { [unowned self] item in
            // 找到这个 item 所在的 index，并进行替换
            self.viewModel.update(item: item)
            self.tableView.reloadData()
        }).addDisposableTo(disposeBag)

        // 带上这个 Variable 到新的 VC
        Router.to(.detail, parameters: ["model": newsItemVariable])
    }
}
```

#### 详情页 View 的处理

```swift
class NewsDetailViewController: UIViewController {
    override func viewDidLoad() {
        // favButton
        navigationItem.rightBarButtonItem = favButton
        favButton.rx.tap
            .subscribe(onNext: { [unowned self] item in
                self.viewModel.toggleFav()
            })
            .addDisposableTo(disposeBag)
        
        // 1
        if let id = self.getExtra("id") as? Int {
            // viewModel.load(id)
        }
        
        // 2
        if let model = self.getExtra("model") as? Variable<NewsItem> {
            favButton.title = model.value.hasFaved ? "♥︎" : "♡"
            viewModel.load(Int(model.value.id))
            NewsDetailViewModel.newsItem = model
        }
        
        handleDataChange()
    }

    // 3
    func handleDataChange() {
        NewsDetailViewModel.newsDetail.asObservable()
            .subscribe(onNext:{ [unowned self] item in
                if let item = item {
                    let request = URLRequest(url: URL(string: item.shareURL)!)
                    self.webView.loadRequest(request)
                }
            })
            .addDisposableTo(disposeBag)
        
        NewsDetailViewModel.newsItem?.asObservable()
            .subscribe(onNext: { [unowned self] item in
                self.favButton.title = item.hasFaved ? "♥︎" : "♡"
            })
            .addDisposableTo(disposeBag)
    }
}
```

1. 这里为通过外部 URL 进来的留一个入口。
2. 通过 `getExtra` 拿到 `Variable` 后，接下来就交给 VM 了。
3. `handleDataChange` 做的事情就是响应 VM 的 properties 的变化，做一些 UI 上的调整。

### Service
之前说过使用 Swift 提供 Service 会比较方便，都不需要在 App 启动时进行注册，利用自带的 Protocol Extension 就能达到效果。这个例子中没有用到，就举个其他的例子吧，以购物车为例：

```swift
// 放在 Services 目录下的 Protocols.swift
protocol Cart {
    public func add(_ item: Item) -> Bool
}

// 具体的实现可以放到对应的页面
extension Cart {
    public func add(_ item: Item) -> Bool {
        // business logic
    }
}
```

对于想要使用这个功能的开发来说，只要看 `Services/Protocols.swift` 就行了。跟 Objective-C 不同，extension 里如果有两个相同的方法，编译器会直接报错，这样就避免了运行期可能出现多个实现的问题。

### Local Reasoning
Local Reasoning 的意思是对于数据的改动都发生在某一个特定的单元。这也是使用 Value Type 的好处，因为如果使用 Reference Type，只要把其中的一个 Reference 给了出去，就不知道什么时间什么场景下数据会在外部被改变，就像给了你一张银行卡，今天看还剩 1 万，可能明天再去看就只剩 1 千了。

使用 VM 后，所有对数据的改动都发生在 VM 里面，同时对数据的消费也尽量在一个地方，方便维护。

## 小结
以上是我自己对「Right Architecture」的一些理解和实践，实际过程中肯定还有很多细节要调整，如果你有什么想法欢迎交流～

