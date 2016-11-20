---
layout: post
title: 说说Core Foundation
category: tech
tag: 技术
---

先来说说「Core Foundation」（以下简称CF）的历史吧。当年乔布斯被自己创办的公司驱逐后，成立了「NeXT Computer」,其实做的还是老本行：卖电脑，但依旧不景气。好在NeXTSTEP系统表现还不错，亏损不至于太严重。正好此时苹果的市场份额大跌，急需一个新的操作系统，结果大家都知道了，乔布斯借此收购，重新回到了苹果。

这里就牵扯到了一个问题，如何让旧有的系统（Mac OS 9）和NeXTSTEP合成为一个新系统？这就需要一个更为底层的核心库可以供Mac Toolbox和OPENSTEP双方调用。CF就这么诞生了。

CF是由C语言实现的，而不是Objective-C，所以如果用到了CF，就需要手动管理内存，ARC是无能为力的。当然因为CF和Foundation之间的友好关系，它们之间的管理权也是可以移交的，这个后面再说。

CF提供了基础功能，如CFString,CFDate,CFNumber等等，以CFString为例，CFString和NSString之间是什么关系？NSString其实是一个「类簇」，也就是抽象接口，所以String Objects并不是NSString实例，而是实现了NSString方法的私有类的实例，也就是CFString。

{% highlight objective-c %}
NSLog(NSStringFromClass([@"Some Class" class]));

# output __NSCFConstantString 
{% endhighlight %}

同时NSStrings和CFStrings之间可以自由转换，也就是「toll free bridging」。比如：

{% highlight objective-c %}
CFStringRef aCFString = (CFStringRef)aNSString;  
NSString *aNSString = (NSString *)aCFString;  
{% endhighlight %}

因为编译器无法自动管理CF的内存，所以CF对象在使用完后，需要手动释放（CFRelease）。如果使用ARC来管理内存，苹果提供了3种方法来处理：

### __bridge

__bridge只是在CF和OC之间传递指针，其他的事啥也没干，所以转换成CF时，还是要手动释放内存。

{% highlight objective-c %}
CFStringRef aCFString = CFStringCreateWithCString(NULL, "test", kCFStringEncodingASCII);  
NSString *aNSString = (__bridge NSString *)aCFString;  
      
(void)aNSString;  
      
CFRelease(aCFString);
{% endhighlight %}

### __bridge_retained

__bridge_retained或者CFBridgingRetain()，将Objective-C对象转换为Core Foundation对象，把对象所有权桥接给Core Foundation对象，同时剥夺ARC的管理权，后续需要开发者使用CFRelease或者相关方法手动来释放对象。

### __bridge_transfer

__bridge_transfer 或者 CFBridgingRelease()  将非Objective-C对象转换为Objective-C对象，同时将对象的管理权交给ARC，开发者无需手动管理内存。

<h2></h2>
最后，因为CF是用C实现的，且处于下层，所以执行速度上会比Foundation稍微快一点，不过也就是一点点，几乎察觉不到。相比Foundation带来的ARC内存管理和更多的API，开发上的效率会大幅提升，所以还是尽量多的使用OC。

### 参考

* [http://ridiculousfish.com/blog/posts/bridge.html](http://ridiculousfish.com/blog/posts/bridge.html)
* [http://blog.csdn.net/yiyaaixuexi/article/details/8553659](http://blog.csdn.net/yiyaaixuexi/article/details/8553659)
