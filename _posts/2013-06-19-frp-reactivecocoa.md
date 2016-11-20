---
layout: post
title: ReactiveCocoa与Functional Reactive Programming
category: tech
tag: 技术
---

## 什么是Functional Reactive Programming

Functional Reactive Programming(以下简称FRP)是一种响应变化的编程范式。先来看一小段代码

{% highlight objective-c %}
a = 2
b = 2
c = a + b // c is 4

b = 3
// now what is the value of c?
{% endhighlight %}

如果使用FRP，`c`的值将会随着`b`的值改变而改变，所以叫做「响应式编程」。比较直观的例子就是Excel，当改变某一个单元格的内容时，该单元格相关的计算结果也会随之改变。

FRP提供了一种信号机制来实现这样的效果，通过信号来记录值的变化。信号可以被叠加、分割或合并。通过对信号的组合，就不需要去监听某个值或事件。

![FRP demo](/image/FRP_demo.png)

这在重交互的应用里是非常有用的。以注册为例：

![register demo](/image/FRP_register_demo.png)

提交按钮的状态要跟输入框的状态绑定，比如必选的输入框没有填完时，提交按钮是灰色的，也就是不可点；如果提交按钮不可点，那么文字变成灰色，不然变成蓝色；如果正在提交，那么输入框的文字颜色变成灰色，且不可点，不然变成默认色且可点；如果注册成功就在状态栏显示成功信息，不然显示错误信息，等等。

可以看到光是注册页就有这么多的联动，在javascript中可以采用事件监听来处理，iOS中更多的是delegate模式，本质上都是事件的分发和响应。这种做法的缺点是不够直观，尤其在逻辑比较复杂的情况下。这也是为什么尽管nodejs很高效，但由于javascript的callback style和异步模式不符合正常的编程习惯，让很多人望而却步。

使用FRP主要有两个好处：直观和灵活。直观的代码容易编写、阅读和维护，灵活的特性便于应对变态的需求。

## ReactiveCocoa

[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)是github去年开源的一个项目，是在iOS平台上对FRP的实现。FRP的核心是信号，信号在ReactiveCocoa(以下简称RAC)中是通过`RACSignal`来表示的，信号是数据流，可以被绑定和传递。

可以把信号想象成水龙头，只不过里面不是水，而是玻璃球(value)，直径跟水管的内径一样，这样就能保证玻璃球是依次排列，不会出现并排的情况(数据都是线性处理的，不会出现并发情况)。水龙头的开关默认是关的，除非有了接收方(subscriber)，才会打开。这样只要有新的玻璃球进来，就会自动传送给接收方。可以在水龙头上加一个过滤嘴(filter)，不符合的不让通过，也可以加一个改动装置，把球改变成符合自己的需求(map)。也可以把多个水龙头合并成一个新的水龙头(combineLatest:reduce:)，这样只要其中的一个水龙头有玻璃球出来，这个新合并的水龙头就会得到这个球。

下面通过一个简单的demo来演示这个模型。

假如对象的某个属性想绑定某个消息，可以使用`RAC`这个宏，相当于给玻璃球套了一个水龙头。

{% highlight objective-c %}
RAC(self.submitButton, enabled) = [RACSignal combineLatest:@[self.usernameField.rac_textSignal, self.passwordField.rac_textSignal] reduce:^id(NSString *userName, NSString *password) {
	return @(userName.length >= 6 && password.length >= 6);
}];
{% endhighlight %}

这样，如果用户名和密码框的长度都超过6，提交按钮就enable了。反之，如果没符合要求，就会处于非开启状态。

可以看到`usernameField`有了一个新的属性`rac_textSignal`，这是RAC在`UITextField`category中添加的，直接用即可。

![combine signal](/image/FRP_combine.png)

## RAC的大统一

RAC统一了对KVO、UI Event、Network request、Async work的处理，因为它们本质上都是值的变化(Values over time)。

### KVO

RAC可以用来监测属性的改变，这点跟KVO很像，不过使用了block，而不是`-observeValueForKeyPath:ofObject:change:context:`

{% highlight objective-c %}
[RACAble(self.username) subscribeNext:^(NSString *newName) {
    NSLog(@"%@", newName);
}];
{% endhighlight %}

使用起来是不是比KVO舒服多了。比KVO更加强大的是信号可以被链起来(chain)

{% highlight objective-c %}
// 只有当名字以'j'开头，才会被记录
[[RACAble(self.username)
   filter:^(NSString *newName) {
       return [newName hasPrefix:@"j"];
   }]
   subscribeNext:^(NSString *newName) {
       NSLog(@"%@", newName);
   }];
{% endhighlight %}

### UI Event

RAC还为系统UI提供了很多category，来方便消息的创建和传递，比如按钮被点击或文本框有改动等等，上面的例子中`self.firstNameField.rac_textSignal`，在对应的文本框有改动时，会自动向数据流中添加新的数据，绑定该消息的其他消息就会收到新的数据，如果有subscriber的话，会自动触发。

### Network Request && Async work

这些可以通过自定义信号，也就是`RACSubject`(继承自`RACSignal`，可以理解为自由度更高的signal)来搞定。比如一个异步网络操作，可以返回一个subject，然后将这个subject绑定到一个subscriber或另一个信号。

{% highlight objective-c %}
- (void)doTest
{
    RACSubject *subject = [self doRequest];
    
    [subject subscribeNext:^(NSString *value){
        NSLog(@"value:%@", value);
    }];
}

- (RACSubject *)doRequest
{
    RACSubject *subject = [RACSubject subject];
	// 模拟2秒后得到请求内容
	// 只触发1次
	// 尽管subscribeNext什么也没做，但如果没有的话map是不会执行的
	// subscribeNext就是定义了一个接收体
    [[[[RACSignal interval:2] take:1] map:^id(id _){
        // the value is from url request
        NSString *value = @"content fetched from web";
        [subject sendNext:value];
        return nil;
    }] subscribeNext:^(id _){}];
    return subject;
}
{% endhighlight %}

## 小结

简单画了下关系图，罗列了些要点

[![关系图和要点](/image/FRP_ReactiveCocoa_middle.png)](/image/FRP_ReactiveCocoa_large.png)

上面只是大概说了一下RAC的使用情景和用法，更多的例子可以到[项目主页](https://github.com/ReactiveCocoa/ReactiveCocoa)中查看。

### 参考

* [http://www.teehanlax.com/blog/getting-started-with-reactivecocoa/](http://www.teehanlax.com/blog/getting-started-with-reactivecocoa/)
* [https://speakerdeck.com/andrewsardone/reactivecocoa-at-mobidevday-2013](https://speakerdeck.com/andrewsardone/reactivecocoa-at-mobidevday-2013)
