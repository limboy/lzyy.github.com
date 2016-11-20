---
layout: post
title: 说说ReactiveCocoa 2
category: tech
tag: 技术
---

[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)是[Github](https://github.com/blog/1107-reactivecocoa-for-a-better-world)开源的一款cocoa FRP 框架，我在[之前的文章](http://blog.leezhong.com/ios/2013/06/19/frp-reactivecocoa.html)里有过介绍(当时还是1.x版本，2.x版本有了新的变化，API也有部分不兼容) 这里再简单地提一下。

Native app有很大一部分的时间是在等待事件发生，然后响应事件，比如等待网络请求完成，等待用户的操作，等待某些状态值的改变等等，等这些事件发生后，再做进一步处理。 但是这些等待和响应，并没有一个统一的处理方式。Delegate, Notification, Block, KVO, 常常会不知道该用哪个最合适。有时需要chain或者compose某几个事件，就需要多个状态变量，而状态变量一多，复杂度也就上来了。为了解决这些问题，Github的工程师们开发了ReactiveCocoa。

## 几个常见的概念

在阅读ReactiveCocoa(以下简称RAC)的相关文章或代码时，经常会出现一些名词，理解它们对于理解RAC有很大的帮助，下面就简要来说说这些常见的概念。

### Signal and Subscriber

这是RAC最核心的内容，这里我想用插头和插座来描述，插座是Signal，插头是Subscriber。想象某个遥远的星球，他们的电像某种物质一样被集中存储，且很珍贵。插座负责去获取电，插头负责使用电，而且一个插座可以插任意数量的插头。当一个插座(Signal)没有插头(Subscriber)时什么也不干，也就是处于冷(Cold)的状态，只有插了插头时才会去获取，这个时候就处于热(Hot)的状态。

Signal获取到数据后，会调用Subscriber的sendNext, sendComplete, sendError方法来传送数据给Subscriber，Subscriber自然也有方法来获取传过来的数据，如：[signal subscribeNext:error:completed]。这样只要没有sendComplete和sendError，新的值就会通过sendNext源源不断地传送过来，举个简单的例子：

{% highlight objc %}
[RACObserve(self, username) subscribeNext: ^(NSString *newName){
	NSLog(@"newName:%@", newName);
}];
{% endhighlight %}

`RACObserve`使用了KVO来监听property的变化，只要username被自己或外部改变，block就会被执行。但不是所有的property都可以被`RACObserve`，该property必须支持KVO，比如NSURLCache的currentDiskUsage就不能被`RACObserve`。

Signal是很灵活的，它可以被修改(map)，过滤(filter)，叠加(combine)，串联(chain)，这有助于应对更加复杂的情况，比如：

{% highlight objc %}
RAC(self.logInButton, enabled) = [RACSignal
        combineLatest:@[
            self.usernameTextField.rac_textSignal,
            self.passwordTextField.rac_textSignal,
            RACObserve(LoginManager.sharedManager, loggingIn),
            RACObserve(self, loggedIn)
        ] reduce:^(NSString *username, NSString *password, NSNumber *loggingIn, NSNumber *loggedIn) {
            return @(username.length > 0 && password.length > 0 && !loggingIn.boolValue && !loggedIn.boolValue);
        }];
{% endhighlight %}

这段代码看起来有点复杂，来细细说一下，首先是左边的`RAC(...)`，它的作用是将`self.logInButton.enabled`属性与右边的signal的sendNext值绑定。也就是如果右边的reduce的返回值为NO，那么enabled就为NO。右边的`combineLatest`是获取这4个signal的next值。其中可以看到`self.usernameTextField.rac_textSignal`这么个东东，`rac_textSignal`是RAC为UITextField添加的category，只要usernameTextField的值有变化，这个值就会被返回(sendNext)。combineLatest需要每个signal至少都有过一次sendNext。reduce的作用是根据接收到的值，再返回一个新的值，这里是@(YES)和@(NO)，必须是object。

上面这段代码用到了Signal的组合，想象一下，如果是传统的方式，写起来还是挺复杂的，而且随着功能的增加，调整起来会更加麻烦。

### 冷信号(Cold)和热信号(Hot)

上面提到过这两个概念，冷信号默认什么也不干，比如下面这段代码

{% highlight objc %}
RACSignal *signal = [RACSignal createSignal:^ RACDisposable * (id<RACSubscriber> subscriber) {
	NSLog(@"triggered");
	[subscriber sendNext:@"foobar"];
    [subscriber sendCompleted];
    return nil;
}];

{% endhighlight %}

我们创建了一个Signal，但因为没有被subscribe，所以什么也不会发生。加了下面这段代码后，signal就处于Hot的状态了，block里的代码就会被执行。

{% highlight objc %}
[signal subscribeCompleted:^{
    NSLog(@"subscription %u", subscriptions);
}];
{% endhighlight %}

或许你会问，那如果这时又有一个新的subscriber了，signal的block还会被执行吗？这就牵扯到了另一个概念：Side Effect

### Side Effect

还是上面那段代码，如果有多个subscriber，那么signal就会又一次被触发，控制台里会输出两次`triggered`。这或许是你想要的，或许不是。如果要避免这种情况的发生，可以使用 `replay` 方法，它的作用是保证signal只被触发一次，然后把sendNext的value存起来，下次再有新的subscriber时，直接发送缓存的数据。


## Cocoa Categories

为了更加方便地使用RAC，RAC给Cocoa添加了很多category，与系统集成地越紧密，使用起来自然也就越方便。下面是我认为比较常用的categories。

### UIView Categories

上面看到的`rac_textSignal`是加在UITextField上的(UITextField+RACSignalSupport.h)，其他常用的UIView也都有添加相应的category，比如UIAlertView，就不需要再用Delegate了。

{% highlight objc %}
UIAlertView *alertView = [[UIAlertView alloc] initWithTitle:@"" message:@"Alert" delegate:nil cancelButtonTitle:@"YES" otherButtonTitles:@"NO", nil];
[[alertView rac_buttonClickedSignal] subscribeNext:^(NSNumber *indexNumber) {
	if ([indexNumber intValue] == 1) {
		NSLog(@"you touched NO");
	} else {
		NSLog(@"you touched YES");
	}
}];
[alertView show];
{% endhighlight %}

有了这些Category，大部分的Delegate都可以使用RAC来做。或许你会想，可不可以subscribe NSMutableArray.rac_sequence.signal，这样每次有新的object或旧的object被移除时都能知道，UITableViewController就可以根据dataSource的变化，来reloadData。但很可惜这样不行，因为RAC是基于KVO的，而NSMutableArray并不会在调用addObject或removeObject时发送通知，所以不可行。不过可以使用NSArray作为UITableView的dataSource，只要dataSource有变动就换成新的Array，这样就可以了。

说到UITableView，再说一下UITableViewCell，RAC给UITableViewCell提供了一个方法：`rac_prepareForReuseSignal`，它的作用是当Cell即将要被重用时，告诉Cell。想象Cell上有多个button，Cell在初始化时给每个button都`addTarget:action:forControlEvents`，被重用时需要先移除这些target，下面这段代码就可以很方便地解决这个问题：

{% highlight objc %}
[[[self.cancelButton
	rac_signalForControlEvents:UIControlEventTouchUpInside]
	takeUntil:self.rac_prepareForReuseSignal]
	subscribeNext:^(UIButton *x) {
	// do other things
}];
{% endhighlight %}

还有一个很常用的category就是`UIButton+RACCommandSupport.h`，它提供了一个property：`rac_command`，就是当button被按下时会执行的一个命令，命令被执行完后可以返回一个signal，有了signal就有了灵活性。比如点击投票按钮，先判断一下有没有登录，如果有就发HTTP请求，没有就弹出登陆框，可以这么实现。

{% highlight objc %}
voteButton.rac_command = [[RACCommand alloc] initWithEnabled:self.viewModel.voteCommand.enabled signalBlock:^RACSignal *(id input) {
	// Assume that we're logged in at first. We'll replace this signal later if not.
	RACSignal *authSignal = [RACSignal empty];
	
	if ([[PXRequest apiHelper] authMode] == PXAPIHelperModeNoAuth) {
		// Not logged in. Replace signal.
		authSignal = [[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
			@strongify(self);
			
			FRPLoginViewController *viewController = [[FRPLoginViewController alloc] initWithNibName:@"FRPLoginViewController" bundle:nil];
			UINavigationController *navigationController = [[UINavigationController alloc] initWithRootViewController:viewController];
			
			[self presentViewController:navigationController animated:YES completion:^{
				[subscriber sendCompleted];
			}];
			
			return nil;
		}]];
	}
	
	return [authSignal then:^RACSignal *{
		@strongify(self);
		return [[self.viewModel.voteCommand execute:nil] ignoreValues];
	}];
}];
[voteButton.rac_command.errors subscribeNext:^(id x) {
	[x subscribeNext:^(NSError *error) {
		[SVProgressHUD showErrorWithStatus:[error localizedDescription]];
	}];
}];
{% endhighlight %}

