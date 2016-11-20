---
layout: post
title: MVVM without ReactiveCocoa
category: tech
tag: 技术
---

MVVM 是 MVC 模式的一种演进，它主要解决了 ViewController 过于臃肿带来的不易维护和测试的问题。其中 ViewModel 的主要职责是处理业务逻辑并提供 View 所需的数据，这样 VC 就不用关心业务，自然也就瘦了下来。ViewModel 只关心业务数据不关心 View，所以不会与 View 产生耦合，也就更方便进行单元测试。

View 是一个壳，它所呈现的内容都需要由 ViewModel 来提供，而 View 又不与 ViewModel 直接沟通，这时就需要 ViewController 来做中间的协调者。

ViewController 持有 View 和 ViewModel，当 VC 初始化时，会让 ViewModel 去取数据，简单来说就是调用 VM 的某个获取数据的方法。

使用 MVVM 最舒服的姿势是搭配 ReactiveCocoa。不过它的问题在于学习成本和维护成本比较高，在小团队中或许还可以尝试，当开发人员数量较多时就很难推起来了。这也是我们今天要讲的主题：如何不借助 ReactiveCocoa 来实现 MVVM。

先从数据的获取开始说起吧。在 ReactiveCocoa 里有一个类叫「RACCommand」，它的主要作用是执行某个会改变数据的操作，然后提供获取数据的方法，跟我们想要达到的目的很像，所以可以借鉴这个思路，写一个简单的 Command。

```objc
typedef void(^MGJCommandCompletionBlock)(id error, id content);

// 1
typedef void(^MGJCommandConsumeBlock)(id input, MGJCommandCompletionBlock completionHandler);

// 2
typedef void(^MGJCommandCancelBlock)();

@interface MGJCommandResult : NSObject
// 3
@property (nonatomic) NSError *error;
// 4
@property (nonatomic) id content;
@end

@interface MGJCommand : NSObject

// 5
@property (nonatomic, readonly) BOOL executing;
// 6
@property (nonatomic, readonly) MGJCommandResult *result;

- (instancetype)initWithConsumeHandler:(MGJCommandConsumeBlock )consumeHandler;
// 7
- (instancetype)initWithConsumeHandler:(MGJCommandConsumeBlock )consumeHandler cancelHandler:(MGJCommandCancelBlock )cancelHandler;

// 8
- (void)execute:(id)input;
// 9
- (void)cancel;
@end
```

1. `input` 是外部传过来的值，比如 user_id，当拿到数据后，调用下 completionHandler，这样 `result` 属性就会变化
2. 有些操作，如 http 请求，需要手动取消
3. 单独把 `error` 作为一个属性放出来，是因为很多数据请求操作都可能出错，当出错后，只需改变这个 error 属性即可。
4. `content` 存放了这个 Command 的数据处理结果。
5. 标识了这个 Command 目前的运行状态，比如可以根据这个状态来显示 loading。
6. 每次 Command 执行完一个任务后，result 都会改变，外部可以 KVO 这个 result，然后就可以实时获取最新的结果了。
7. Command 的执行逻辑，如果实现了 `cancelHandler` 的话，外部调用 `cancel`，这个 Handler 就会被触发。
8. 外部可以调用这个方法来触发 Command 的执行，同时可以传一个参数进来。
9. 外部可以调用这个方法来取消 Command 的执行。

实现起来也蛮简单的，这里就不多说了。用起来大概是这样：

```objc
// SomeViewModel.m

@weakify(self);
self.followCommand = [[MGJCommand alloc] initWithConsumeHandler:^(id input, MGJCommandCompletionBlock completionHandler) {
    @strongify(self);
    [FollowRequest getFollowList:(NSDictionary *)input success:^(NSArray *users) {
        self.usersToFollow = users;
        completionHandler(nil, kFollowExpertSearchSucceedSignal);
    } failure:^(StatusEntity *error) {
        completionHandler(error, nil);
    }];
}];
```

在 ViewController 里的用法大概像这样

```objc
// SomeViewController.m

- (void)didTapFollowButton:(UIButton *)button
{
	// 根据 button 找到 userID
	[self.viewModel.followCommand execute:userID];
}
```

就是这样，VC 本身不处理业务逻辑，都交给 ViewModel 去处理，而这些数据请求的结果处理又有不同的处理方式。

### Delegate

当 ViewModel 拿到数据后，可以把结果以 Delegate 的方式通知 VC，就像这样

