---
layout: post
title: (译)Objective-C的动态特性
category: tech
tag: 技术
---

这是一篇译文，原文[在此](http://pilky.me/view/21)，上一篇文章就是受这篇文章启发，这次干脆都翻译过来。

过去的几年中涌现了大量的Objective-C开发者。有些是从动态语言转过来的，比如Ruby或Python，有些是从强类型语言转过来的，如Java或C#，当然也有直接以Objective-C作为入门语言的。也就是说有很大一部分开发者都没有使用Objective-C太长时间。当你接触一门新语言时，更多地会关注基础知识，如语法和特性等。但通常有一些更高级的，更鲜为人知又有强大功能的特性等待你去开拓。

这篇文章主要是来领略下Objective-C的运行时(runtime)，同时解释是什么让Objective-C如此动态，然后感受下这些动态化的技术细节。希望这回让你对Objective-C和Cocoa是如何运行的有更好的了解。

##The Runtime
Objective-C是一门简单的语言，95%是C。只是在语言层面上加了些关键字和语法。真正让Objective-C如此强大的是它的运行时。它很小但却很强大。它的核心是消息分发。

###Messages
如果你是从动态语言如Ruby或Python转过来的，可能知道什么是消息，可以直接跳过进入下一节。那些从其他语言转过来的，继续看。

执行一个方法，有些语言，编译器会执行一些额外的优化和错误检查，因为调用关系很直接也很明显。但对于消息分发来说，就不那么明显了。在发消息前不必知道某个对象是否能够处理消息。你把消息发给它，它可能会处理，也可能转给其他的Object来处理。一个消息不必对应一个方法，一个对象可能实现一个方法来处理多条消息。

在Objective-C中，消息是通过`objc_msgSend()`这个runtime方法及相近的方法来实现的。这个方法需要一个target，selector，还有一些参数。理论上来说，编译器只是把消息分发变成`objc_msgSend`来执行。比如下面这两行代码是等价的。

{% highlight objc %}
[array insertObject:foo atIndex:5];
objc_msgSend(array, @selector(insertObject:atIndex:), foo, 5);
{% endhighlight %}

###Objects, Classes, MetaClasses
大多数面向对象的语言里有 classes 和 objects 的概念。Objects通过Classes生成。但是在Objective-C中，classes本身也是objects(译者注：这点跟python很像)，也可以处理消息，这也是为什么会有类方法和实例方法。具体来说，Objective-C中的Object是一个结构体(struct)，第一个成员是`isa`，指向自己的class。这是在objc/objc.h中定义的。

{% highlight objc %}
typedef struct objc_object {
	Class isa;
} *id;
{% endhighlight %}

object的class保存了方法列表，还有指向父类的指针。但classes也是objects，也会有`isa`变量，那么它又指向哪儿呢？这里就引出了第三个类型: `metaclasses`。一个 metaclass被指向class，class被指向object。它保存了所有实现的方法列表，以及父类的metaclass。如果想更清楚地了解objects,classes以及metaclasses是如何一起工作地，可以阅读[这篇文章](http://www.sealiesoftware.com/blog/archive/2009/04/14/objc_explain_Classes_and_metaclasses.html)。

###Methods, Selectors and IMPs
我们知道了运行时会发消息给对象。我们也知道一个对象的class保存了方法列表。那么这些消息是如何映射到方法的，这些方法又是如何被执行的呢？

第一个问题的答案很简单。class的方法列表其实是一个字典，key为selectors，IMPs为value。一个IMP是指向方法在内存中的实现。很重要的一点是，selector和IMP之间的关系是在运行时才决定的，而不是编译时。这样我们就能玩出些花样。

IMP通常是指向方法的指针，第一个参数是self，类型为id，第二个参数是_cmd，类型为SEL，余下的是方法的参数。这也是`self`和`_cmd`被定义的地方。下面演示了Method和IMP

{% highlight objc %}
- (id)doSomethingWithInt:(int)aInt{}

id doSomethingWithInt(id self, SEL _cmd, int aInt){}
{% endhighlight %}

###其他运行时的方法
现在我们知道了objects,classes,selectors,IMPs以及消息分发，那么运行时到底能做什么呢？主要有两个作用：

1. 创建、修改、自省classes和objects
2. 消息分发

之前已经提过消息分发，不过这只是一小部分功能。所有的运行时方法都有特定的前缀。下面是一些有意思的方法：

####class
class开头的方法是用来修改和自省classes。方法如`class_addIvar`, `class_addMethod`, `class_addProperty`和`class_addProtocol`允许重建classes。`class_copyIvarList`, `class_copyMethodList`, `class_copyProtocolList`和`class_copyPropertyList`能拿到一个class的所有内容。而`class_getClassMethod`, `class_getClassVariable`, `class_getInstanceMethod`, `class_getInstanceVariable`, `class_getMethodImplementation`和`class_getProperty`返回单个内容。

也有一些通用的自省方法，如`class_conformsToProtocol`, `class_respondsToSelector`, `class_getSuperclass`。最后，你可以使用`class_createInstance`来创建一个object。

####ivar
这些方法能让你得到名字，内存地址和Objective-C type encoding。

####method
这些方法主要用来自省，比如`method_getName`, `method_getImplementation`, `method_getReturnType`等等。也有一些修改的方法，包括`method_setImplementation`和`method_exchangeImplementations`，这些我们后面会讲到。

####objc
一旦拿到了object，你就可以对它做一些自省和修改。你可以get/set ivar, 使用`object_copy`和`object_dispose`来copy和free object的内存。最NB的不仅是拿到一个class，而是可以使用`object_setClass`来改变一个object的class。待会就能看到使用场景。

####property
属性保存了很大一部分信息。除了拿到名字，你还可以使用`property_getAttributes`来发现property的更多信息，如返回值、是否为atomic、getter/setter名字、是否为dynamic、背后使用的ivar名字、是否为弱引用。

####protocol
Protocols有点像classes，但是精简版的，运行时的方法是一样的。你可以获取method, property, protocol列表, 检查是否实现了其他的protocol。 

####sel
最后我们有一些方法可以处理 selectors，比如获取名字，注册一个selector等等。


现在我们对Objective-C的运行时有了大概的了解，来看看它们能做哪些有趣的事情。

##Classes And Selectors From Strings
比较基础的一个动态特性是通过String来生成Classes和Selectors。Cocoa提供了`NSClassFromString`和`NSSelectorFromString`方法，使用起来很简单：

{% highlight objc %}
Class stringclass = NSClassFromString(@"NSString");
{% endhighlight %}

于是我们就得到了一个string class。接下来：

{% highlight objc %}
NSString *myString = [stringclass stringWithString:@"Hello World"];
{% endhighlight %}

为什么要这么做呢？直接使用Class不是更方便？通常情况下是，但有些场景下这个方法会很有用。首先，可以得知是否存在某个class，`NSClassFromString` 会返回nil，如果运行时不存在该class的话。比如可以检查`NSClassFromString(@"NSRegularExpression")`是否为nil来判断是否为iOS4.0+。

另一个使用场景是根据不同的输入返回不同的class或method。比如你在解析一些数据，每个数据项都有要解析的字符串以及自身的类型（String，Number，Array）。你可以在一个方法里搞定这些，也可以使用多个方法。其中一个方法是获取type，然后使用if来调用匹配的方法。另一种是根据type来生成一个selector，然后调用之。以下是两种实现方式：

{% highlight objc %}
- (void)parseObject:(id)object {
    for (id data in object) {
        if ([[data type] isEqualToString:@"String"]) {
            [self parseString:[data value]]; 
        } else if ([[data type] isEqualToString:@"Number"]) {
            [self parseNumber:[data value]];
        } else if ([[data type] isEqualToString:@"Array"]) {
            [self parseArray:[data value]];
        }
    }
}
- (void)parseObjectDynamic:(id)object {
    for (id data in object) {
    	[self performSelector:NSSelectorFromString([NSString stringWithFormat:@"parse%@:", [data type]]) withObject:[data value]];
    }
}
- (void)parseString:(NSString *)aString {}
- (void)parseNumber:(NSString *)aNumber {}
- (void)parseArray:(NSString *)aArray {}
{% endhighlight %}

可一看到，你可以把7行带if的代码变成1行。将来如果有新的类型，只需增加实现方法即可，而不用再去添加新的 else if。

##Method Swizzling
之前我们讲过，方法由两个部分组成。Selector相当于一个方法的id；IMP是方法的实现。这样分开的一个便利之处是selector和IMP之间的对应关系可以被改变。比如一个 IMP 可以有多个 selectors 指向它。

而 Method Swizzling 可以交换两个方法的实现。或许你会问“什么情况下会需要这个呢？”。我们先来看下Objective-C中，两种扩展class的途径。首先是 subclassing。你可以重写某个方法，调用父类的实现，这也意味着你必须使用这个subclass的实例，但如果继承了某个Cocoa class，而Cocoa又返回了原先的class(比如 NSArray)。这种情况下，你会想添加一个方法到NSArray，也就是使用Category。99%的情况下这是OK的，但如果你重写了某个方法，就没有机会再调用原先的实现了。

Method Swizzling 可以搞定这个问题。你可以重写某个方法而不用继承，同时还可以调用原先的实现。通常的做法是在category中添加一个方法(当然也可以是一个全新的class)。可以通过`method_exchangeImplementations`这个运行时方法来交换实现。来看一个demo，这个demo演示了如何重写`addObject:`方法来纪录每一个新添加的对象。

{% highlight objc %}
#import  <objc/runtime.h>

@interface NSMutableArray (LoggingAddObject)
- (void)logAddObject:(id)aObject;
@end

@implementation NSMutableArray (LoggingAddObject)

+ (void)load {
    Method addobject = class_getInstanceMethod(self, @selector(addObject:));
    Method logAddobject = class_getInstanceMethod(self, @selector(logAddObject:));
    method_exchangeImplementations(addObject, logAddObject);
}

- (void)logAddObject:(id)aobject {
    [self logAddObject:aObject];
    NSLog(@"Added object %@ to array %@", aObject, self);
}

@end
{% endhighlight %}

我们把方法交换放到了`load`中，这个方法只会被调用一次，而且是运行时载入。如果指向临时用一下，可以放到别的地方。注意到一个很明显的递归调用`logAddObject:`。这也是Method Swizzling容易把我们搞混的地方，因为我们已经交换了方法的实现，所以其实调用的是`addObject:`

![Method Swizzling](http://pilky.me/static/blogmedia/objcdynamictips_methodswizzling.png)

##动态继承、交换
我们可以在运行时创建新的class，这个特性用得不多，但其实它还是很强大的。你能通过它创建新的子类，并添加新的方法。

但这样的一个子类有什么用呢？别忘了Objective-C的一个关键点：object内部有一个叫做`isa`的变量指向它的class。这个变量可以被改变，而不需要重新创建。然后就可以添加新的ivar和方法了。可以通过以下命令来修改一个object的class

{% highlight objc %}
object_setClass(myObject, [MySubclass class]);
{% endhighlight %}

这可以用在Key Value Observing。当你开始observing an object时，Cocoa会创建这个object的class的subclass，然后将这个object的isa指向新创建的subclass。点击[这里](http://www.mikeash.com/pyblog/friday-qa-2009-01-23.html)查看更详细的解释。

##动态方法处理
目前为止，我们讨论了方法交换，以及已有方法的处理。那么当你发送了一个object无法处理的消息时会发生什么呢？很明显，"it breaks"。大多数情况下确实如此，但Cocoa和runtime也提供了一些应对方法。

首先是**动态方法处理**。通常来说，处理一个方法，运行时寻找匹配的selector然后执行之。有时，你只想在运行时才创建某个方法，比如有些信息只有在运行时才能得到。要实现这个效果，你需要重写`+resolveInstanceMethod:` 和/或 `+resolveClassMethod:`。如果确实增加了一个方法，记得返回YES。

{% highlight objc %}
+ (BOOL)resolveInstanceMethod:(SEL)aSelector {
    if (aSelector == @selector(myDynamicMethod)) {
        class_addMethod(self, aSelector, (IMP)myDynamicIMP, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:aSelector];
}
{% endhighlight %}

那Cocoa在什么场景下会使用这些方法呢？Core Data用得很多。NSManagedObjects有许多在运行时添加的属性用来处理get/set属性和关系。那如果Model在运行时被改变了呢？

##消息转发
如果 resolve method 返回NO，运行时就进入下一步骤：消息转发。有两种常见用例。1) 将消息转发到另一个可以处理该消息的object。2) 将多个消息转发到同一个方法。

消息转发分两步。首先，运行时调用`-forwardingTargetForSelector:`，如果只是想把消息发送到另一个object，那么就使用这个方法，因为更高效。如果想要修改消息，那么就要使用`-forwardInvocation:`，运行时将消息打包成NSInvocation，然后返回给你处理。处理完之后，调用`invokeWithTarget:`。

Cocoa有几处地方用到了消息转发，主要的两个地方是代理(Proxies)和响应链(Responder Chain)。NSProxy是一个轻量级的class，它的作用就是转发消息到另一个object。如果想要惰性加载object的某个属性会很有用。NSUndoManager也有用到，不过是截取消息，之后再执行，而不是转发到其他的地方。

响应链是关于Cocoa如何处理与发送事件与行为到对应的对象。比如说，使用Cmd+C执行了copy命令，会发送`-copy:`到响应链。首先是First Responder，通常是当前的UI。如果没有处理该消息，则转发到下一个`-nextResponder`。这么一直下去直到找到能够处理该消息的object，或者没有找到，报错。

##使用Block作为Method IMP
iOS 4.3带来了很多新的runtime方法。除了对properties和protocols的加强，还带来一组新的以 imp 开头的方法。通常一个 IMP 是一个指向方法实现的指针，头两个参数为 object(self)和selector(_cmd)。iOS 4.0和Mac OS X 10.6 带来了block，`imp_implementationWithBlock()` 能让我们使用block作为 IMP，下面这个代码片段展示了如何使用block来添加新的方法。

{% highlight objc %}
IMP myIMP = imp_implementationWithBlock(^(id _self, NSString *string) {
	NSLog(@"Hello %@", string);
});
class_addMethod([MYclass class], @selector(sayHello:), myIMP, "v@:@");
{% endhighlight %}

如果想知道这是如何实现的，可以查看[这篇文章](http://www.friday.com/bbum/2011/03/17/ios-4-3-imp_implementationwithblock/)

可以看到，Objective-C 表面看起来挺简单，但还是很灵活的，可以带来很多可能性。动态语言的优势在于在不扩展语言本身的情况下做很多很灵巧的事情。比如Key Value Observing，提供了优雅的API可以与已有的代码无缝结合，而不需要新增语言级别的特性。

希望这篇文章能让你更深入地了解Objective-C，在开发app时也能开阔思路，考虑更多的可能性。
