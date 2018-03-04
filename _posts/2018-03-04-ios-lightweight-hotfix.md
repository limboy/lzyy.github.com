---
layout: post
title: 轻量级低风险 iOS Hotfix 方案
category: tech
tags: 技术
---

我们都知道苹果对 Hotfix 抓得比较严，强大好用的 JSPatch 也成为了过去式。但即使测试地再细致，也难保线上 App 不出问题，小问题还能忍忍，大问题就得重新走发布流程，然后等待审核通过，等待用户升级，周期长且麻烦。如果有一种方式相对比较安全，不需要 JSPatch 那么完善，但也足够应付一般场景，使用起来还比较轻量就好了，这也是本文要探讨的主题。

要达到这个目的，Native 层只要透出两种能力就基本可以了：

1.  在任意方法前后注入代码的能力，可能的话最好还能替换掉。
2.  调用任意类/实例方法的能力。

第 2 点不难，只要把 `[NSObject performSelector:...]` 那一套通过 `JSContext` 暴露出来即可。难的是第 1 点。其实细想一下，这不就是 AOP 么，而 iOS 有一个很方便的 AOP Library: [Aspects](https://github.com/steipete/Aspects)，只要把它的几个方法通过 JSContext 暴露给 JS 不就可以了么？

选择 Aspects 的原因是它已经经过了验证，不光是功能上的，更重要的是可以通过 AppStore 的审核。

> This is stable and used in hundreds of apps since it's part of PSPDFKit, an iOS PDF framework that ships with apps like Dropbox or Evernote.

Aspects 使用姿势：

```objc
[UIViewController aspect_hookSelector:@selector(viewWillAppear:) withOptions:AspectPositionAfter usingBlock:^(id<AspectInfo> aspectInfo, BOOL animated) {
    NSLog(@"View Controller %@ will appear animated: %tu", aspectInfo.instance, animated);
} error:NULL];
```

前插、后插、替换某个方法都可以。使用类的方式很简单，`NSClassFromString` 即可，Selector 也一样 `NSSelectorFromString`，这样就能通过外部传入 String，内部动态构造 Class 和 Selector 来达到 Fix 的效果了。

这种方式的安全性在于：

1.  不需要中间 JS 文件，准备工作全部在 Native 端完成。
2.  没有使用 App Store 不友好的类/方法。

### Demo

假设线上运行这这样一个 Class，由于疏忽，没有对参数做检查，导致特定情况下会 Crash。

```objc
@interface MightyCrash: NSObject
- (float)divideUsingDenominator:(NSInteger)denominator;
@end

@implementation MightyCrash
// 传一个 0 就 gg 了
- (float)divideUsingDenominator:(NSInteger)denominator
{
    return 1.f/denominator;
}
@end
```

现在我们要避免 Crash，就可以通过这种方式来修复

```objc
[Felix fixIt];

NSString *fixScriptString = @" \
fixInstanceMethodReplace('MightyCrash', 'divideUsingDenominator:', function(instance, originInvocation, originArguments){ \
    if (originArguments[0] == 0) { \
        console.log('zero goes here'); \
    } else { \
        runInvocation(originInvocation); \
    } \
}); \
\
";

[Felix evalString:fixScriptString];
```

运行一下看看

```objc
MightyCrash *mc = [[MightyCrash alloc] init];
float result = [mc divideUsingDenominator:3];
NSLog(@"result: %.3f", result);
result = [mc divideUsingDenominator:0];
NSLog(@"won't crash");

// output
// result: 0.333
// Javascript log: zero goes here
// won't crash
```

It Works, 是不是有那么点意思了。以下是可以正常运行的代码，仅供参考。

```objc
#import <Aspects.h>
#import <objc/runtime.h>
#import <JavaScriptCore/JavaScriptCore.h>

@interface Felix: NSObject
+ (void)fixIt;
+ (void)evalString:(NSString *)javascriptString;
@end


@implementation Felix
+ (Felix *)sharedInstance
{
    static Felix *sharedInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedInstance = [[self alloc] init];
    });

    return sharedInstance;
}

+ (void)evalString:(NSString *)javascriptString
{
    [[self context] evaluateScript:javascriptString];
}

+ (JSContext *)context
{
    static JSContext *_context;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _context = [[JSContext alloc] init];
        [_context setExceptionHandler:^(JSContext *context, JSValue *value) {
            NSLog(@"Oops: %@", value);
        }];
    });
    return _context;
}

+ (void)_fixWithMethod:(BOOL)isClassMethod aspectionOptions:(AspectOptions)option instanceName:(NSString *)instanceName selectorName:(NSString *)selectorName fixImpl:(JSValue *)fixImpl {
    Class klass = NSClassFromString(instanceName);
    if (isClassMethod) {
        klass = object_getClass(klass);
    }
    SEL sel = NSSelectorFromString(selectorName);
    [klass aspect_hookSelector:sel withOptions:option usingBlock:^(id<AspectInfo> aspectInfo){
        [fixImpl callWithArguments:@[aspectInfo.instance, aspectInfo.originalInvocation, aspectInfo.arguments]];
    } error:nil];
}

+ (id)_runClassWithClassName:(NSString *)className selector:(NSString *)selector obj1:(id)obj1 obj2:(id)obj2 {
    Class klass = NSClassFromString(className);
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
    return [klass performSelector:NSSelectorFromString(selector) withObject:obj1 withObject:obj2];
#pragma clang diagnostic pop
}

+ (id)_runInstanceWithInstance:(id)instance selector:(NSString *)selector obj1:(id)obj1 obj2:(id)obj2 {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
    return [instance performSelector:NSSelectorFromString(selector) withObject:obj1 withObject:obj2];
#pragma clang diagnostic pop
}

+ (void)fixIt
{
    [self context][@"fixInstanceMethodBefore"] = ^(NSString *instanceName, NSString *selectorName, JSValue *fixImpl) {
        [self _fixWithMethod:NO aspectionOptions:AspectPositionBefore instanceName:instanceName selectorName:selectorName fixImpl:fixImpl];
    };

    [self context][@"fixInstanceMethodReplace"] = ^(NSString *instanceName, NSString *selectorName, JSValue *fixImpl) {
        [self _fixWithMethod:NO aspectionOptions:AspectPositionInstead instanceName:instanceName selectorName:selectorName fixImpl:fixImpl];
    };

    [self context][@"fixInstanceMethodAfter"] = ^(NSString *instanceName, NSString *selectorName, JSValue *fixImpl) {
        [self _fixWithMethod:NO aspectionOptions:AspectPositionAfter instanceName:instanceName selectorName:selectorName fixImpl:fixImpl];
    };

    [self context][@"fixClassMethodBefore"] = ^(NSString *instanceName, NSString *selectorName, JSValue *fixImpl) {
        [self _fixWithMethod:YES aspectionOptions:AspectPositionBefore instanceName:instanceName selectorName:selectorName fixImpl:fixImpl];
    };

    [self context][@"fixClassMethodReplace"] = ^(NSString *instanceName, NSString *selectorName, JSValue *fixImpl) {
        [self _fixWithMethod:YES aspectionOptions:AspectPositionInstead instanceName:instanceName selectorName:selectorName fixImpl:fixImpl];
    };

    [self context][@"fixClassMethodAfter"] = ^(NSString *instanceName, NSString *selectorName, JSValue *fixImpl) {
        [self _fixWithMethod:YES aspectionOptions:AspectPositionAfter instanceName:instanceName selectorName:selectorName fixImpl:fixImpl];
    };

    [self context][@"runClassWithNoParamter"] = ^id(NSString *className, NSString *selectorName) {
        return [self _runClassWithClassName:className selector:selectorName obj1:nil obj2:nil];
    };

    [self context][@"runClassWith1Paramter"] = ^id(NSString *className, NSString *selectorName, id obj1) {
        return [self _runClassWithClassName:className selector:selectorName obj1:obj1 obj2:nil];
    };

    [self context][@"runClassWith2Paramters"] = ^id(NSString *className, NSString *selectorName, id obj1, id obj2) {
        return [self _runClassWithClassName:className selector:selectorName obj1:obj1 obj2:obj2];
    };

    [self context][@"runVoidClassWithNoParamter"] = ^(NSString *className, NSString *selectorName) {
        [self _runClassWithClassName:className selector:selectorName obj1:nil obj2:nil];
    };

    [self context][@"runVoidClassWith1Paramter"] = ^(NSString *className, NSString *selectorName, id obj1) {
        [self _runClassWithClassName:className selector:selectorName obj1:obj1 obj2:nil];
    };

    [self context][@"runVoidClassWith2Paramters"] = ^(NSString *className, NSString *selectorName, id obj1, id obj2) {
        [self _runClassWithClassName:className selector:selectorName obj1:obj1 obj2:obj2];
    };

    [self context][@"runInstanceWithNoParamter"] = ^id(id instance, NSString *selectorName) {
        return [self _runInstanceWithInstance:instance selector:selectorName obj1:nil obj2:nil];
    };

    [self context][@"runInstanceWith1Paramter"] = ^id(id instance, NSString *selectorName, id obj1) {
        return [self _runInstanceWithInstance:instance selector:selectorName obj1:obj1 obj2:nil];
    };

    [self context][@"runInstanceWith2Paramters"] = ^id(id instance, NSString *selectorName, id obj1, id obj2) {
        return [self _runInstanceWithInstance:instance selector:selectorName obj1:obj1 obj2:obj2];
    };

    [self context][@"runVoidInstanceWithNoParamter"] = ^(id instance, NSString *selectorName) {
        [self _runInstanceWithInstance:instance selector:selectorName obj1:nil obj2:nil];
    };

    [self context][@"runVoidInstanceWith1Paramter"] = ^(id instance, NSString *selectorName, id obj1) {
        [self _runInstanceWithInstance:instance selector:selectorName obj1:obj1 obj2:nil];
    };

    [self context][@"runVoidInstanceWith2Paramters"] = ^(id instance, NSString *selectorName, id obj1, id obj2) {
        [self _runInstanceWithInstance:instance selector:selectorName obj1:obj1 obj2:obj2];
    };

    [self context][@"runInvocation"] = ^(NSInvocation *invocation) {
        [invocation invoke];
    };

    // helper
    [[self context] evaluateScript:@"var console = {}"];
    [self context][@"console"][@"log"] = ^(id message) {
        NSLog(@"Javascript log: %@",message);
    };
}
@end
```
