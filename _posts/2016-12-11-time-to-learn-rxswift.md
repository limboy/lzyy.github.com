---
layout: post
title: 是时候学习 RxSwift 了
category: tech
tags: 技术
---

相信在过去的一段时间里，对 RxSwift 多少有过接触或耳闻，或者已经积累了不少实战经验。此文主要针对那些在门口徘徊，想进又拍踩坑的同学。

### 为什么要学习 RxSwift
当决定做一件事情时，至少要知道为什么。RxSwift 官网举了[几个例子](https://github.com/ReactiveX/RxSwift/blob/master/Documentation/Why.md)，比如可以统一处理 `Delegate`, `KVO`, `Notification`，可以绑定 UI，方便网络请求的处理等等。但这些更多的是描述可以用 RxSwift 来做什么，跟为什么要使用 RxSwift 还是会有点不同。

我们先来分析下 GUI 编程的本质，我喜欢把它抽象为视图和数据的结合。其中视图负责两件事：展示和交互，展示什么由数据决定。

![](/image/14814474678383.jpg)


其中单向数据流可以通过[之前介绍的 ReSwift](http://limboy.me/tech/2016/12/04/reswift-analyze.html) 完成。看起来好像没 RxSwift 什么事情，其实不然，RxSwift 可以在 UniDirectional Data Flow 的各个阶段都发挥作用，从而让 Data 的处理和流动更加简洁和清晰。

![](/image/14814474032179.jpg)

1. 通过对 RxCocoa 的各种回调进行统一处理，方便了「Interact」的处理。
2. 通过对 `Observable` 的 transform 和 composite，方便了 `Action` 的生成（比如使用 `throttle` 来压缩 `Action`）。
3. 通过对网络请求以及其他异步数据的获取进行 `Observable` 封装，方便了异步数据的处理。
4. 通过 RxCocoa 的 binding，方便了数据的渲染。

所以 ReSwift 规范了数据流，RxSwift 为数据的处理提供了方便，这两个类库的结合，可以产生清晰的架构和易维护的代码。

当然，前提是对它们有足够的了解，尤其是 RxSwift，也就是我们今天的主角。

### 什么是 RxSwift
在 GUI 编程中，我认为比较复杂的有三个部分：

1. 非原生 UI 效果的实现（比如产品经理们经常冒出来的各种想法）。
2. 大量状态的维护。
3. 异步数据的处理。

1）不在这次的讨论范畴（这里的学问也很多，比如流畅性和性能）。2) 可以通过单向数据流来解决（结合 Immutable Data）。3) 可以通过 RxSwift 来解决。那么 RxSwift 是如何处理异步数据的呢？

