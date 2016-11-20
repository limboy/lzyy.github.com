---
layout: post
title: 阅读源码的乐趣
category: tech
tag: 技术
---

阅读源码尤其是优秀的源码是一件很有乐趣的事情，可以拓宽视野，提高品位，锻炼思维，就像间接地在跟作者沟通一样。Quora 上有一个问题是：[TJ-Holowaychunk是如何学习编程的](http://www.quora.com/How-did-TJ-Holowaychuk-learn-to-program)，他的回答是

>>> I don't read books, never went to school, I just read other people's code and always wonder how things work

如果有足够的好奇心，并且总想知道「How Things Work」，那么阅读源码就是个不错的途径。

源码的复杂度不同，需要投入的时间、使用的方法也不同，以一个中等复杂度的项目为例，简单分享下我阅读源码的一些经验。 

WWDC 2014，有一个 Session 是讲[「Advanced User Interfaces with Collection Views」](https://developer.apple.com/videos/wwdc/2014/#232)，之所以选择这个，是因为它是我们还算熟悉的对象（Collection View），但苹果用了一些「特殊」的架构来做到代码复用，并且减少 VC 的体积，而且使用了部分 iTunes Connect 的源码，而不是简单的演示代码。所以决定一窥究竟。

为了有一个大概的感受，先看一遍视频，不需要领会每个要点，先记录一些关键信息，方便到时翻源码。

* 这套结构可以处理复杂的 DataSource
* 可以同时适配 iPhone / iPad
* 有一个统一的 loading indicator
* 可以设置某个 Header 是否置顶
* 可以有一个全局的 Header
* 通过聚合 DataSource 的方法来达到代码复用，并且只有一个 VC
* 可以设置聚合形式为 Segmented / Composed
* layout信息可以配置，且可以覆盖
* 使用了有限状态机
* 子 DataSource 在数据载入完成后会有一个 block，所需的 DataSource 都载入完成时，这些 block 会被统一执行
* Section Metrics 可以设置 Section 的具体表现
* layout 的信息会在内部被保存，避免重复计算 (Snapshot Metrics)
* Optional Layout Methods 会有意想不到的好效果

产生了一些疑问，比如 

* 多个子 DataSource 被组合成一个 Composed DataSource 时，如何通过 IndexPath 找到对应的 DataSource？
* 找到之后如何处理？
* 是否置顶是如何实现的？
* 如何通过有限状态机来管理 Loading 状态？
* 如果有按钮，那么按钮的点击事件如何处理？ 
* Collection View 没有 headerView，这又是怎么实现的？
* 数据是怎么载入的？

大概有了些概念和疑问之后，就可以打开源码痛快看了，先来看看目录结构 (可以在这里[在线浏览](https://github.com/zwaldowski/AAPLAdvancedCollectionView))

{% highlight objc %}
|- Framework
	|- Categories
	|- DataSources
	|- Layouts
	|- ViewControllers
	|- Views
|- Application
{% endhighlight %}

看来关键的信息都在 Framework 里了，那如何切入呢？反其道而行之，先来看看这些 Framework 是怎么用的，最直接的就从 ViewController 入手。那就先来看看 AAPLCatListViewController 这个类吧，如果没猜错的话，应该是展示喵咪列表（直观的名字很重要）。

果然很小，居然只有 140 行，如果不分离的话，1400 行也是可以轻松达到的。看到定义了一个 AAPLSegmentedDataSource，脑海里大概可以想象出是一个可以切换 Tag 的页面，接着又看到了两个 DataSource，那这两个页面的数据源应该就是它们了。

{% highlight objc %}
@interface APPLCatListViewController ()
@property (nonatomic, strong) AAPLSegmentedDataSource *segmentedDataSource;
@property (nonatomic, strong) AAPLCatListDataSource *catsDataSource;
@property (nonatomic, strong) AAPLCatListDataSource *favoriteCatsDataSource;
@property (nonatomic, strong) NSIndexPath *selectedIndexPath;
@property (nonatomic, strong) id selectedDataSourceObserver;
@end
{% endhighlight %}

然后又看到这么一行

{% highlight objc %}
- (void)dealloc
{
    [self.segmentedDataSource aapl_removeObserver:self.selectedDataSourceObserver];
}
{% endhighlight %}

看起来是苹果自己实现了一个 KVO Wrapper，果然他们自己也无法忍受原生的KVO，哈哈。接着到了 ViewDidLoad，新建了两个 DataSource，那新建的时候都干了些什么？

{% highlight objc %}
- (AAPLCatListDataSource *)newAllCatsDataSource
{
    AAPLCatListDataSource *dataSource = [[AAPLCatListDataSource alloc] init];
    dataSource.showingFavorites = NO;

    dataSource.title = NSLocalizedString(@"All", @"Title for available cats list");
    dataSource.noContentMessage = NSLocalizedString(@"All the big ...", @"The message to show when no cats are available");
    dataSource.noContentTitle = NSLocalizedString(@"No Cats", @"The title to show when no cats are available");
    dataSource.errorMessage = NSLocalizedString(@"A problem with the network ....", @"Message to show when unable to load cats");
    dataSource.errorTitle = NSLocalizedString(@"Unable To Load Cats", @"Title of message to show when unable to load cats");

    return dataSource;
}
{% endhighlight %}

所以只是初始化，然后设置一些信息，Nothing Special。然后看到了 AAPLLayoutSectionMetrics ，看起来是设置 Layout 的一些显示信息，如 height / backgroundColor 之类的。

最后创建了一个 KVO 来监测 selectedDataSource 的变化，界面上做相应的调整。

接下来看看 AAPLCatListDataSource 的实现，一进去发现

{% highlight objc %}
@interface AAPLCatListDataSource : AAPLBasicDataSource
/// Is this list showing the favorites or all available cats?
@property (nonatomic) BOOL showingFavorites;
@end
{% endhighlight %}

看来 AAPLBasicDataSource 一定做了很多事，进入到 AAPLBasicDataSource.m 文件，看到这个方法

{% highlight objc %}
- (void)setShowingFavorites:(BOOL)showingFavorites
{
    if (showingFavorites == _showingFavorites)
        return;

    _showingFavorites = showingFavorites;
    [self resetContent];
    [self setNeedsLoadContent];

    if (showingFavorites)
        [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(observeFavoriteToggledNotification:) name:AAPLCatFavoriteToggledNotificationName object:nil];
}
{% endhighlight %}

注意到有一个 `setNeedsLoadContent` 方法，看起来数据的载入应该是通过这个方法来触发的，进去看看

{% highlight objc %}
- (void)setNeedsLoadContent
{
    [NSObject cancelPreviousPerformRequestsWithTarget:self selector:@selector(loadContent) object:nil];
    [self performSelector:@selector(loadContent) withObject:nil afterDelay:0];
}
{% endhighlight %}

第一个方法没怎么接触过，查一下文档先，原来是可以取消之前通过 `performSelector:withObject:afterDelay:` 触发的方法，为了加深印象，顺便 Google 一下这个方法，原来 `performSelector:withObject:afterDelay` 在方法被执行前，会持有 Object，方法执行后在解除对 Object 的持有，如果不小心多次调用这个方法就有可能导致内存泄露，所以在调用此方法前先 cancel 一下是个好习惯。

再来看看这个 `loadContent` 都做了什么

{% highlight objc %}
- (void)loadContent
{
    // To be implemented by subclasses…
}
{% endhighlight %}

看来需要在子类实现这个方法，那就到 AAPLCatListDataSource 里看看这个方法都做了什么

{% highlight objc %}
- (void)loadContent
{
    [self loadContentWithBlock:^(AAPLLoading *loading) {
        void (^handler)(NSArray *cats, NSError *error) = ^(NSArray *cats, NSError *error) {
            // Check to make certain a more recent call to load content hasn't superceded this one…
            if (!loading.current) {
                [loading ignore];
                return;
            }

            if (error) {
                [loading doneWithError:error];
                return;
            }

            if (cats.count)
                [loading updateWithContent:^(AAPLCatListDataSource *me) {
                    me.items = cats;
                }];
            else
                [loading updateWithNoContent:^(AAPLCatListDataSource *me) {
                    me.items = @[];
                }];
        };

        if (self.showingFavorites)
            [[AAPLDataAccessManager manager] fetchFavoriteCatListWithCompletionHandler:handler];
        else
            [[AAPLDataAccessManager manager] fetchCatListWithCompletionHandler:handler];
    }];
}
{% endhighlight %}

使用了 `loadContentWithBlock:` 方法，进去看看，这个方法做了什么

{% highlight objc %}
- (void)loadContentWithBlock:(AAPLLoadingBlock)block
{
    [self beginLoading];

    __weak typeof(&*self) weakself = self;

    AAPLLoading *loading = [AAPLLoading loadingWithCompletionHandler:^(NSString *newState, NSError *error, AAPLLoadingUpdateBlock update){
        if (!newState)
            return;

        [self endLoadingWithState:newState error:error update:^{
            AAPLDataSource *me = weakself;
            if (update && me)
                update(me);
        }];
    }];

    // Tell previous loading instance it's no longer current and remember this loading instance
    self.loadingInstance.current = NO;
    self.loadingInstance = loading;
    
    // Call the provided block to actually do the load
    block(loading);
}
{% endhighlight %}

简单说来就是生成了一个 loading，然后把 loading 传给 block，那 `loadingWithCompletionHandler:` 这个方法又做了什么

{% highlight objc %}
+ (instancetype)loadingWithCompletionHandler:(void(^)(NSString *state, NSError *error, AAPLLoadingUpdateBlock update))handler
{
    NSParameterAssert(handler != nil);
    AAPLLoading *loading = [[self alloc] init];
    loading.block = handler;
    loading.current = YES;
    return loading;
}
{% endhighlight %}

所以就是生成一个 loading 实例，然后把 handler 存到 block 属性里。既然存了，那将来某个时候一定会用到，从名字上来看，应该是 loading 完成时会被调用，搜索 block 关键字，发现只有在下面这个方法中 block 才会被调用

{% highlight objc %}
- (void)_doneWithNewState:(NSString *)newState error:(NSError *)error update:(AAPLLoadingUpdateBlock)update
{
#if DEBUG
    if (!OSAtomicCompareAndSwap32(0, 1, &_complete))
        NSAssert(false, @"completion method called more than once");
#endif

    void (^block)(NSString *state, NSError *error, AAPLLoadingUpdateBlock update) = _block;

    dispatch_async(dispatch_get_main_queue(), ^{
        block(newState, error, update);
    });

    _block = nil;
}
{% endhighlight %}

既然是 _ 开头，那应该是内部方法，对外封装了几种状态，如 `ignore`, `done`, `updateWithContent:` 等。

咦，这里为什么要先把 _block 赋给一个临时变量 block，然后再把 _block 设为 nil呢？看起来像是为了解决某种内存问题。如果直接 `_block(newState, error, update)` 会怎样？哦，虽然这里没有出现 self，但 _block 是一个 instance 变量，所以在 ^{} 里会对 self 进行强引用。而如果赋给一个临时变量，那么只会对这个临时变量强引用，就不会出现循环引用的情况。

AAPLLoading 看的差不多了，再出来看 `loadContentWithBlock:` ，注意到在 CompletionHandler 里，有这么一段

{% highlight objc %}
[self endLoadingWithState:newState error:error update:^{
	AAPLDataSource *me = weakself;
	if (update && me)
		update(me);
}];
{% endhighlight %}

这里的 self 是 AAPLDataSource （Block嵌套多了，还真是容易晕啊），来看看 `endLoadingWithState:error:update` 这个方法都做了什么

{% highlight objc %}
- (void)endLoadingWithState:(NSString *)state error:(NSError *)error update:(dispatch_block_t)update
{
    self.loadingError = error;
    self.loadingState = state;

    if (self.shouldDisplayPlaceholder) {
        if (update)
            [self enqueuePendingUpdateBlock:update];
    }
    else {
        [self notifyBatchUpdate:^{
            // Run pending updates
            [self executePendingUpdates];
            if (update)
                update();
        }];
    }

    self.loadingComplete = YES;
    [self notifyContentLoadedWithError:error];
}
{% endhighlight %}

设置一些状态，然后在恰当的时机调用 update block，咦，这里有个 dispatch_block_t 没怎么见过，查了一下原来是一个内置的空传值和空返回的block。

看了下 `enqueuePendingUpdateBlock`，会把现在的这个 update 结合之前的 updateBlock，形成一个新的 updateBlock，应该就是视频里提到的当所有的 DataSource 都载入完时，统一执行之前的 update block

`notifyBatchUpdate:` 所做的是看一下 Delegate 是否响应 `dataSource:performBatchUpdate:complete:` 如果响应则走你，不然挨个执行 update / complete。

看完了 `loadContentWithBlock` 再来看看这个 Block 里面都做了什么，大意是根据 self.showingFavorites 来切换不同的数据源，这里看到了一个新的类 AAPLDataAccessManager，看起来像是统一的数据层，瞄一眼

{% highlight objc %}
@class AAPLCat;

@interface AAPLDataAccessManager : NSObject

+ (AAPLDataAccessManager *)manager;

- (void)fetchCatListWithCompletionHandler:(void(^)(NSArray *cats, NSError *error))handler;
- (void)fetchFavoriteCatListWithCompletionHandler:(void(^)(NSArray *cats, NSError *error))handler;
- (void)fetchDetailForCat:(AAPLCat *)cat completionHandler:(void(^)(AAPLCat *cat, NSError *error))handler;
- (void)fetchSightingsForCat:(AAPLCat *)cat completionHandler:(void(^)(NSArray *sightings, NSError *error))handler;

@end
{% endhighlight %}

果然如此，将来数据的载入形式有变化，或需要做缓存啥的，都可以在这一层处理，其他部分不会感觉到变化。

这一轮看下来已经有不少信息量了，来简单捋一下：

{% highlight bash %}
[SegmentedDataSource setNeedsLoadContent]
                ↓
	[CatListDataSource loadContent]
                ↓
   [DataSource loadContentWithBlock:]
                ↓
创建 loading，设置 loading 完成后要做的事 → 拿到数据后放到 updateQueue 里，等全部拿到再执行 batchUpdate
                ↓
执行 loadContentBlock → 使用 DataAccessManager 去获取数据，拿到后交给 loading
{% endhighlight %}

到这里，我们还没有运行 Project 看效果，因为我觉得代码包含的信息会更丰富，而且这么看下来后，对于界面会长啥样也有个大概的了解。

这只是开始，继续挖掘下去还会有不少好东西，比如 Favorite 按钮的处理，它是通过 Responder Chain 而不是 Delegate 来实现的，也是一个思路。通过有限状态机来管理 loading 状态也是很有意思的实现。

如果有兴趣，可以看下 ComposedDataSource，先不看实现，如果要自己写大概会是什么思路，比如当调用 `[UICollectionView 
cellForItemAtIndexPath:]` 时，如何找到对应的 DataSource，找到之后如何渲染对应的 Cell 等。

所以看源码真的是一件很有意思的事情，像一场冒险，总是会有意外收获，可能在不知不觉中，能力就得到了提升。
