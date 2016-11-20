---
layout: post
title: 简单说说iOS的图形和动画
category: tech
tag: 技术
---

### Core Graphics

Core Graphics是一组用来绘制2D图形的API，使用CPU进行计算。 新建一个项目时，模板已经自动载入了CoreGraphics.framwork。

### Core Animation

Core Animation包含于QuartzCore.framwork中，是一组自由度更大的图形绘制和动画API，但实现起来也会比Core Graphics麻烦一点。iOS上的UIKit和动画效果大部分都是通过Core Animation实现的。

### Core Image

Core Image是一组用于图像、视频处理的API，如添加滤镜之类的。

### OpenGL / OpenGL ES

底层的图形绘制API，自由度最大，但学习成本也很高。如果不是做大型游戏，推荐使用更高层的API。

### 硬件加速

硬件加速是指用到了GPU的API，以下这些情况不会用到硬件加速

* 所有在drawRect中完成的图形绘制。
* shouldRasterize属性为YES的CALayer。
* 用到了mask或drop shadow的CALayer。
* Text (包括UILabels, CATextLayers, Core Text, 等等)。
* 使用CGContexts绘制的图形

## Core Animation

虽然是Animation，但实际上它也干Drawing的活，这就需要CALayer的帮助。iOS中，所有的UIView都自带了一个CALayer（可以通过UIView.layer访问），UIView的渲染和动画最终也是通过layer来实现的。从这个意义上说，UIView就是简单的一层壳，把图形绘制需要的信息传递给layer。当然UIView还有一个重要的功能就是处理事件，如点击按钮，滑动等等。

事实上layer也是一层壳(Model Tree)，背后还有呈现树(Presenting Tree)和渲染树(Render Tree)，渲染树对呈现树的数据进行渲染。

跟view一样，layer也存在着一个树状结构。可以直接创建，或通过view.layer获取。

layer有很多的动画属性，如anchorPoint(view没有这个属性)、frame、transform等等，详细的属性列表[见此](http://developer.apple.com/library/ios/#documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html#//apple_ref/doc/uid/TP40004514-CH11-SW1)。配合Core Animation的API可以实现很多UIView Animation无法实现的效果，比如3D动画。

## UIView Animation

这个是我们经常会用到的，它对Core Animation做了更高层的封装，方便使用，当然自由度也降低了。

{% highlight objective-c %}
+ (void)animateWithDuration:(NSTimeInterval)duration delay:(NSTimeInterval)delay options:(UIViewAnimationOptions)options animations:(void (^)(void))animations completion:(void (^)(BOOL finished))completion
{% endhighlight %}

animation block里可以设置view的动画属性的终止值，如frame, rotation等。options可以设置动画的相关选项，如下：

{% highlight objective-c %}
enum {
    UIViewAnimationOptionLayoutSubviews            = 1 <<  0,
    UIViewAnimationOptionAllowUserInteraction      = 1 <<  1,
    UIViewAnimationOptionBeginFromCurrentState     = 1 <<  2,
    UIViewAnimationOptionRepeat                    = 1 <<  3,
    UIViewAnimationOptionAutoreverse               = 1 <<  4,
    UIViewAnimationOptionOverrideInheritedDuration = 1 <<  5,
    UIViewAnimationOptionOverrideInheritedCurve    = 1 <<  6,
    UIViewAnimationOptionAllowAnimatedContent      = 1 <<  7,
    UIViewAnimationOptionShowHideTransitionViews   = 1 <<  8,
 
    UIViewAnimationOptionCurveEaseInOut            = 0 << 16,
    UIViewAnimationOptionCurveEaseIn               = 1 << 16,
    UIViewAnimationOptionCurveEaseOut              = 2 << 16,
    UIViewAnimationOptionCurveLinear               = 3 << 16,
 
    UIViewAnimationOptionTransitionNone            = 0 << 20,
    UIViewAnimationOptionTransitionFlipFromLeft    = 1 << 20,
    UIViewAnimationOptionTransitionFlipFromRight   = 2 << 20,
    UIViewAnimationOptionTransitionCurlUp          = 3 << 20,
    UIViewAnimationOptionTransitionCurlDown        = 4 << 20,
    UIViewAnimationOptionTransitionCrossDissolve   = 5 << 20,
    UIViewAnimationOptionTransitionFlipFromTop     = 6 << 20,
    UIViewAnimationOptionTransitionFlipFromBottom  = 7 << 20,
};
typedef NSUInteger UIViewAnimationOptions;
{% endhighlight %}

所以一般的动画view animation都可以应付。

## TableView优化

TableView是iOS中非常重要的组成部分，如果处理不当，就很容易出现不流畅的现象。比如一个TableViewCell中有多个subview。上面说过一个view对应了一个layer，多个view自然也就对应多个layer，好比photoshop的图层。滑动时GPU需要分别对每一个layer进行处理，如果不能在短时间内完成，就容易掉帧。

要保证TableView的流畅，首先TableViewCell的生成时间要短（少于1/60秒），其次移动时帧频尽量保持在60（也就是每秒60帧）。前者取决于CPU，后者取决于GPU。

以twitter为例，可以通过subviews来实现，不过性能会有点影响，但实现起来简单。

![twitter TableViewCell](/image/twitter_tvc_subviews.jpg) 

因为cell在形态上不会经常改变，所以也可以通过drawRect直接绘制，只要这个时间足够短就可以。好处是layer不用处理多个子layer的组合和叠加，就像一张jpg图片一样，滑动会更流畅。

![twitter TableViewCell](/image/twitter_tvc_drawrect.png) 

### 参考

* [http://geeklu.com/2012/09/animation-in-ios/](http://geeklu.com/2012/09/animation-in-ios/)
* [http://robots.thoughtbot.com/post/33427366406/designing-for-ios-taming-uibutton](http://robots.thoughtbot.com/post/33427366406/designing-for-ios-taming-uibutton)
* [https://news.ycombinator.com/item?id=4645585](https://news.ycombinator.com/item?id=4645585)
* [http://stackoverflow.com/q/6731545/94962](http://stackoverflow.com/q/6731545/94962)
* [http://giorgiocalderolla.com/blog.html#customizing-uitableviewcells-a-better-way](http://giorgiocalderolla.com/blog.html#customizing-uitableviewcells-a-better-way)
* [https://blog.twitter.com/2012/simple-strategies-smooth-animation-iphone](https://blog.twitter.com/2012/simple-strategies-smooth-animation-iphone)
* [layer trees vs flat drawing graphics performance across ios device generations](http://floriankugler.com/blog/2013/5/24/layer-trees-vs-flat-drawing-graphics-performance-across-ios-device-generations)