这段代码节选自AshFurrow的[FunctionalReactivePixels](https://github.com/AshFurrow/FunctionalReactivePixels)，有删减。

### Data Structure Categories

常用的数据结构，如NSArray, NSDictionary也都有添加相应的category，比如`NSArray`添加了`rac_sequence`，可以将`NSArray`转换为`RACSequence`，顺便说一下`RACSequence`, `RACSequence`是一组immutable且有序的values，不过这些values是运行时计算的，所以对性能提升有一定的帮助。`RACSequence`提供了一些方法，如`array`转换为`NSArray`，`any:`检查是否有Value符合要求，`all:`检查是不是所有的value都符合要求，这里的符合要求的，block返回YES，不符合要求的就返回NO。

### NotificationCenter Category

`NSNotificationCenter`, 默认情况下`NSNotificationCenter`使用`Target-Action`方式来处理Notification，这样就需要另外定义一个方法，这就涉及到编程领域的两大难题之一：起名字。有了RAC，就有Signal，有了Signal就可以subscribe，于是`NotificationCenter`就可以这么来处理，还不用担心移除observer的问题。

{% highlight objc %}
[[[NSNotificationCenter defaultCenter] rac_addObserverForName:@"MyNotification" object:nil] subscribeNext:^(NSNotification *notification) {
	NSLog(@"Notification Received");
}];
{% endhighlight %}

### NSObject Categories

NSObject有不少的Category，我觉得比较有用的有这么几个

#### NSObject+RACDeallocating.h

顾名思义就是在一个object的dealloc被触发时，执行的一段代码。

{% highlight objc %}
NSArray *array = @[@"foo"];
[[array rac_willDeallocSignal] subscribeCompleted:^{
	NSLog(@"oops, i will be gone");
}];
array = nil;
{% endhighlight %}

#### NSObject+RACLifting.h

有时我们希望满足一定条件时，自动触发某个方法，有了这个category就可以这么办

{% highlight objc %}
- (void)test
{
    RACSignal *signalA = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        double delayInSeconds = 2.0;
        dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delayInSeconds * NSEC_PER_SEC));
        dispatch_after(popTime, dispatch_get_main_queue(), ^(void){
            [subscriber sendNext:@"A"];
        });
        return nil;
    }];
    
    RACSignal *signalB = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [subscriber sendNext:@"B"];
        [subscriber sendNext:@"Another B"];
        [subscriber sendCompleted];
        return nil;
    }];
    
    [self rac_liftSelector:@selector(doA:withB:) withSignals:signalA, signalB, nil];
}

