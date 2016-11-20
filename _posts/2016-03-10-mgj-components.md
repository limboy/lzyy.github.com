---
layout: post
title: 蘑菇街 App 的组件化之路
category: tech
tag: 技术
---

在组件化之前，蘑菇街 App 的代码都是在一个工程里开发的，在人比较少，业务发展不是很快的时候，这样是比较合适的，能一定程度地保证开发效率。

慢慢地代码量多了起来，开发人员也多了起来，业务发展也快了起来，这时单一工程开发模式就会显露出一些弊端

* 耦合比较严重（因为没有明确的约束，「组件」间引用的现象会比较多）
* 容易出现冲突（尤其是使用 Xib，还有就是 Xcode Project，虽说有[脚本](https://github.com/truebit/xUnique)可以改善）
* 业务方的开发效率不够高（只关心自己的组件，却要编译整个项目，与其他不相干的代码糅合在一起）

为了解决这些问题，就采取了「组件化」策略。它能带来这些好处

* 加快编译速度（不用编译主客那一大坨代码了）
* 自由选择开发姿势（MVC / MVVM / FRP）
* 方便 QA 有针对性地测试
* 提高业务开发效率

先来看下，组件化之后的一个大概架构

![](/image/14575533415332.jpg)

「组件化」顾名思义就是把一个大的 App 拆成一个个小的组件，相互之间不直接引用。那如何做呢？

### 实现方式

#### 组件间通信
以 iOS 为例，由于之前就是采用的 URL 跳转模式，理论上页面之间的跳转只需 open 一个 URL 即可。所以对于一个组件来说，只要定义「支持哪些 URL」即可，比如详情页，大概可以这么做的

```objc
[MGJRouter registerURLPattern:@"mgj://detail?id=:id" toHandler:^(NSDictionary *routerParameters) {
    NSNumber *id = routerParameters[@"id"];
    // create view controller with id
    // push view controller
}];
```

首页只需调用 `[MGJRouter openURL:@"mgj://detail?id=404"]` 就可以打开相应的详情页。

那问题又来了，我怎么知道有哪些可用的 URL？为此，我们做了一个后台专门来管理。

![](/image/14575445324533.jpg)

然后可以把这些短链生成不同平台所需的文件，iOS 平台生成 .{h,m} 文件，Android 平台生成 .java 文件，并注入到项目中。这样开发人员只需在项目中打开该文件就知道所有的可用 URL 了。

目前还有一块没有做，就是参数这块，虽然描述了短链，但真想要生成完整的 URL，还需要知道如何传参数，这个正在开发中。

还有一种情况会稍微麻烦点，就是「组件A」要调用「组件B」的某个方法，比如在商品详情页要展示购物车的商品数量，就涉及到向购物车组件拿数据。

类似这种同步调用，iOS 之前采用了比较简单的方案，还是依托于 `MGJRouter`，不过添加了新的方法 `- (id)objectForURL:`，注册时也使用新的方法进行注册

```objc
[MGJRouter registerURLPattern:@"mgj://cart/ordercount" toObjectHandler:^id(NSDictionary *routerParamters){
	// do some calculation
	return @42;
}]
```

使用时 `NSNumber *orderCount = [MGJRouter objectForURL:@"mgj://cart/ordercount"]` 这样就拿到了购物车里的商品数。

稍微复杂但更具通用性的方法是使用「协议」 <-> 「类」绑定的方式，还是以购物车为例，购物车组件可以提供这么个 Protocol

```objc
@protocol MGJCart <NSObject>
+ (NSInteger)orderCount;
@end
```

可以看到通过协议可以直接指定返回的数据类型。然后在购物车组件内再新建个类实现这个协议，假设这个类名为`MGJCartImpl`，接着就可以把它与协议关联起来 `[ModuleManager registerClass:MGJCartImpl forProtocol:@protocol(MGJCart)]`，对于使用方来说，要拿到这个 `MGJCartImpl`，需要调用 `[ModuleManager classForProtocol:@protocol(MGJCart)]`。拿到之后再调用 `+ (NSInteger)orderCount` 就可以了。

那么，这个协议放在哪里比较合适呢？如果跟组件放在一起，使用时还是要先引入组件，如果有多个这样的组件就会比较麻烦了。所以我们把这些公共的协议统一放到了 `PublicProtocolDomain.h` 下，到时只依赖这一个文件就可以了。

Android 也是采用类似的方式。

#### 组件生命周期管理
理想中的组件可以很方便地集成到主客中，并且有跟 `AppDelegate` 一致的回调方法。这也是 `ModuleManager` 做的事情。

先来看看现在的入口方法

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    [MGJApp startApp];

    [[ModuleManager sharedInstance] loadModuleFromPlist:[[NSBundle mainBundle] pathForResource:@"modules" ofType:@"plist"]];
    NSArray *modules = [[ModuleManager sharedInstance] allModules];
    for (id<ModuleProtocol> module in modules) {
        if ([module respondsToSelector:_cmd]) {
            [module application:application didFinishLaunchingWithOptions:launchOptions];
        }
    }
    
    [self trackLaunchTime];
    return YES;
}
```
其中 `[MGJApp startApp]` 主要负责一些 SDK 的初始化。`[self trackLaunchTime]` 是我们打的一个点，用来监测从 `main` 方法开始到入口方法调用结束花了多长时间。其他的都由 `ModuleManager` 搞定，`loadModuleFromPlist:pathForResource:` 方法会读取 bundle 里的一个 plist 文件，这个文件的内容大概是这样的

![](/image/14575489295366.jpg)

每个 `Module` 都实现了 `ModuleProtocol`，其中有一个 `- (BOOL)applicaiton:didFinishLaunchingWithOptions:` 方法，如果实现了的话，就会被调用。

还有一个问题就是，系统的一些事件会有通知，比如 `applicationDidBecomeActive` 会有对应的 `UIApplicationDidBecomeActiveNotification`，组件如果要做响应的话，只需监听这个系统通知即可。但也有一些事件是没有通知的，比如 `- application:didRegisterUserNotificationSettings:`，这时组件如果也要做点事情，怎么办？

一个简单的解决方法是在 `AppDelegate` 的各个方法里，手动调一遍组件的对应的方法，如果有就执行。

```objc
- (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken
{
    NSArray *modules = [[ModuleManager sharedInstance] allModules];
    for (id<ModuleProtocol> module in modules) {
        if ([module respondsToSelector:_cmd]) {
            [module application:application didRegisterForRemoteNotificationsWithDeviceToken:deviceToken];
        }
    }
}
```

#### 壳工程
既然已经拆出去了，那拆出去的组件总得有个载体，这个载体就是壳工程，壳工程主要包含一些基础组件和业务SDK，这也是主工程包含的一些内容，所以如果在壳工程可以正常运行的话，到了主工程也没什么问题。不过这里存在版本同步问题，之后会说到。

#### 遇到的问题

##### 组件拆分
由于之前的代码都是在一个工程下的，所以要单独拿出来作为一个组件就会遇到不少问题。首先是组件的划分，当时在定义组件粒度时也花了些时间讨论，究竟是粒度粗点好，还是细点好。粗点的话比较有利于拆分，细点的话灵活度比较高。最终还是选择粗一点的粒度，先拆出来再说。

假如要把详情页迁出来，就会发现它依赖了一些其他部分的代码，那最快的方式就是直接把代码拷过来，改个名使用。比较简单暴力。说起来比较简单，做的时候也是挺有挑战的，因为正常的业务并不会因为「组件化」而停止，所以开发同学们需要同时兼顾正常的业务和组件的拆分。

##### 版本管理
我们的组件包括第三方库都是通过 Cocoapods 来管理的，其中组件使用了私有库。之所以选择 Cocoapods，一个是因为它比较方便，还有就是用户基数比较大，且社区也比较活跃（活跃到了会时不时地触发 Github 的 rate limit，导致长时间 clone 不下来··· [见此](https://github.com/CocoaPods/CocoaPods/issues/4989#issuecomment-193772935)），当然也有其他的管理方式，比如 submodule / subtree，在开发人员比较多的情况下，方便、灵活的方案容易占上风，虽然它也有自己的问题。主要有版本同步和更新/编译慢的问题。

假如基础组件做了个 API 接口升级，这个升级会对原有的接口做改动，自然就会升一个中位的版本号，比如原先是 1.6.19，那么现在就变成 1.7.0 了。而我们在 Podfile 里都是用 `~` 指定的，这样就会出现主工程的 pod 版本升上去了，但是壳工程没有同步到，然后群里就会各种反馈编译不过，而且这个编译不过的长尾有时能拖上两三天。

然后我们就想了个办法，如果不在壳工程里指定基础库的版本，只在主工程里指定呢，理论上应该可行，只要不出现某个基础库要同时维护多个版本的情况。但实践中发现，壳工程有时会莫名其妙地升不上去，在 podfile 里指定最新的版本又可以升上去，所以此路不通。

还有一个问题是 `pod update` 时间过长，经常会在 `Analyzing Dependency` 上卡 10 多分钟，非常影响效率。后来排查下来是跟组件的 Podspec 有关，配置了 subspec，且依赖比较多。

然后就是 pod update 之后的编译，由于是源码编译，所以这块的时间花费也不少，接下去会考虑 framework 的方式。

### 持续集成
在刚开始，持续集成还不是很完善，业务方升级组件，直接把 podspec 扔到 private repo 里就完事了。这样最简单，但也经常会带来编译通不过的问题。而且这种随意的版本升级也不太能保证质量。于是我们就搭建了一套持续集成系统，大概如此

![](/image/14575538180893.jpg)

每个组件升级之前都需要先通过编译，然后再决定是否升级。这套体系看起来不复杂，但在实施过程中经常会遇到后端的并发问题，导致业务方要么集成失败，要么要等不少时间。而且也没有一个地方可以呈现当前版本的组件版本信息。还有就是业务方对于这种命令行的升级方式接受度也不是很高。

![](/image/14575547778269.jpg)

基于此，在经过了几轮讨论之后，有了新版的持续集成平台，升级操作通过网页端来完成。

大致思路是，业务方如果要升级组件，假设现在的版本是 0.1.7，添加了一些 feature 之后，壳工程测试通过，想集成到主工程里看看效果，或者其他组件也想引用这个最新的，就可以在后台手动把版本升到 0.1.8-rc.1，这样的话，原先依赖 `~> 0.1.7` 的组件，不会升到 0.1.8，同时想要测试这个组件的话，只要手动把版本调到 0.1.8-rc.1 就可以了。这个过程不会触发 CI 的编译检查。

当测试通过后，就可以把尾部的 `-rc.n` 去掉，然后点击「集成」，就会走 CI 编译检查，通过的话，会在主工程的 podfile 里写上固定的版本号 0.1.8。也就是说，podfile 里所有的组件版本号都是固定的。

![](/image/14575547304396.jpg)

### 周边设施

#### 基础组件及组件的文档 / Demo / 单元测试
无线基础的职能是为集团提供解决方案，只是在蘑菇街 App 里能 work 是远远不够的，所以就需要提供入口，知道有哪些可用组件，并且如何使用，就像这样（目前还未实现）

![](/image/14575551851317.jpg)

这就要求组件的负责人需要及时地更新 README / CHANGELOG / API，并且当发生 API 变更时，能够快速通知到使用方。

#### 公共 UI 组件
组件化之后还有一个问题就是资源的重复性，以前在一个工程里的时候，资源都可以很方便地拿到，现在独立出去了，也不知道哪些是公用的，哪些是独有的，索性都放到自己的组件里，这样就会导致包变大。还有一个问题是每个组件可能是不同的产品经理在跟，而他们很可能只关注于自己关心的页面长什么样，而忽略了整体的样式。公共 UI 组件就是用来解决这些问题的，这些组件甚至可以跨 App 使用。（目前还未实现）

![](/image/14575557095716.jpg)

### 小结
「组件化」是 App 膨胀到一定体积后的解决方案，能一定程度上解决问题，在提高开发效率的过程中，采坑是难免的，希望这篇文章能够带来些帮助。