```objc
// SomeViewController.m

- (void)didFollowUserWithResult:(id)result
{
	self.followButton.enabled = YES;
	[self.followButton doSomeAnimation];
}
```

这样做的好处是比较符合苹果既有的设计模式，而且也可以通过查看 Delegate 协议来知道 VM 暴露了哪些接口供外部使用。

不过这种方法少了点灵活性，比如需要联合多个属性的变化来做一些事情时，处理起来就会比较麻烦，这也是 RAC 强大的地方。

### KVO

RAC 是基于 KVO 构建的，所以也可以用 KVO 来让 VC 获取 VM 的变化。

但我们都知道 KVO 的槽点比较多，比如使用起来不方便，用完还要记得移除等。这里可以使用 Facebook 开源的 [KVOController](https://github.com/facebook/KVOController)，它比较好的处理了 KVO 存在的一些问题，同时又能发挥 KVO 带来的便捷性。

有了它我们就能在一个地方把 VM 的更新处理掉了

```objc
- (void)handleViewModelUpdate
{
	[self.KVOController observe:self.viewModel keyPath:@"followCommand.result" options:NSKeyValueObservingOptionInitial|NSKeyValueObservingOptionNew block:^(id observer, id target, NSDictionary *change) {
		id newValue = change[NSKeyValueChangeNewKey];
		// doSomething with the newValue
	}];

	// 对 VM 其他 keyPath 的处理也都可以放到这里
}
```

如果觉得这样的写法还是太麻烦，可以做一层简单的封装，使用起来就像这样

```objc
- (void)handleViewModelUpdate
{
	[self observe:self.viewModel keyPath:@"followCommand.result" block: ^(id newValue){
		// use newValue to update view
	}];
}
```

是不是会好一点，使用 KVO 比 Delegate 好的一点是不用再额外声明协议和方法，而且支持 block，使用起来也会方便些。

对于像 `error` 这样很多操作都会产生同样结果的场景，可以单独拿出来，作为 ViewModel 的一个属性，使用时，直接 KVO 这个属性即可。


### 细节处理

如果不涉及到 TableView 等会出现复用场景的地方，MVVM 使用起来还是比较方便的。如果有了 TableView，又要做一些额外的处理。

一般来说，VC 可以带一个 VM，那如果出现 Cell 时怎么办，Cell 里又包含了按钮，按钮又需要数据请求又怎么处理？这些都是比较常见的场景，也可以通过 MVVM 来解决。

我们知道 VM 的职责是为 View 提供数据支持，Cell 也是一个 View，那么为 Cell 配备一个 VM
不就可以了么。

这样的话，VC 的 VM 需要包含一个数组，里面的元素是 CellVM，使用起来就像这样

```objc
// SomeViewController.m

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
	UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell" forIndexPath:indexPath];
	cell.viewModel = self.viewModel.cellViewModels[indexPath.row];

	// cell 可能会用到 cellVM 里的 Command，所以在这里处理 command 的执行结果
	[self observe:cell keyPath:@"likeCommand.result" block: ^(id newValue){
		// update cell after like
	}];

	return cell
}
```

当然仅仅如此是不够的，我们需要找个恰当的时机把 KVO 移除，避免多次监听。`UITableViewDelegate` 里的这个方法就很适合。

```objc
// SomeViewController.m

- (void)tableView:(UITableView *)tableView didEndDisplayingCell:(UITableViewCell *)cell forRowAtIndexPath:(NSIndexPath *)indexPath
{
	[self unobserve:cell keyPath:@"likeCommand.result"];
}
```

不过这里也要讲究一个平衡，如果 Cell 的类型比较多，且涉及 Command 的地方不多，只是做呈现方面的工作，直接使用 Entity 会更方便。

### Tips

* `ViewController` 尽量不涉及业务逻辑，让 `ViewModel` 去做这些事情。
* `ViewController` 只是一个中间人，接收 `View` 的事件、调用 `ViewModel` 的方法、响应 `ViewModel` 的变化。
* `ViewModel` 不能包含 View，不然就跟 View 产生了耦合，不方便复用和测试。
* `ViewModel` 之间可以有依赖。
* `ViewModel` 避免过于臃肿，不然维护起来也是个问题。

MVVM 并不复杂，跟 MVC 也是兼容的，只是多了一个 ViewModel 层，但就是这么一个小改动，就能让代码变得更加容易阅读和维护，不妨试一下吧。
