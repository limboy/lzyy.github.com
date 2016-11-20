---
layout: post
title: 使用objection来模块化开发iOS项目
category: tech
tag: 技术
---

[objection](https://github.com/atomicobject/objection) 是一个轻量级的依赖注入框架，受[Guice](https://code.google.com/p/google-guice/)的启发，[Google Wallet](http://www.google.com/wallet/) 也是使用的该项目。「依赖注入」是面向对象编程的一种设计模式，用来减少代码之间的耦合度。通常基于接口来实现，也就是说不需要new一个对象，而是通过相关的控制器来获取对象。2013年最火的PHP框架 [laravel](http://laravel.com) 就是其中的典型。

假设有以下场景：ViewControllerA.view里有一个button，点击之后push一个ViewControllerB，最简单的写法类似这样：

{% highlight objc %}
- (void)viewDidLoad
{
    [super viewDidLoad];
    UIButton *button = [UIButton buttonWithType:UIButtonTypeSystem];
    button.frame = CGRectMake(100, 100, 100, 30);
    [button setTitle:@"Button" forState:UIControlStateNormal];
    [button addTarget:self action:@selector(buttonTapped) forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:button];
}

- (void)buttonTapped
{
	ViewControllerB *vc = [[ViewControllerB alloc] init];
    [self.navigationController pushViewController:vc animated:YES];
}
{% endhighlight %}

这样写的一个问题是，ViewControllerA需要import ViewControllerB，也就是对ViewControllerB产生了依赖。依赖的东西越多，维护起来就越麻烦，也容易出现循环依赖的问题，而objection正好可以处理这些问题。

实现方法是：先定义一个协议(protocol)，然后通过objection来注册这个协议对应的class，需要的时候，可以获取该协议对应的object。对于使用方无需关心到底使用的是哪个Class，反正该有的方法、属性都有了(在协议中指定)。这样就去除了对某个特定Class的依赖。也就是通常所说的「面向接口编程」。

{% highlight objc %}
JSObjectionInjector *injector = [JSObjection defaultInjector]; // [1]
UIViewController <ViewControllerAProtocol> *vc = [injector getObject:@protocol(ViewControllerAProtocol)]; // [2]
vc.backgroundColor = [UIColor lightGrayColor]; // [3]
UINavigationController *nc = [[UINavigationController alloc] initWithRootViewController:vc];
self.window.rootViewController = nc;
{% endhighlight %}

* [1] 获取默认的injector，这个injector已经注册过`ViewControllerAProtocol`了。
* [2] 获取`ViewControllerAProtocol`对应的Object。
* [3] 拿到VC后，设置它的某些属性，比如这里的backgroundColor，因为在`ViewControllerAProtocol`里有定义这个属性，所以不会有warning。

可以看到这里没有引用ViewControllerA。再来看看这个`ViewControllerAProtocol`是如何注册到injector中的，这里涉及到了`Module`，对Protocol的注册都是在Module中完成的。Module只要继承`JSObjectionModule`这个Class即可。

{% highlight objc %}
@interface ViewControllerAModule : JSObjectionModule
@end

@implementation ViewControllerAModule
- (void)configure
{
    [self bindClass:[ViewControllerA class] toProtocol:@protocol(ViewControllerAProtocol)];
}
@end
{% endhighlight %}

绑定操作是在`configure`方法里进行的，这个方法在被添加到injector里时会被自动触发。

{% highlight objc %}
JSObjectionInjector *injector = [JSObjection defaultInjector]; // [1]
injector = injector ? : [JSObjection createInjector]; // [2]
injector = [injector withModule:[[ViewControllerAModule alloc] init]]; // [3]
[JSObjection setDefaultInjector:injector]; // [4]
{% endhighlight %}

* [1] 获取默认的 injector
* [2] 如果默认的 injector 不存在，就新建一个
* [3] 往这个 injector 里注册我们的 Module
* [4] 设置该 injector 为默认的 injector

这段代码可以直接放到 `+ (void)load`里执行，这样就可以避免在AppDelegate里import各种Module。

因为我们无法直接获得对应的Class，所以必须要在协议里定义好对外暴露的方法和属性，然后该Class也要实现该协议。

{% highlight objc %}
@protocol ViewControllerAProtocol <NSObject>
@property (nonatomic) NSUInteger currentIndex;
@property (nonatomic) UIColor *backgroundColor;
@end

@interface ViewControllerA : UIViewController <ViewControllerAProtocol>
@end
{% endhighlight %}

通过objection实现依赖注入后，就能更好地实现SRP(Single Responsibility Principle)，代码更简洁，心情更舒畅，生活更美好。拿Pinterest来说，下面的页面就可以划分为3个Section。

![](/image/demo_4_objection.png)

各个Section可以由不同的人负责，然后串到一起就行，也能一定程度地避免MVC(Mess View Controller)的出现。

总体来说，这个lib还是挺靠谱的，已经维护了两年多，也有一些项目在用，对于提高开发成员的效率也会有不少的帮助，可以考虑尝试下。

---- update (2014-04-30) ----

写了个壁纸的demo，[https://github.com/lzyy/bizhi](https://github.com/lzyy/bizhi)
