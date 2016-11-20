---
layout: post
title: Builder Pattern 在 Objective-C 中的使用
category: tech
tag: 技术
---

在说 Builder Pattern 之前，我们先来看看一个场景。假设我们要预定一个 iPhone 6，要 64G 的，金色的，用代码表述大概是这样

{% highlight objc %}
// PFX 是一个前缀，因为直接写 iPhone6 不符合类名大写的习惯，写成 IPhone6 更是怪异 ╮(╯▽╰)╭
PFXiPhone6 *iphone = [[PFXiPhone6 alloc] init];
iphone.storage = 64;
iphone.color = [UIColor goldenColor];
{% endhighlight %}

也可以是另一种方式

{% highlight objc %}
PFXiPhone6 *iPhone = [[PFXiPhone6 alloc] initWithStorage:64 color:[UIColor goldenColor]];
{% endhighlight %}

第一种方式可扩展性好些，但无法约束必须设置某些 property。第二种方式修正了这个问题，但扩展性也差了。

假如又有了新需求，要让客户可以选择发售区域，比如港行，国行，美版等等。对于第一种，自然可以新增一个属性，但使用者很可能完全不知道新增了这么个属性。对于第二种，则只能再新建一个初始化方式，如 `-[initWithStorage:color:place]`。那如果又有新的需求，比如选择是否刻字，以及刻哪些字呢？或者可以选择外壳的种类等等。这两种方式都不能很好地处理需求的变更。

现在我们来说说 Builder Pattern，这个模式可以在各种语言里被很方便地实现，比如 javascript

{% highlight javascript %}
new PFXiPhone6Builder()
  .setStorage(64)
  .setColor('golden')
  .setPlace('HK')
  .build();
{% endhighlight %}

当有新的属性时，再加一个 `setProperty` 即可。如果漏写了某个属性，可以在 `build` 里检查。

在 Objective-C 里，这样的链式写法不是很流行（[Masonry](https://github.com/Masonry/Masonry)里这种写法用的比较多），所以，在 OC 里写起来大概会是这样

{% highlight objc %}
PFXiPhone6Builder *builder = [[PFXiPhone6Builder alloc] init];
builder.storage = 64;
builder.color = [UIColor goldenColor];
builder.place = @"HK";
PFXiPhone6 *iphone = [builder build];
{% endhighlight %}

如果少了什么属性，在 `build` 时检查下即可。这样既解决了不方便扩展的问题，当有新的属性时也可以知道。

不过看起来还是不够优雅，这个 builder 只是一个临时工具，用完了就扔掉了，既然这样，那有没有可能写法上符合 OC 的传统，又让这个 builder 「临时出现」一下？且看下面这段代码

{% highlight objc %}
PFXiPhone6 *iPhone6 = [PFXiPhone6 createWithBuilder:^(PFXiPhone6Builder *builder){
	builder.storage = 64;
	builder.color = [UIColor goldenColor];
	builder.place = @"HK";
}];
{% endhighlight %}

是不是看起来舒服多了。builder 只是在 block 范围内起作用，不会影响当前 context 的变量。这个 `+[createWithBuilder:]` 的代码如下

{% highlight objc %}
+ (instancetype)createWithBuilder:(BuilderBlock)block {
	NSParameterAssert(block);
	PFXiPhone6Builder *builder = [[PFXiPhone6Builder alloc] init];
	block(builder);
	return [builder build];
}
{% endhighlight %}

这里 `build` 方法，也有两种实现，第一种

{% highlight objc %}
// PFXiPhone6Builder
- (PFXiPhone6 *)build
{
	return [[PFXiPhone6 alloc] initwithBuilder:self];
}

// PFXiPhone6
- (instancetype)initwithBuilder:(PFXiPhone6Builder *)builder
{
	if (self = [super init]) {
		_storage = builder.storage;
		_color = builder.color;
		_place = builder.place;
	}
}
{% endhighlight %}

另外一种是把两个过程合并为一个过程

{% highlight objc %}
// PFXiPhone6Builder
- (PFXiPhone6 *)build
{
	// 可以在这里对 property 做检查
	NSAssert(self.place, @"发行区别忘了填哦");

	PFXiPhone6 *iphone6 = [[PFXiPhone6 alloc] init];
	iPhone6.storage = self.storage;
	iPhone6.color = self.color;
	iPhone6.place = self.place;

	return iPhone6;
}
{% endhighlight %}

这两种方式的区别在于对参数的处理，前一个是在目标 Class 中处理，后一种是在 Builder 中处理。

在 Facebook 的 [pop](https://github.com/facebook/pop) 中也有类似的使用，如

{% highlight objc %}
POPAnimatableProperty *animatableProperty = [POPAnimatableProperty propertyWithName:@"property" initializer:^(POPMutableAnimatableProperty *prop) {
    prop.writeBlock = ^(id obj, const CGFloat values[]) {
    };
    prop.readBlock = ^(id obj, CGFloat values[]) {
    };
}];
{% endhighlight %}

这里的 `initializer` 其实就是 `builder`

我在写蘑菇街的基础框架时，也有用到过几处，觉得还是蛮方便的，尤其对使用者来说。比如这个可以横向或纵向滚动的包含可点击 Items 的 collectionView。

{% highlight objc %}
self.collectionView = [MGJFlowCollectionView collectionViewWithBuilder:^(MGJFlowCollectionViewBuilder *builder) {
	builder.scrollDirection = UICollectionViewScrollDirectionHorizontal;
	builder.minimumInteritemSpacing = 10;
	builder.minimumLineSpacing = 10;
	builder.sectionInset = UIEdgeInsetsMake(10, 10, 10, 10);
	CGSize itemSize = CGSizeMake(81, 100);
	builder.itemSize = itemSize;
	builder.dataSource = @[@1,@2,@3,@4,@5,@6, @7,@8, @9, @10];
	builder.cellBuilder = ^UIView *(NSNumber *number){
		UIView *view = [[UIView alloc] initWithFrame:CGRectMake(0, 0, itemSize.width, itemSize.height)];
		view.backgroundColor = [UIColor mgj_random];
		return view;
	};
}];
{% endhighlight %}

这样就能通过简单的配置来生成一个水平的或垂直的 collectionView 了。

使用 Builder Pattern 还有一个好处，就是可以将零散的配置统一起来。比如要创建一个 CollectionView，我们需要设置 layout，还要设置 layout 的一些属性，还要设置 DataSource / Delegate 等，现在可以在一个地方统一设置，可读性上也会好一些。

所以如果遇到需要多个参数，甚至某个参数自己还包含了各种参数时，可以考虑下 Builder Pattern。

参考：[http://www.annema.me/the-builder-pattern-in-objective-c](http://www.annema.me/the-builder-pattern-in-objective-c)
