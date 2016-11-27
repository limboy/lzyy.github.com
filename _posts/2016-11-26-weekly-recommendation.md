---
layout: post
title: 每周推荐 (2016/11/26)
category: weekly-recommendation
tags: 周荐
---

### 文章

#### Your First Startup
[Source](https://blog.asana.com/2015/11/your-first-startup), 作者是 Facebook 和 Asana 的联合创始人 [Dustin Moskovitz](https://en.wikipedia.org/wiki/Dustin_Moskovitz)，关于他是如何成为 Facebook 联合创始人的可以看下[这篇文章](http://www.businessinsider.com/how-moskovitz-became-facebook-cofounder-2016-3)，他是 Zuck 的室友，在 Zuck 扩张缺少人手时，主动提出支援，之前没有接触过 PHP，看了几天之后就开始干活了。 因此他在选择公司方面的眼光还是有参考价值的。这张图概括了不同体量的公司在不同维度带来的影响。

![](https://asanablog-wpengine.netdna-ssl.com/wp-content/post-images/Screen-Shot-2015-11-11-at-6.17.47-PM-1024x780.png)

同时多从侧面去了解某家公司，比如招聘页、跟员工交流、看对外发布的演说／视频 等。

PS，这篇文章又引出了另一篇 Dustin 之前写的[关于「工作与生活」的文章](https://medium.com/building-asana/work-hard-live-well-ead679cb506d#.uwdt4s7sq)，简单总结下

* 像亚马逊那样高强度的工作虽然造就了某种文化，但副作用也很明显，不值得提倡。
* 每天如果能够集中精力高效工作一段时间，其产出会比低效得延长工作时间好得多，后者不仅自己感觉累，工作业绩也可能比较一般，还容易耽误生活。
* 有些公司为了营造紧张、拼搏的氛围，提倡加班，效果不一定好。

![](https://cdn-images-1.medium.com/max/1600/1*h_Pnw6kJ8FXRGBIPuZOd3Q.png)

#### The One Method I’ve Used to Eliminate Bad Tech Hires
[Source](https://mattermark.com/the-one-method-ive-used-to-eliminate-bad-tech-hires/), 招聘一直是个老大难的问题，都知道它很重要，但好像没有特别靠谱的办法。此文提供了一种方法：给候选人一个模拟任务，在规定时间后给出结果，并向组员解释实现过程。其中有几个重要的点

* 这是一个给钱的任务。比如给你一个周末的时间去解决，同时支付 500 元作为费用。
* 任务要具备开放性，但要明确方向。比如不要问：你觉得我怎么样？而应该问你觉得我的着装还有哪些地方可以改善。
* 向团队成员解释过程。以此来考察候选人的各方面素质。

个人觉得此法还是不错的，对于建立公司口碑也有不少帮助。如果周末的时间偏长可以给一个更短的时间，比如两个小时，然后再看结果。相对于其他的面试套路，这个方法能更加全面地考察候选人。

#### Why Native Apps Really are Doomed
这是 [系列2](https://medium.com/javascript-scene/why-native-apps-really-are-doomed-native-apps-are-doomed-pt-2-e035b43170e9#.1cc6bsf7t)，作者之前还有一篇[系列1](https://medium.com/javascript-scene/native-apps-are-doomed-ac397148a2c0#.9e4987uwt)，表达的意识主要是 Native App 有很多问题，PWA 才是趋势。简单总结下

* Native App 的安装／使用成本较高（需要大概 6 次点击）
* 开发成本较高，各个平台都得整一套
* Google Play 里 60% 的 App 没有被下载过（这个数据还是挺恐怖的）
* 美国将近一半的手机用户每个月都不怎么下 App
* Alibaba 升级到 PWA 后，各方面的数据都得到提升

PWA 在 Android 上借助 Chrome 能获得比较大的能力包括离线缓存、通知、Service Worker 等，但 iOS 平台上就惨淡了些，没有像样的 manifest 文件，只能通过各种 tag 来定义，而且也不支持 Service Worker。说到这，我倒觉得微信的小程序是个出路，一个是能够抹平各个平台的差异，也能充分利用平台特性，加上广大的装机量和方便的获取／卸载成本，PWA 能做的，基本都能做到，甚至更强，而且 iOS 也一并搞定（除了不能在桌面上生成图标）。

#### How we structure our work and teams at Basecamp
[Source](https://m.signalvnoise.com/how-we-set-up-our-work-cbce3d3d9cae#.nv07gqshs) 这是 37Signals 的创始人 Jason Fried 带来的文章，满满的干货，介绍了他们的产品迭代流程：

* 6 周一个迭代周期，其中包含了 1 到 2 个大改动（Big Batch）和一系列的小改动（Small Batch），[样本](https://public.3.basecamp.com/p/yCe87HJ5EX1Nx48N3R6fhfHX)
* 迭代周期过后会留有 1 - 2 周的缓冲期，用来做一些 Side Project 或修一下之前的坑
* 内部使用 Basecamp 来创建 Basecamp，典型的 eat your own dog food
* 灵感来自员工、客户等，有想法可以整理成 pitch 放到 Basecamp 里，大家可以留言讨论，[样本](https://public.3.basecamp.com/p/22KB2DCxpEQLZow8Vc2iJhEq)
* 不跟踪时间，关心结果，随时注意已经完成的和剩下的，避免到最后才知道赶不上进度
* 没有 PM，Designer 充当了类似的角色
* 设计师与一刀两个开发搭配形成临时的 Team，可以按兴趣自由组队，或者指派
* 项目相关的内容（任务／讨论／公告等）都在一个 Basecamp project 里
* 下一个 Cycle 要做什么，由 Jason (CEO) / David (CTO) / Ryan (Strategy) 商讨决定

#### How Muji Fuels Its Explosive Growth Without Ads
[Source](https://www.fastcodesign.com/3064723/the-fast-company-innovation-festival/how-muji-fuels-its-explosive-growth-without-advertiseme)，这篇介绍了为什么很少见到无印良品的广告。「Muji」的全称翻译过来就是「no brand, good quality」，注重「Less is More」，尽管请了大牌设计师，但也是主打功能优先，而弱化设计。取材也不是追求奢侈，而是期望通过创新手法把低廉的原材料发挥到极致。还有一个不打广告的原因是，要把价格压在一定的范围，打了广告后，价格就得上去了。

### 编程

#### 从 Swift 的面向协议编程说开去
[Source](http://www.jianshu.com/p/fc105512bf40)，作者从 Swift 的面相协议编程聊到了面向接口编程，同时又由面向接口编程的问题带出了多继承，然后分析了不同语言对多继承的不同实现（个人比较吐槽的是 Python 的多继承）。作者认为 POP 取代 OOP 是无稽之谈，因为多继承也是 OOP，我倒不这么认为，POP 强调的是遇到问题，先从 `Protocol` 的角度去考虑，而 `OOP` 强调的是从 `Object` 的角度去考虑，因此出发点全然不一样。继承本身不是太大的问题，复杂的继承关系才是问题，而 POP 可以把这个继承关系拉平。无论 POP 的本质是接口还是多重继承，它影响的是编程思维，记住这点就好。

PS: 作者的文章里引用了一个链接，对于帮助理解 Swift 的 Protocol Extension 的动态／静态派发还蛮有帮助的。简单来说，在不显示声明实例类型的情况下，尽量不调默认实现。

![](https://d262ilb51hltx0.cloudfront.net/max/1600/1*SIcSsfmBCp4tNzLxGJAbdw.png)

#### Incremental Swift
[Source](https://realm.io/news/tryswift-amy-dyer-incremental-swift/), 来自 Etsy 工程师的一篇演讲稿，对于那些想在已有工程尝试 Swift 的同学比较有帮助。他们的做法是先通过对已有的 OC 写单元测试来找到感觉，然后通过线上的 A/B 测试来踩坑，最后再用 Swift 来写新 Feature（原则是 OC 可复用，把一些高级特性藏在内部）。

#### How Kotlin became our primary language for Android
[Source](https://medium.com/uptech-team/how-kotlin-became-our-primary-language-for-android-3af7fd6a994c#.8c9a5pple)，作者是 UPTech 的 Co-Founder，这里是[译文](http://mp.weixin.qq.com/s/AZAWsvGomz49cP_IUC8Mqg)。

Kotlin 是一门基于 JVM 开发的，100% 兼容 JAVA 的编程语言，提供了更方便的函数式操作。JAVA 语言的特性是 Verbose，有时过头了就会显得信噪比较低。Kotlin 的语法更加友好，本文可以算作是入门型文章，没有介绍太多使用经验和技巧。对这门语言比较好奇的可以看下。

#### katana-swift
[Source](https://github.com/BendingSpoons/katana-swift), 这是一个受 React 和 Redux 启发的 iOS 开发框架，比起 [ReSwift](https://github.com/ReSwift/ReSwift)，做的事情更加多，也更像 React，包括 Component 的声明和组合，还没有细看，感觉有点意思。

### 影视

#### 地球脉动2
BBC 在 10 年前拍摄了「地球脉动」，10 年后出了第二季，豆瓣和 IMDB 的评分都达到了恐怖的 9.9。通过 4K 摄像机和无人机的配合，效果非常棒，除此之外，里面的一个个小故事也是很吸引我的点。可以在[腾讯视频](https://v.qq.com/x/cover/0qcd5h3k537846y/q0022m7d26n.html)上观看。可以通过这个小短片感受下。

<video poster="null" style="max-width:670px" controls="controls" _v-654ff50c="" src="http://61.240.152.15/vlive.qqvideo.tc.qq.com/b00229gs2yl.mp4?vkey=5DA716F4AB7216C72C9B743A17A6EF784B3E7DA8437BB6D67ED4548F96F031946843346BF11E8B6E97C860F28C45CCBD3E1925A9C43DAC36C0290FF1770EFFB0481BBA05B010E8C2DA573B44854322E460C817A34020ED95"> </video>

#### 控方证人
这也是在电脑里躺了 3 个来月的一部片子，因为是黑白的，一直没有勇气打开看，后来想了下，如果真的是好的故事，跟年代不会有太大的关系，事实证明果真如此。

### Quote

> 优秀的人就像一团光芒，和他们待久了，也就再也不想走回黑暗了！