在说 RxSwift 之前，先来说下 Rx， [ReactiveX](http://reactivex.io/) 是一种编程模型，最初由微软开发，结合了观察者模式、迭代器模式和函数式编程的精华，来更方便地处理异步数据流。其中最重要的一个概念是 `Observable`。

举个简单的例子，当别人在跟你说话时，你就是那个观察者，别人就是那个 `Observable`，它有几个特点：

1. 可能会不断地跟你说话。（`onNext:`）
2. 可能会说错话。（`onError:`）
3. 结束会说话。（`onCompleted`）

你在听到对方说的话后，也可以有几种反应：

1. 根据说的话，做相应的事，比如对方让你借本书给他。（`subscribe`）
2. 把对方说的话，加工下再传达给其他人，比如对方说小张好像不太舒服，你传达给其他人时就变成了小张失恋了。（`map:`）
3. 参考其他人说的话再做处理，比如 A 说某家店很好吃，B 说某家店一般般，你需要结合两个人的意见再做定夺。（`zip:`）

所以，从生活中也能看到 Rx 的影子。「有些事情急不得，你得等它自己熟」，异步，其实就是跟时间打交道，不同的时间，拿到的数据也会不一样。可以[在线感受下](http://rxmarbles.com)

![](/image/14814518766811.jpg)

这里的核心是当数据有变化时，能够立刻知晓，并且通过组合和转换后，可以即时作出响应。有点像塔防，先在路上的各个节点埋好武器，然后等着小怪兽们过来。

### RxSwift Workflow

大致分为这么几个阶段：先把 Native Object 变成 Observable，再通过 Observable 内置的各种强大的转换和组合能力变成新的 Observable，最后消费新的 Observable 的数据。

![](/image/14814540314644.jpg)

#### Native Object -> Observable

##### .rx extension

假设需要处理点击事件，正常的做法是给 Tap Gesture 添加一个 Target-Action，然后在那里实现具体的逻辑，这样的问题在于需要重新取名字，而且丢失了上下文。RxSwift (确切说是 RxCocoa) 给系统的诸多原生控件（包括像 `URLSession`）提供了 rx 扩展，所以点击的处理就变成了这样：

```swift
let tapBackground = UITapGestureRecognizer()

tapBackground.rx.event
    .subscribe(onNext: { [weak self] _ in
        self?.view.endEditing(true)
    })
    .addDisposableTo(disposeBag)
    
view.addGestureRecognizer(tapBackground)
```

是不是简洁了很多。

##### Observable.create

通过这个方法，可以将 Native 的 object 包装成 `Observable`，比如对网络请求的封装：

```swift
public func response(_ request: URLRequest) -> Observable<(Data, HTTPURLResponse)> {
	return Observable.create { observer in
		let task = self.dataTaskWithRequest(request) { (data, response, error) in
			observer.on(.next(data, httpResponse))
			observer.on(.completed)
		}

		task.resume()

		return Disposables.create {
			task.cancel()
		}
	}
}
```

出于代码的简洁，略去了对 error 的处理，使用姿势类似

```swift
let disposeBag = DisposeBag()

response(aRequest)
  .subscribe(onNext: { data in
    print(data)
  })
  .addDisposableTo(disposeBag)
```

这里有两个注意点：

1. `Observerable` 返回的是一个 `Disposable`，表示「可扔掉」的，扔哪里呢，就扔到刚刚创建的袋子里，这样当袋子被回收（`dealloc`）时，会顺便执行一下 `Disposable.dispose()`，之前创建 `Disposable` 时申请的资源就会被一并释放掉。
2. 如果有多个 subscriber 来 subscribe `response(aRequest)` 那么会创建多个请求，从代码也可以看得出来，来一个 observer 就创建一个 task，然后执行。这很有可能不是我们想要的，如何让多个 subscriber 共享一个结果，这个后面会提到。

##### Variable()

`Variable(value)` 可以把 value 变成一个 `Observable`，不过前提是使用新的赋值方式 `aVariable.value = newValue`，来看个 Demo

```swift
let magicNumber = 42

let magicNumberVariable = Variable(magicNumber)
magicNumberVariable.asObservable().subscribe(onNext: {
    print("magic number is \($0)")
})

magicNumberVariable.value = 73

// output
// 
// magic number is 42
// magic number is 73
```

起初看到时，觉得还蛮神奇的，跟进去看了下，发现是通过 `subject` 来做的，大意是把 `value` 存到一个内部变量 `_value` 里，当调用 `value` 方法时，先更新 `_value` 值，然后调用内部的 `_subject.on(.next(newValue))` 方法告知 subscriber。

##### Subject

`Subject` 简单来说是一个可以主动发射数据的 `Observable`，多了 `onNext(value)`, `onError(error)`, 'onCompleted' 方法，可谓全能型选手。

```swift
let disposeBag = DisposeBag()
let subject = PublishSubject<String>()
    
subject.addObserver("1").addDisposableTo(disposeBag)
subject.onNext("🐶")
subject.onNext("🐱")
    
subject.addObserver("2").addDisposableTo(disposeBag)
subject.onNext("🅰️")
subject.onNext("🅱️")
```

记得在 RAC 时代，subject 是一个不太推荐使用的功能，因为过于强大了，容易失控。RxSwift 里倒是没有太提及，但还是少用为佳。

#### Observable -> New Observable
`Observable` 的强大不仅在于它能实时更新 value，还在于它能被修改／过滤／组合等，这样就能随心所欲地构造自己想要的数据，还不用担心数据发生变化了却不知道的情况。

##### Combine
Combine 就是把多个 `Observable` 组合起来使用，比如 `zip` (小提示：如果对这些函数不太敏感，可以[实际操作下](http://rxmarbles.com/)，体会会更深些)

`zip` 对应现实中的例子就是拉链，拉链需要两个元素这样才能拉上去，这里也一样，只有当两个 `Observable` 都有了新的值时，subscribe 才会被触发。

```swift
let stringSubject = PublishSubject<String>()
let intSubject = PublishSubject<Int>()

Observable.zip(stringSubject, intSubject) { stringElement, intElement in
	"\(stringElement) \(intElement)"
	}
	.subscribe(onNext: { print($0) })
	.addDisposableTo(disposeBag)

stringSubject.onNext("🅰️")
stringSubject.onNext("🅱️")

intSubject.onNext(1)
intSubject.onNext(2)

// output
//
// 🅰️ 1
// 🅱️ 2
```

如果这里 `intSubject` 始终没有执行 `onNext`，那么将不会有输出，就像拉链掉了一边的链子就拉不上了。

除了 `zip`，还有其他的 combine 的姿势，比如 `combineLatest` / `switchLatest` 等。

##### Transform
这是最常见的操作了，对一个 `Observable` 的数值做一些小改动，然后产出新的值，依旧是一个 `Observable`。

```swift
let disposeBag = DisposeBag()
Observable.of(1, 2, 3)
    .map { $0 * $0 }
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

这是大致的实现（摘自官网）

```swift
extension ObservableType {
    func myMap<R>(transform: E -> R) -> Observable<R> {
        return Observable.create { observer in
            let subscription = self.subscribe { e in
                    switch e {
                    case .next(let value):
                        let result = transform(value)
                        observer.on(.next(result))
                    case .error(let error):
                        observer.on(.error(error))
                    case .completed:
                        observer.on(.completed)
                    }
                }

            return subscription
        }
    }
}
```

接受一个 transform 闭包，然后返回一个 `Observable`，因为接下来使用者将会对 `myMap` 的结果进行 subscribe，所以需要在 create 内部 subscribe 一下，不然最开始的那个 `Observable` 就是个 `Cold Observable`，一个 `Cold Observable` 是不会产生新的数据的。

##### Filter
Filter 的作用是对 `Observable` 传过来的数据进行过滤，只有符合条件的才有资格被 subscribe。写法上跟 map 差不多，就不赘述了。

##### Connect
这是挺有意思的一块，在之前介绍 `Observable.create` 时有提到过，一个 `Observable` 被多次 subscribe 就会被多次触发，如果一个网络请求只想被触发一次，同时支持多个 subscriber，就可以使用 `publish` + `connect` 的组合。

当一个 `Observable` 使用了 `publish()` 方法后，正常的 subscribe 就不会触发它了，除非 `connect()` 方法被调用。而且每次 subscribe 不会导致 `Observable` 重新针对 observer 处理一遍。看一下这张图

![](http://reactivex.io/documentation/operators/images/publishConnect.c.png)

有两块需要注意：

1. `connect()` 之前的两次 `subscribe` 并没有产生新的 value。
2. `connect()` 之后 `subscribe` 的，只是等待新的 value，同时新的 value 还会分发给之前的 subscriber。
3. 即使所有的 `subscription` 被 `dispose`, `Observable` 依旧处于 `hot` 状态，就好像还以为有人关心新的值一样。（这可能不是想要的结果）

针对第 3 点，可以使用 `refcount()` 来代替 `connect()`，前者会在没有 subscriber 时自动「冷」下来，不会再产生新的值。（Demo 取自[这里](http://www.tailec.com/blog/understanding-publish-connect-refcount-share)）

```swift
let myObservable = Observable<Int>.interval(1, scheduler: MainScheduler.instance).publish().refCount() // 1)

let mySubscription = myObservable.subscribe(onNext: {
    print("Next: \($0)")
})

delay(3) {
    print("Disposing at 3 seconds")
    mySubscription.dispose()
}

delay(6) {
    print("Subscribing again at 6 seconds")
    myObservable.subscribe(onNext: {
        print("Next: \($0)")
    })
}
```

输出

```
Starting at 0 seconds
Next: 0
Next: 1
Next: 2
Disposing at 3 seconds
Subscribing again at 6 seconds
Next: 0
Next: 1
```

可以看到，3 秒后 subscription dispose，此时没有任何 subscriber 还关心 `Observable`，因此就重置了，所以 6 秒后又回到了初始状态（如果变成 `connect` 方法的话，会发现 6 秒后会输出 `Next: 6 / Next: 7`）

那如果后加入的 subscriber 想要之前的数据怎么办？可以对原始的 `Observable` 设置 `replay(n)`，表示最多返回 n 个元素给后加入的 subscriber。

### Tips
上面介绍的是最基本的概念。顺便提一下比较常见的几个问题：

#### 如何处理 Scheduler？

默认代码都是在当前线程中执行的，如果要手动切换线程，可以使用 `subsribeOn` 和 `observeOn` 两种方式，一般来说后者用得会多一些，那这两者有什么区别呢？

`subscribeOn` 跟位置无关，也就是无论在链式调用的什么地方，`Observable` 和 `subscription` 都会受影响；而 `observeOn` 则仅对之后的调用产生影响，看个 Demo：

```swift
var observable = Observable<Int>.create { (observer: AnyObserver<Int>) -> Disposable in
    print("observable thread: \(Thread.current)")
    observer.onNext(1)
    observer.onCompleted()
    return Disposables.create()
}

let disposeBag = DisposeBag()

observable
    .map({ (e) -> Int in
        print("map1 thread: \(Thread.current)")
        return e + 1
    })
    .observeOn(ConcurrentDispatchQueueScheduler(qos: .userInteractive)) // 1
    .map({ (e) -> Int in
        print("map2 thread: \(Thread.current)")
        return e + 2
    })
    .subscribe(onNext:{ (e) -> Void in
        print("subscribe thread: \(Thread.current)")
    })
    .addDisposableTo(disposeBag)
```

如果 1) 是 `observeOn`，那么输出如下

```
observable thread: <NSThread: 0x7f901cc0d510>{number = 1, name = main}
map1 thread: <NSThread: 0x7f901cc0d510>{number = 1, name = main}
map2 thread: <NSThread: 0x7f901ce15560>{number = 3, name = (null)}
subscribe thread: <NSThread: 0x7f901ce15560>{number = 3, name = (null)}
```

可以看到 observable thread 和 map1 thread 依旧保持当前线程，但 `observeOn` 之后就变成了另一个线程。

如果 1) 是 `subscribeOn`，那么会输出

```
observable thread: <NSThread: 0x7fbdf1e097a0>{number = 3, name = (null)}
map1 thread: <NSThread: 0x7fbdf1e097a0>{number = 3, name = (null)}
map2 thread: <NSThread: 0x7fbdf1e097a0>{number = 3, name = (null)}
subscribe thread: <NSThread: 0x7fbdf1e097a0>{number = 3, name = (null)}
```

可以看到全都变成了 `subscribeOn` 指定的 Queue。所以 `subscribeOn` 的感染力很强，连 `Observable` 都能影响到。

#### Cold Observable 和 Hot Observable

Cold 相当于 InActive，就像西部世界里，未被激活的机器人一样；Hot 就是处于工作状态的机器人。

#### Subscription 为什么要 Dispose？

因为有了 `Subscriber` 所以 `Observable` 被激活，然后内部就会使用各种变量来保存资源，如果不 `dispose` 的话，这些资源就会一直被 keep，很容易造成内存泄漏。

同时手动 dispose 又嫌麻烦，所以就有了 `DisposeBag`，当这个 Bag 被回收时，Bag 里面的 subscription 会自动被 dispose，相当于从 MRC 变成了 ARC。

### 小结
RxSwift 如果概念上整理清楚了，会发现其实并不难，多从 `Observable` 的角度去思考问题，多想着转换和组合，慢慢就会从命令式编程转到声明式编程，对于抽象能力和代码的可读性都会有提升。