- (void)doA:(NSString *)A withB:(NSString *)B
{
    NSLog(@"A:%@ and B:%@", A, B);
}
{% endhighlight %}

这里的`rac_liftSelector:withSignals` 就是干这件事的，它的意思是当signalA和signalB都至少sendNext过一次，接下来只要其中任意一个signal有了新的内容，`doA:withB`这个方法就会自动被触发。

如果你有兴趣，可以想想上面这段代码会输出什么。

#### NSObject+RACSelectorSignal.h

这个category有`rac_signalForSelector:`和`rac_signalForSelector:fromProtocol:` 这两个方法。先来看前一个，它的意思是当某个selector被调用时，再执行一段指定的代码，相当于hook。比如点击某个按钮后，记个日志。后者表示该selector实现了某个协议，所以可以用它来实现Delegate。

## MVVM

RAC带来的变化还不仅仅是这些，它还带来了架构层面的变化。我们都知道苹果推荐的是MVC架构，那MVVM又是什么呢？

![](https://f.cloud.github.com/assets/432536/867984/291ed380-f760-11e2-9106-d3158320af39.png)

跟MVC最大的区别是多了个`ViewModel`，它直接与View绑定，而且对View一无所知。拿做菜打比方的话，ViewModel就是调料，它不关心做的到底是什么菜。这不是跟`Model`很像吗？是的，它可以扮演Model的职责，但其实它是Model的中介，这样当Model的API有变化，或者由本地存储变为远程API调用时，ViewModel的public API可以保持不变。

使用ViewModel的好处是，可以让Controller更加简单和轻便，而且ViewModel相对独立，也更加方便测试和重用。那Controller这时又该做哪些事呢？在MVVM体系中，Controller可以被看成View，所以它的主要工作是处理布局、动画、接收系统事件、展示UI。

MVVM还有一个很重要的概念是 `data binding`，view的呈现需要data，这个data就是由ViewModel提供的，将view的data与ViewModel的data绑定后，将来双方的数据只要一方有变化，另一方就能收到。[这里](https://github.com/ReactiveCocoa/ReactiveViewModel)有Github 开源的一个ViewModel Base Class。

## 其他

RAC在使用时有一些注意事项，可以参考官方的[DesignGuildLines](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/Documentation/DesignGuidelines.md)，这里简单说一下。

当一个signal被一个subscriber subscribe后，这个subscriber何时会被移除？答案是当subscriber被sendComplete或sendError时，或者手动调用[disposable dispose]。

当subscriber被dispose后，所有该subscriber相关的工作都会被停止或取消，如http请求，资源也会被释放。

Signal events是线性的，不会出现并发的情况，除非显示地指定Scheduler。所以`-subscribeNext:error:completed:`里的block不需要锁定或者synchronized等操作，其他的events会依次排队，直到block处理完成。

Errors有优先权，如果有多个signals被同时监听，只要其中一个signal sendError，那么error就会立刻被传送给subscriber，并导致signals终止执行。相当于Exception。

生成Signal时，最好指定Name, `-setNameWithFormat:` 方便调试。

block代码中不要阻塞。

## 小结

尽管洋洋洒洒写了这么多，也只是对RAC有了个大概的了解，如果要更深入地了解RAC还是需要多读文档、代码和相关项目。

RAC学习起来稍显吃力，且相关的文章目前还不多，中文的就更少了，希望这篇文章能带给你些帮助。

以下是我觉得还不错的RAC相关资源

* [FunctionalReactivePixels](https://github.com/AshFurrow/FunctionalReactivePixels) 作者同时还出了一本FRP相关的书，个人觉得看源码就足够了。
* [GroceryList](https://github.com/jspahrsummers/GroceryList) RAC的作者之一 jspahrsummers 的一个项目
* [ReactiveCocoa Essentilas: Understanding and Using RACCommand](http://codeblog.shape.dk/blog/2013/12/05/reactivecocoa-essentials-understanding-and-using-raccommand/) 介绍了RACCommand的使用，同时也涉及了RAC相关的一些点。
* [Transparent OAuth Token Refresh Using ReactiveCocoa](http://codeblog.shape.dk/blog/2013/12/02/transparent-oauth-token-refresh-using-reactivecocoa/) 这篇文章讲了如何使用RAC来透明地获取Access Token，然后继续发送请求。
* [BNR: An Introduction to ReactiveCocoa](http://vimeo.com/78749139)(视频)
