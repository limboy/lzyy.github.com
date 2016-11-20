---
layout: post
title: 类簇在iOS开发中的应用
category: tech
tag: 技术
---

类簇(class cluster)是一种设计模式，在Foundation Framework中被广泛使用，举个简单的例子

{% highlight objc %}
NSArray *arr = [NSArray arrayWithObjects:@"foo",@"bar", nil];
NSLog(@"arr class:%@", [arr class]);
// output: __NSArrayI
{% endhighlight %}

显然`__NSArrayI`是一个私有类，来看看这个类的头文件

{% highlight objc %}
@interface __NSArrayI : NSArray {
    unsigned int _used;
}

//...
{% endhighlight %}

可以看出`__NSArrayI`继承了`NSArray`。为什么要这么设计呢？拿NSNumber来举个例子，我们都知道NSNumber可以存储多种类型的数字，如Int/Float/Double等等，一种方式是把NSNumber作为基类，然后分别去实现各自的子类，像这样：

![](https://developer.apple.com/library/ios/documentation/general/conceptual/CocoaEncyclopedia/Art/cluster1.gif)

初看起来也没什么问题，但如果子类很多，像这样：

![](https://developer.apple.com/library/ios/documentation/general/conceptual/CocoaEncyclopedia/Art/cluster2.gif)

这对使用者来说显然不够方便，得记住这么多类。如果使用类簇，问题就变得简单了，把Number作为抽象基类，子类各自实现存取方式，然后在基类中定义多个初始化方式，像这样：

![](https://developer.apple.com/library/ios/documentation/general/conceptual/CocoaEncyclopedia/Art/cluster3.gif)

现在只需要记住一个类就可以了。`NSNumber`的初始化伪代码大概像这样：

{% highlight objc %}
- (id)initWithBool
{
	return [[__NSCFBoolean alloc]init];
}

- (id)initWithLong
{
	return [[__NSCFNumber alloc]init];
}

//...
{% endhighlight %}

### 在iOS项目中的应用

在开发app时经常会遇到表现和行为完全一样，但数据源不一样的情况。以花瓣app为例，同样是瀑布流，可能来自我喜欢的图片、某个画板下的图片、某个用户的图片等等。如果为每一种表现方式都新建一个Controller，并且使用这个Controller来初始化，那么就会遇到最开始提到的问题：子类太多，使用不便。这正好可以通过类簇来很方便地搞定。比如这样：

{% highlight objc %}
@implementation HBWaterfallViewController
- (id)initWithLiked
{
	return [[HBLikedViewController alloc]init];
}

- (id)initWithBoardID:(NSInteger)boardID
{
	return [[HBBoardViewController alloc]initWithBoardID:boardID];
}

#pragma mark - 通用的方法

- (PSUICollectionViewCell *)collectionView:(PSUICollectionView *)collectionView
                    cellForItemAtIndexPath:(NSIndexPath *)indexPath
{
	// ...
}

// ...

#pragma mark - 每个子类需要实现的方法

- (void)fetchMoreData
{
    NSAssert(NO, @"子类需要实现此方法");
}

{% endhighlight %}

使用起来类似这样`[[HBWaterfallViewController alloc]initWithBoardID:9527]` 或 `[[HBWaterfallViewController alloc]initWithLiked]`。如果有新的DataSource，新加一个初始化方法即可，对于使用者来说，打开头文件，看下init开头的方法就行了。

再举个例子，现在很多应用需要同时兼顾iOS6和iOS7，在表现上需要为不同的系统加载不同的图片资源，最简单粗暴的方法就是各种if/else判断，像这样：

{% highlight objc %}
if ([[UIDevice currentDevice]systemMajorVersion] < 7)
{
    /* iOS 6 and previous versions */
}
else
{
    /* iOS 7 and above */
}
{% endhighlight %}

不够优雅，可以使用类簇的思想来去掉if/else判断，把跟视图具体元素无关的代码放在基类，跟系统版本相关的代码拆成两个子类，然后在各自的类中加载相应的资源。

{% highlight objc %}
/* TestView.h */
@interface TestView: UIView

/* Common method */
- ( void )test;

@end

/* TestView.m */
@implementation TestView

+ (id)alloc
{
    if ([self class]== [TestView class])
    {
        if ([[UIDevice currentDevice] systemMajorVersion] < 7)
        {
            return [TestViewIOS6 alloc];
        }
        else
        {
            return [TestViewIOS7 alloc];
        }
    }
    else
    {
        return [super alloc];
    }
}

- ( void )test
{}

@end
{% endhighlight %}

这里`alloc`时并没有返回`TestView`类，而是根据系统版本返回`TestViewIOS6` 或 `TestViewIOS7`。

{% highlight objc %}
/* TestViewIOS6.m */
@implementation TestViewIOS6: TestView

- (void)drawRect: (CGRect)rect
{
    /* Custom iOS6 drawing code */
}

@end

/* TestViewIOS7.m */
@implementation TestViewIOS7

- (void)drawRect: (CGRect)rect
{
    /* Custom iOS7 drawing code */
}

@end
{% endhighlight %}

### 小结

类簇的本质其实是[抽象工厂](http://zh.wikipedia.org/wiki/%E6%8A%BD%E8%B1%A1%E5%B7%A5%E5%8E%82)，类簇也可以有多个基类，如`NSArray`, `NSMutableArray`, 后者就是继承的前者。它对一些「大同小异」的问题，往往会有不错的效果。

### 参考

* [Cocoa Core Competencies](https://developer.apple.com/library/mac/documentation/general/conceptual/devpedia-cocoacore/ClassCluster.html)
* [Strategies to support iOS7 UI](http://www.noxeos.com/2013/06/18/strategies-support-ios7-ui/)
* [Class Cluster Concepts in Objective-C](https://developer.apple.com/library/ios/documentation/general/conceptual/CocoaEncyclopedia/ClassClusters/ClassClusters.html)
