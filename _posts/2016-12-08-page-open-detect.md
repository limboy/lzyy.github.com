---
layout: post
title: 「无侵入页面加载完成检测」的一些思路
category: tech
tags: 技术
---

### 前言

在诸多的性能指标里，「页面加载完成耗时」是非常重要的一项，尤其是重点页面，如详情页，1 秒内打开和 3 秒内打开差别是很大的，直接影响 GMV。

再来说一下「页面加载完成」的定义，不是页面 layout 完成，不是请求完成，而是图片和文字都已被渲染完成。比较常见的做法是在页面的 `ViewDidAppear` 和请求完成且数据被转换成 Model 之后分别打点，前者表示页面出现的时间，后者表示数据获取的时间，基本可以体现出页面加载时间。但也有一些问题比如：

1. 业务经常调整，所以埋点也需要调整，这个过程中很容易出现错埋、漏埋的问题。
2. 有些页面会有多个请求，只有这些请求全部完成后，页面才能渲染，这时数据请求埋点就会有点麻烦。
3. 这几个时间点跟用户真正看到的时间还是会有差别，不够准确。

所以一种无侵入的检测机制就很重要了。正好在[掘金](https://gold.xitu.io)上看到了[用图像识别来自动确认网页加载成功](https://gold.xitu.io/post/58440e98128fe1006c4c951d)，受此启发，觉得此路有戏。

### 实现方案
当 push／present 一个页面时，隔一段时间去截屏并分析当前页面的空白（纯色）部分占比，如果超过某个阈值，就认为页面未加载完成。这里会有几个注意点：

1. 需要主动去截屏检测，而不能加载完成后告知。这其中的差别在于无法得知具体哪个时间加载完成了。
2. 有些页面被故意设计成有较多留白，这时就不容易判断了。
3. 「未加载完成」不同的页面会有不同的表现。
4. 当用户滑动时，有可能之前的页面已经加载了

### 纯色占比
最简单的方案就是把图片上的每个像素点都取出来，放到一个字典里，之后如果有相同色值的像素，那么 `count++`。问题也很明显，一个屏幕几十万个点，这一轮都还没分析完，用户已经打开第二个页面了。

再回到想要达到的目标：纯色部分占比。那么将图片压缩到更小的 size 不就行了么。老套路，铺张画布，把图片浇上去。

```objc
+ (UIImage *)imageWithImage:(UIImage *)image scaledToSize:(CGSize)newSize {
    UIGraphicsBeginImageContextWithOptions(newSize, NO, 1.0);
    [image drawInRect:CGRectMake(0, 0, newSize.width, newSize.height)];
    UIImage *newImage = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    return newImage;
}
```

接下来就是真正的计算了，过程也比较简单：

```objc
// 把 UIImage 转换成 CGImage Data
CGDataProviderRef provider = CGImageGetDataProvider(image.CGImage);
CFDataRef pixelData = CGDataProviderCopyData(provider);
const UInt8* data = CFDataGetBytePtr(pixelData);
long dataLength = CFDataGetLength(pixelData);
int numberOfColorComponents = 4; // R,G,B, and A

// 用来装 color ，key 为 R-G-B 字符串，value 为出现次数
NSMutableDictionary *colors = [[NSMutableDictionary alloc] init];

int colorCount = 0;

for (int i = 0; i < (dataLength); i += numberOfColorComponents) {
	if (data[i+3] != 0) {
		colorCount++;
		UInt8 red = data[i];
		UInt8 green = data[i+1];
		UInt8 blue = data[i+2];
		
		// 以 R-G-B 为 key
		NSString *result = [NSString stringWithFormat:@"%d-%d-%d", red, green, blue];
		if (!colors[result]) {
			colors[result] = @1;
		} else {
			colors[result] = @([colors[result] integerValue] + 1);
		}
	}
}

// 按出现次数排序
NSArray *sortedColorCount = [colors.allValues sortedArrayUsingComparator:^NSComparisonResult(id  _Nonnull obj1, id  _Nonnull obj2) {
	return [obj1 intValue] < [obj2 intValue] ? NSOrderedDescending : NSOrderedAscending;
}];

NSMutableArray *percent = [[NSMutableArray alloc] init];

// 计算占比，并从高到低排序，取前 10 个
[sortedColorCount enumerateObjectsUsingBlock:^(NSNumber *count, NSUInteger idx, BOOL * _Nonnull stop) {
	if (idx >= 10) {
		*stop = YES;
	}
	[percent addObject:@([count intValue] / (float)(colorCount))];
}];

return [percent copy];
```

先把 `UIImage` 转换成 `CFDataRef`，再遍历获取 `Color`，对相同的的 `Color` 进行累加，最后排一下序即可。

返回的数据类似这样：

```objc
(
	0.4586517,
	0.06202247,
	0.02921348,
	...
)
```

这样就能拿到了颜色的占比。

### 实战

假设设定纯色区域超过 30% 认为没有完全加载，来找几个 Demo 测试下：

![](http://s16.mogucdn.com/p1/161208/idid_ifqtcztfhbqwendcmmzdambqgyyde_600x1036.png)

结果符合「未加载完毕」定义

```objc
(
    "0.4139326",
    "0.06808989",
    "0.05438202",
    ...
)
```

再换一个

![](http://s16.mogucdn.com/p1/161208/idid_ifrgizrwme2wgndcmmzdambqmeyde_600x1036.png)

虽然没有加载完，但结果少于 30%

```objc
(
    "0.2788764",
    "0.06808989",
    "0.04853933",
    ...
)
```

如果把值设得小一些，那么有可能「误杀」，比如这个界面

![](http://s17.mogucdn.com/p1/161208/idid_ifrdimlemfrggndcmmzdambqmeyde_600x1067.png)

结果

```objc
(
    "0.4530337",
    "0.06561798",
    "0.02921348",
    ...
)
```

这个界面已经加载完成了，但由于空白面积较多，因此纯色的占比也较多，如果按照之前的公式就会误伤，如何解决这个问题，之后再讨论。

接下来看另一个未加载完毕的页面：

![](http://s16.mogucdn.com/p1/161208/idid_ie4tszldguytanlcmmzdambqgqyde_750x1278.png)

这个页面的结果是这样

```objc
(
    "0.3433708",
    "0.1941573",
    "0.1822472",
    ...
)
```

如果中间部分加载出来（也就是面积最大的那一块），那么就变成了

```objc
    "0.1941573",
    "0.1822472",
```

这也属于页面未加载完成，但又是一个新的规则了。

### 小结
再来回顾一下「截图分析纯色占比」带来的问题：

1. 隔 N 秒去截图时，用户可能滑到第 2 屏了，这时第 1 屏加载完了，但 2 屏还没有加载完，不应该属于「页面加载未完成」范畴。
2. 不同页面的纯色特性不一样，有的比较分散，有的正常状态下也会有比较多的纯色，这样就容易误判。

对于场景 1 还没有想到特别好的处理方式，一种办法是通过判断 runloop 的 mode 是否等于 `UITrackingRunLoopMode` 来判断是否有滑动，不太优雅，但可能行得通。

对于场景 2 可以把数据发送到服务端，让服务端去计算某个页面的纯色分布情况，比如大部分都是 < 10%，有少部分在 20% 以上，那么就可以判定为未加载完成，不过成本还是有点高。

所以这个方案虽然可以做到无侵入，但在结果判定上还是存在些缺陷，期待有更成熟的方案。


