---
layout: post
title: iOS 统计打点那些事
category: tech
tag: 技术
---

统计打点是 App 开发里很重要的一个环节，App 的运行状态、改版后的效果、用户的各种行为等都需要打点，市面上也有不少可供选择的第三方库。 假设产品有这么个需求：当用户在详情页点击购买按钮时，记录一下事件。我们实现起来大概会是这样

```objc
// DetailViewController.m

- (void)onBuyButtonTapped:(UIButton *)button
{
    // do some stuff, maybe send a request to server
    [XXXAnalytics event:kSomeEventYouDefined];
}
```

这个需求就这样轻松搞定了，但细细想想还是有不少问题的：

1. 页面上会有其他的 Button，可能每个 Button 都要放上这么一段代码。
2. 这些统计其实跟具体的业务无关，没必要跟业务代码混杂在一起，不优雅。
3. 当改版或者重构时，有可能忘了把相应的打点代码迁移过去。

所以需要一种更好的方式来做这件事，这就是使用 AOP([Aspect-Oriented-Programming](http://en.wikipedia.org/wiki/Aspect-oriented_programming))，翻译过来就是「面向切面编程」

> 通过预编译方式和运行期动态代理实现在不修改源代码的情况下给程序动态统一添加功能的一种技术。

简单来说，就是可以动态的在函数调用的前后插一段代码。iOS 可以使用 Pete Steinberger 开发的 [Aspects](https://github.com/steipete/Aspects) 这个库，大致原理是在 runtime 层，通过 swizzle method 来实现的。

来看一个小 Demo

```objc
[UIViewController aspect_hookSelector:@selector(viewWillAppear:) withOptions:AspectPositionAfter usingBlock:^(id<AspectInfo> aspectInfo, BOOL animated) {
    NSLog(@"View Controller %@ will appear animated: %tu", aspectInfo.instance, animated);
} error:NULL];
```

这样在 `UIViewController` 的 `viewWillAppear:` 被调用后，还会再调一下我们定义的 Block，这段日志就会被输出。而打点正好符合这种场景：正事干完之后，额外干一些跟业务无关的事情。

上面的例子，我们通过 AOP 来做的话，大概就是这样

```objc
// DetailViewController.m
- (void)onBuyButtonTapped:(UIButton *)button
{
    // do some stuff, maybe send a request to server
    // no need to call [XXXAnalytics event:]
}

// AppDelegate.m
- (void)setupAnalytics
{
    [DetailViewController aspect_hookSelector:@selector(onBuyButtonTapped:) withOptions:AspectPositionAfter usingBlock:^(id<AspectInfo> aspectInfo, BOOL animated) {
        [XXXAnalytics event:kSomeEventYouDefined];
    } error:NULL];
}
```

这样统计代码就从业务代码中剥离出来了。但是又产生了一个新问题，多个 Button Event，岂不是要写很多行这样的代码，「重复」这样的事情，作为一个程序员怎么能忍，简单，造一个方法

```objc
- (void)trackEventWithClass:(Class)klass selector:(SEL)selector event:(NSString *)event
{
    [klass aspect_hookSelector:@selector(selector) withOptions:AspectPositionAfter usingBlock:^(id<AspectInfo> aspectInfo, BOOL animated) {
        [XXXAnalytics event:event];
    } error:NULL];
}
```

使用起来就像这样

```objc
- (void)setupAnalytics
{
    [self trackEventWithClass:DetailViewController selector:@seletor(onBuyButtonTapped:) event:kSomeEventYouDefined];
    [self trackEventWithClass:ListViewController selector:@seletor(followButtonTapped:) event:kAnotherEventYouDefined];
    // ...
}
```

看起来又干净了些。这时，产品经理又提了个需求：当这个按钮点击时，如果已经登录了，发送 EventA，如果没有登录则发送 EventB，也就是说，不再只是 `[XXXAnalytics event:]` 这么简单了，还需要加上额外的逻辑，这也难不倒我们，加上一个 `block` 即可。

```objc
- (void)trackEventWithClass:(Class)klass
                   selector:(SEL)selector
               eventHandler:(void (^)(id<AspectInfo> aspectInfo))eventHandler
{
    [klass aspect_hookSelector:@selector(selector) withOptions:AspectPositionAfter usingBlock:^(id<AspectInfo> aspectInfo, BOOL animated) {
        if (eventHandler) {
            eventHandler(aspectInfo);
        }
    } error:NULL];
}

// 使用
[self trackEventWithClass:DetailViewController selector:@seletor(onBuyButtonTapped:) eventHandler:^(id<AspectInfo> aspectInfo){
    user.loggedIn ? [XXXAnalytics event:EventA] : [XXXAnalytics event:EventB];
}];
```

好了，现在只要不是太复杂的打点逻辑(那些需要方法上下文变量的)我们都能应付了，接下来就该等产品来验收了。产品搬了个凳子坐在身边，然后点一下 Button，看一下 Console，被几轮蹂躏后，产品也慢慢地接受了这种验收方式。后来某一天，忽然发现某一项或某几项数据有异常，然后找到开发，瞄了一眼：哦，这个方法被重构了。或者新加的方法忘了加统计了。只能等到下个版本再加上了，如果只是一般的统计数据倒还好，跟钱相关的就麻烦了。

那么有没有一种直观的验证方式呢？当然，程序员是万能的呀。一个理想的状况是，产品打开 App 后，开启某个开关就能看到所有会发送 Event 的按钮，就像这样

![](/image/analytics_highlight.jpg)

其中数字代表了 `EventID`。如何实现呢？还记得注册事件时，我们有传入 `class` 和 `selector` 么，一般我们都会有一个 `BaseViewController`，那么就可以在 `BaseViewController` 的 `viewDidAppear:` 里做点文章了。

```objc
// BaseViewController.m

- (void)viewDidAppear:(BOOL)animated
{
    [super viewDidAppear:animated];
    // 获取已经注册过的 classes
    NSDictionary *registeredClasses = [OurAnalytics sharedInstance].registeredClasses;

    [registeredClasses enumerateKeysAndObjectsUsingBlock:^(NSString *className, NSArray *selectors, BOOL *stop) {
        if ([self isKindOfClass:NSClassFromString(className)]) {
            // 如何根据 selector 找到它的宿主？
        }
    }];
}
```

所以现在问题就剩下，如何根据 `selector` 找到对应的 Button，这里要注意，有些 Button 可能要等网络请求完成才会出现，比如 `TableViewCell` 里的 Button。

没有想到太方便的方法，简单粗暴点就是设置个 Timer 每隔一段时间扫一下 subviews，如果是 button 或 包含 tapGesture 的，就拿它们的 action 对比一下，如果 match 就可以高亮那个 button / view 了。

EventID 也一样，之前在注册时也会传一个 EventID 过来，这里直接显示出来即可。对于那些传 `eventHandler` 的就不行了。

所以理论上是可行的，性能上会稍微有点损耗，尤其是当 subViews 的结构比较复杂时，不过只是内部用来做验证，所以这也不是什么问题。

看起来效果已经不错了，有没有可能让这套体系再灵活一些？比如可以从后端制定打点规则？客户端只是读取一个配置文件，就像这样

```objc
- (void)setupAnalytics
{
    // analyticsRules 是从配置文件中读取出来的
    [analyticsRules enumerateObjectsUsingBlock:^(NSDictionary *rules, NSUInteger idx, BOOL *stop) {
        Class klass = NSClassFromString(rules[@"class"]);
        SEL selector = NSSelectorFromString(rules[@"selector"]);
        NSString *eventID = rules[@"eventID"];
        [self trackEventWithClass:klass seletor:seletor event: eventID];
    }];
}
```

那如果在后台的时候填错了 Class 或 Selector 怎么办？还好有 `objc_getClassList` 和 `class_copyMethodList` 这两个运行时方法，有了它们就可以在 App 启动时扫一遍已注册的类（过滤掉 UI / NS 开头的），然后将它们的 seletor 也一并保存下来发送给服务端，当然这种操作只需在适当的时机做一下就可以了，比如集成打包时。

现在，这套体系就比较完整了。当然这只是我的一些构想，并没有在实践中尝试过，所以肯定会踩到各种各样的坑，不过至少看起来是个可行的方案。
