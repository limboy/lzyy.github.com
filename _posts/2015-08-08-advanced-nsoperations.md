---
layout: post
title: Advanced NSOperations
category: tech
tag: 技术
---

### 前言

这篇文章是对 [WWDC 2015 Session 226: Advanced NSOperations](https://developer.apple.com/videos/wwdc/2015/?id=226) 的一个小结，在那个视频中，[Dave DeLong](https://twitter.com/davedelong) 分享了 NSOperation 的高级玩法，WWDC App 就是基于这套玩法做的，还是挺开阔思路的。

### NSOperation 和 NSOperationQueue 简介

我们知道 NSOperation 可以执行一些后台操作，如 HTTP 请求，在 iOS 4.0 之前是基于 NSThread 来实现的，iOS 4.0 带了 GCD，NSOperation 底层也基于 GCD 重写了底层实现。

所以 NSOperation 是 GCD 的高层封装，同时也带来了一些更加便利的功能，比如取消任务，设置依赖等。在进入高级玩法前，先简单的介绍下 NSOperation 和 NSOperationQueue。

#### NSOperationQueue maxConcurrentOperationCount

这个属性表示的是 NSOperationQueue 最多可以同时处理几个任务，假如我们希望它一次只处理一个，也就是线性 Queue，可以设置 `maxConcurrentOperationCount = 1`

![](/image/nsoperation-1.png)

中间的点表示任务的状态，在上一个任务完成前，下一个任务不会被执行，因为只有一个 worker。

如果希望一次能处理多个，将这个值设置为大于 1 即可，或者直接使用默认值，系统会自动设置一个合理的最大值。

![](/image/nsoperation-2.png)

#### NSOperation cancel

从上面的图可以看到，正在被执行的任务的状态跟在后面排队的状态是不一样的，有这么几种状态：pending, ready, executing, finished, cancelled。

![](/image/nsoperation-3.png)

之前提到过 NSOperation 一个很重要的特性是可以被取消，但不同状态的取消处理也不一样。比如当 Operation 处于 pending, ready 状态时，系统可以去看一下这个 Operation 是否已经被取消了(判断 self.cancelled)，如果是的话，就不执行任务了。但是当 Operation 处于 executing 状态时，取消的操作就只能自己处理了，比如

```objc
@implementation MyOperation: NSOperation
- (void)main
{
    // ...
    while (!self.cancelled) {
        // executing
    }
}
@end
```

#### NSOperation dependency

NSOperation 还有一个很重要的特性是可以设置依赖

![](/image/nsoperation-4.png)

任务 A 需要等待 任务 B 和 任务 C 完成，才能被执行，而任务 B 需要等到 任务 D 完成才能被执行。

当然前提是这些 Operation 都需要被放到某个 Queue 里，这样它们的状态才会发生改变。

### 高级玩法

开发 App 的过程中，有一些逻辑是可以共用的，比如登录、网络状况等，最好可以组装起来，就像超能陆战队里的 megabot 一样

![](/image/megabot.jpg)

基于前面提到的 NSOperation / NSOperationQueue 的一些特点，苹果的工程师们想到了他们的解决方法。

#### Condition

Condition，也就是条件，它可以被附加到 Operation 上，只有当 Condition 被满足时，Operation 才能被执行。比如只有在有网络的情况下才能进行交易，这时「网络状况」就是附加给「交易」的 Condition。

一个 Condition 主要包含了 3 个方法：

```swift
// 1
static var isMutuallyExclusive: Bool { get }
// 2
func dependencyForOperation(operation: Operation) -> NSOperation?
// 3
func evaluateForOperation(operation: Operation, completion: OperationConditionResult -> Void)
```

1. 这个属性用来表明这个 Condtion 是否是排他的，如果是的话，同一时间只能出现一个该类型的实例，类型的指定是通过设置 `name` 来实现的。
2. 为传入的 operation 返回一个依赖的 operation，比如「喜欢」这个 Operation 需要用户已处于登录状态，那么「登录」这个 Condition 的这个方法就可以返回一个「登录」的 Operation。
3. 这个方法是查看这个 Condition 的执行结果，比如前面的「登录」Operation 结束后，系统将要执行「喜欢」这个 Operation，然后这个方法就会被触发，如果没有错误发生的话，就执行「喜欢」，如果有错误发生「喜欢」就会自动结束。

所以总结起来 Condition 主要干了这么三件事

![](/image/nsoperation-condition.png)

来看一个简单的 Condition (来自 WWDC Sample)

```swift
struct ReachabilityCondition: OperationCondition {
    static let hostKey = "Host"
    static let name = "Reachability"
    static let isMutuallyExclusive = false

    let host: NSURL

    // 1
    init(host: NSURL) {
        self.host = host
    }

    // 2
    func dependencyForOperation(operation: Operation) -> NSOperation? {
        return nil
    }

    func evaluateForOperation(operation: Operation, completion: OperationConditionResult -> Void) {
        ReachabilityController.requestReachability(host) { reachable in
            if reachable {
                // 3
                completion(.Satisfied)
            }
            else {
                let error = NSError(code: .ConditionFailed, userInfo: [
                    OperationConditionKey: self.dynamicType.name,
                    self.dynamicType.hostKey: self.host
                ])
                // 4
                completion(.Failed(error))
            }
        }
    }
}
```

1. Condtion 初始化时可以传参数进来。
2. 这个 Condition 没有生成一个 `dependencyForOperation`，因为生成依赖 Operation 的目的是当这个 Operation 运行完后，可以在 evaluateForOperation 时获取之前的运行结果，而这里直接调用 ReachabilityController 的 requestReachability 方法就可以了，所以就免去了这一步。
3. 当结果符合预期时，调用 `completion(.Satisfied)`
4. 当出现异常时，调用 `completion(.Failed(error))`

#### Operation

`Operation` 继承自 `NSOperation`，同时添加了一些方法，主要可以分为 4 部分

* 设置状态变量，同时手动设置 KVO
* 执行 conditions 的 `evaluateForOperation` 方法
* 添加 Observer
* 添加 Condtion

##### 设置状态变量，同时手动设置 KVO

在系统提供的状态的基础上，又添加了一些新的状态，如 `EvaluatingConditions`, `Pending` 等，这些状态的改变都需要触发内置状态的 KVO，如 `isExecuting`, `isFinished`, `isReady` 等。通常的做法会是这样：

```objc
[self willChangeValueForKey:@"isExecuting"];
_state = Executing;
[self didChangeValueForKey:@"isExecuting"];
```
当只有少量的状态改变时，在前后包一层还可以接受，但如果多了的话，就不美观了，这时可以使用 KVO 的一个方法 `+ keyPathsForValuesAffectingValueForKey:`，它的意思是，哪些 keyPaths 的改变会导致 `Key` 发生变化。所以可以定义这几个方法，然后正常设置 `state` 就可以了。

```swift
class func keyPathsForValuesAffectingIsReady() -> Set<NSObject> {
    return ["state"]
}

class func keyPathsForValuesAffectingIsExecuting() -> Set<NSObject> {
    return ["state"]
}

class func keyPathsForValuesAffectingIsFinished() -> Set<NSObject> {
    return ["state"]
}
```

当然，这只是完成了一半，系统知道 state 变了后， `isReady` 会变，然后就会调用 `ready` 方法，所以这三个方法我们也要一并覆盖掉。

```swift
override var executing: Bool {
    return state == .Executing
}

override var finished: Bool {
    return state == .Finished
}

override var ready: Bool {
    switch state {

        case .Pending:
            // 省去不相关的代码
            if super.ready {
                // 1
                evaluateConditions()
            }

            // Until conditions have been evaluated, "isReady" returns false
            return false

        case .Ready:
            return super.ready || cancelled

        default:
            return false
    }
}

```

1. 可以看到，当系统在问某个 Operation 是否 ready 时，`evaluateConditions` 方法会被触发，这里包含了该 Operation 的所有 Conditions 的 `evaluateForOperation` 的执行结果。

##### 执行 conditions 的 `evaluateForOperation` 方法

```swift
private func evaluateConditions() {
    assert(state == .Pending && !cancelled, "evaluateConditions() was called out-of-order")

    state = .EvaluatingConditions

    // 1
    OperationConditionEvaluator.evaluate(conditions, operation: self) { failures in
        self._internalErrors.extend(failures)
        self.state = .Ready
    }
}
```

1. 遍历当前 Operation 的 conditions，执行它们的 `evaluateForOperation` 方法，然后将错误保存在 `_internalErrors` 里，同时将当前的状态设置为 `.Ready`。

或许你会问，如果出现错误，是不是表示条件不满足，如果条件不满足，为什么还要将状态设置为 `.Ready`？ 这是因为当状态设置为 `.Ready` 后，就会执行 `main` 方法，在那里会对 `_internalErrors` 做统一判断。

```swift
override final func main() {
    assert(state == .Ready, "This operation must be performed on an operation queue.")

    if _internalErrors.isEmpty && !cancelled {
        state = .Executing

        // 1
        for observer in observers {
            observer.operationDidStart(self)
        }

        execute()
    }
    else {
        finish()
    }
}
```

1. 这里出现了 observer，当 Operation 处于不同状态时，会调用 observers 的不同方法

##### 添加 Observers

observer 的实现还是比较简单的，首先定义一个 Protocol，所有的 observer 都需要实现这个 Protocol 里的方法，然后 Operation 内置一个数组作为容器，`addObserver` 时，将 observer 添加到容器，当处于不同状态时，遍历容器里的 observer，调用相应的方法。

这不免让我们想起了 delegate，跟 delegate 相比，observer 的好处就在于可以指定多个观察者，而 delegate 只能指定一个。

##### 添加 Condtions

跟 observer 的实现思路基本一致。你或许会问，添加的这些 Conditions 什么时候会被触发呢？没错，就是在将 Operation 添加到 OperationQueue 时。

#### OperationQueue

`OperationQueue` 也是继承自系统的 `NSOperationQueue`，同时重写了 `addOperation` 方法，这个方法主要做了 3 件事

* 给 Operation 添加 observer
* 处理 Operation 的 dependencies 的 `dependencyForOperation`
* 处理 Operation 的 dependencies 的排他性

##### 给 Operation 添加 observer

```swift
let delegate = BlockObserver(
    startHandler: nil,
    produceHandler: { [weak self] in
        // 1
        self?.addOperation($1)
    },
    finishHandler: { [weak self] in
        if let q = self {
            // 2
            q.delegate?.operationQueue?(q, operationDidFinish: $0, withErrors: $1)
        }
    }
)
op.addObserver(delegate)
```

1. 我们前面说过，一个 Operation 可以生成一个新的 Operation，这个 Operation 生成后也需要被放到 Queue 里，这个放置的过程就是在这个 delegate 里实现的。
2. operationQueue 自己有一个 delegate，当 queue 里的一个 operation 执行完时，会向 delegate 报告。

##### 处理 Operation 的 dependencies 的 dependencyForOperation

```swift
// Extract any dependencies needed by this operation.
let dependencies = op.conditions.flatMap {
    $0.dependencyForOperation(op)
}

for dependency in dependencies {
    op.addDependency(dependency)

    self.addOperation(dependency)
}
```

这个就很简单了，调用 `dependencyForOperation` 方法，拿到 operation，然后将当前的 op 依赖该 operation，同时将这个 operation 放到 queue 里，所以在 conditions 的 operations 执行完之前，op 是不会执行的。

##### 处理 Operation 的 dependencies 的排他性

```swift
let concurrencyCategories: [String] = op.conditions.flatMap { condition in
    if !condition.dynamicType.isMutuallyExclusive { return nil }

    return "\(condition.dynamicType)"
}

if !concurrencyCategories.isEmpty {
    // Set up the mutual exclusivity dependencies.
    let exclusivityController = ExclusivityController.sharedExclusivityController

    exclusivityController.addOperation(op, categories: concurrencyCategories)

    op.addObserver(BlockObserver { operation, _ in
        exclusivityController.removeOperation(operation, categories: concurrencyCategories)
    })
}
```

在这里可能看不出「排他」的实现，因为是在 `exclusivityController` 里面实现的，调用了它的 `addOperation` 方法后，它会去查看这个类型的数组是否为空，如果不为空，就让这个 operation 依赖数组的最后一个。这样在之前的 operation 执行完之前，这个 operation 是不会被执行的。

### 使用

有了 Operation 和 OperationQueue 之后，就可以开始生产 megabot 了，来看一个「查看原网页」的 Operation，这个 Operation 的作用就是展示传入的 URL。

```swift
import Foundation
import SafariServices

/// An `Operation` to display an `NSURL` in an app-modal `SFSafariViewController`.
class MoreInformationOperation: Operation {

    let URL: NSURL

    init(URL: NSURL) {
        self.URL = URL
        super.init()
        // 1
        addCondition(MutuallyExclusive<UIViewController>())
    }

    override func execute() {
        dispatch_async(dispatch_get_main_queue()) {
            self.showSafariViewController()
        }
    }

    private func showSafariViewController() {
        if let context = UIApplication.sharedApplication().keyWindow?.rootViewController {
            let safari = SFSafariViewController(URL: URL, entersReaderIfAvailable: false)
            safari.delegate = self
            context.presentViewController(safari, animated: true, completion: nil)
        }
        else {
            finish()
        }
    }
}

extension MoreInformationOperation: SFSafariViewControllerDelegate {
    func safariViewControllerDidFinish(controller: SFSafariViewController) {
        controller.dismissViewControllerAnimated(true) {
            // 2
            self.finish()
        }
    }
}
```

1. 因为这是一个 `ViewController` 相关的 Operation，所以其他同类型的 Operation，需要等我完成后才能被执行。
2. 当这个 controller 被关闭时，表示这个 Operation 结束，调用一下 `finish` 方法。

如果需要的话，可以给这个 Operation 再加一个 `ReachabilityCondition`，当没有网络时就不打开了。

再来看看在 VC 层面的使用。

```swift
override func tableView(tableView: UITableView, didSelectRowAtIndexPath indexPath: NSIndexPath) {

    // 1
    let operation = BlockOperation {
        self.performSegueWithIdentifier("showEarthquake", sender: nil)
    }

    operation.addCondition(MutuallyExclusive<UIViewController>())

    // 2
    let blockObserver = BlockObserver { _, errors in
        /*
            If the operation errored (ex: a condition failed) then the segue
            isn't going to happen. We shouldn't leave the row selected.
        */
        if !errors.isEmpty {
            dispatch_async(dispatch_get_main_queue()) {
                tableView.deselectRowAtIndexPath(indexPath, animated: true)
            }
        }
    }

    operation.addObserver(blockObserver)

    // 3
    operationQueue.addOperation(operation)
}
```

1. 类似 `NSBlockOperation`， `BlockOperation` 也可以快速生成一个 Operation。
2. `BlockObserver` 也是一个快速生成 observer 的方法，这里描述了当 Operation 完成后的处理。
3. 调用方需要新建一个 queue，然后把 Operation 放到这个 queue 里。

相比起正常的调用，还是会多了些步骤。

### 小结

基于 Operation 来架构的思想还是蛮新颖的，可以将复杂的任务拆分成粒度更细的 Operation，然后再组装。但实际使用起来也会有不少问题，比如之前提到的写起来会复杂些，调试时看 backtrace 会很累，不确定是否会带来更好的可维护性等等。不过既然苹果都已经把它用到了线上的 App，至少说明是可行的，至于与已有的架构相比会带来怎样的提升，可能需要实际写起来才知道。
