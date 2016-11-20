---
layout: post
title: 自学 iOS 开发的一些经验
category: tech
tag: 技术
---

不知不觉作为 iOS 开发也有两年多的时间了，记得当初看到 OC 的语法时，愣是被吓了回去，隔了好久才重新耐下心去啃一啃。啃了一阵，觉得大概有了点概念，看到 Cocoa 那么多的 Class，又懵了，怎么才能调用系统的相机？怎么保存信息？怎么做一个像 Twitter 个人页那样的页面？总之就是不知道该从哪切入。

现在回想起来，其实路一直都在，而且有很多条，当初如果有人能够指出一条还不错的道，或许就能走得不那么艰难。于是就有了这篇文章，希望对后人能有所帮助吧。

### 基础

#### 一定的编程经验

这里说的编程经验是至少熟练一门编程语言，对 OOP 有一定的了解，最好熟悉一些基本的设计模式。遇到过的好多 iOS 开发，大多是从别的语言转过来的，所以有一定的编程基础，学起来会更容易 get the point.

如果是第一次接触编程，当然也是没问题的，只是要做好心理准备，可能会比想象的难。

#### 英语

发现不少开发对于英语似乎有点接受不能，通常都是中文优先，除非迫不得已，才硬着头皮看看 StackOverflow，英文文章，文档等。忘了是谁说过「难走的路越走越好走」，通常如此。其实只要稍微 push 一下自己，那些技术文章啃下来应该不会有太大的问题，有过几次成功的体验后，这种恐惧感就会减少很多。优质的文章、视频、书籍，多是英文的，不迈过这个坎，将来要么成为瓶颈，要么花更大的成本去填补。


### 入门

#### 书籍

要学习 iOS 开发，自然要先学 Objective-C （当然现在也可以直接上 Swift，不过如果多人协作的话，OC目前还是主流），因为 OC 是 C 语言的超集，所以了解 C 语言对于学习 OC 肯定会有帮助，不过就算不了解，直接学 OC 也没太大问题。

这里推荐 BNR (Big Nerd Ranch) 的这本 [Objective-C Programming The Big Nerd Ranch Guide](http://www.amazon.com/Objective-C-Programming-Ranch-Guide-Guides/dp/032194206X)，讲解地比较细致，能帮助你更好的理解 OC，更重要的是教你遇到问题时，如何去解决问题，以及这个问题对应的一些知识点，如何使用文档等等。

来到一个新的世界，肯定会对这个世界充满好奇，想订阅一大堆博客，买一堆书，看各种教程和视频，然后就变得浮躁，不知该从哪下手，这会导致拖延症。我渴了，给我倒一杯水，这个很直接，马上就可以做，但如果是给我买一瓶饮料，而自己对那些饮料又不怎么熟悉时，就纠结了，不如刷会微博，看看朋友圈，玩个小游戏先。

所以一本好的入门教材很重要，要契合自己当前的水平，且常常会有收获，这种成就感会激励着你继续学下去。

在看书的过程中，往往会有这样的经历：书中提到某个人、观点、知识点、书、文章，然后就顺着它提到的这些东西出去了，可能某个知识点又牵扯到另一些内容，然后就这样越走越远。想起了一个故事

> 三只猎狗追一只土拔鼠,土拔鼠逃跑时钻进了一个树洞。这个树洞只有一个出口,不一会儿,忽然从树洞里跑出一只兔子。兔子飞快地向前跑,并爬上另一棵大树。兔子因为慌乱在树上没站稳,掉了下来,砸晕了正仰头看的三只猎狗,最后,兔子终于逃脱。

对于这个故事可以从不同的角度去解读，我更愿意以初心去解读。兔子为什么会爬树？为什么能砸晕三只猎狗？这不是重点，重点是，之前追赶的土拨鼠哪去了？看书时难免会有延伸阅读，这个深度我觉得不宜超过 2 层，不然很容易就回不来了。

还有就是如果有可能，最好每天都看点，这其实是很难的，因为总是会有优先级更高的事，或者之前的某些习惯在干扰。一旦断了几天，就不想再拿起来了。

还有，苹果官方的 [Start Developing iOS Apps Today](https://developer.apple.com/library/ios/referencelibrary/GettingStarted/RoadMapiOS/) 也是很不错的入门材料。

#### 视频

推荐斯坦福老头子(Paul Hegarty)的 [Developing iOS 7 Apps for iPhone and iPad](https://itunes.apple.com/us/course/developing-ios-7-apps-for/id733644550) ，当初也是看的这个（那时还是更老的版本），Paul 是资深的 Mac/iOS 开发（前苹果员工？），很多知识点讲得很到位，学生们的提问也大都在点上，同时配有Demo，总之听下来会对 iOS 开发有比较全面的了解。

同时推荐一本小册子：[objc-zen-book](https://github.com/objc-zen/objc-zen-book)，花不长时间就能看完，里面是一些 Best Practices，对于编写优质代码会很有帮助。

#### 笔记

这是一个持久的过程，任何阶段都适用。以前也没太在意这个，觉得概念性的东西，脑子过一遍，就大概知道了，然后就去啃其他的东西了，现在看来，如果有记笔记的话，会更有助于消化概念、知识点，也可以记录自己的思考过程。达芬奇就记录了10000多页的笔记。

记笔记可以加深对知识点的理解，而成为编程巨星的唯一秘诀就是：[对所做的事情理解地越深，就会做得越好](http://www.codesimplicity.com/post/the-singular-secret-of-the-rockstar-programmer/)。同时如果遵循[遗忘曲线](http://zh.wikipedia.org/wiki/%E9%81%97%E5%BF%98%E6%9B%B2%E7%BA%BF)去复习的话，效果更佳。对知识点了解地足够透彻后，Debug 时才更有可能知道问题出在哪，解决问题也更容易有思路。

笔记不仅可以记知识点，也可以记录调试过程，比如[这篇笔记](http://borkware.com/bnr/CampWhereIOS6.html)，有一种调试方法：[小黄鸭调试法](http://zh.wikipedia.org/wiki/%E5%B0%8F%E9%BB%84%E9%B8%AD%E8%B0%83%E8%AF%95%E6%B3%95)

> 许多程序员都有过向别人（甚至可能向完全不会编程的人）提问及解释编程问题，就在解释的过程中击中了问题的解决方案。一边阐述代码的意图一边观察它实际上的意图并做调试，这两者之间的任何不协调会变得很明显，并且更容易发现自己的错误。

生活中我们可能不会真的这么去做，这时抽离出另一个自己，记录下跟ta的对话，也是个发现问题的好方法。

#### 练习

这也是一个持续的过程，知道了些概念或原理后，总是会想着去验证下是不是这样，无论结果是否如自己预期，实践的过程会降低对语言的陌生感，慢慢地培养一种驾驭这门语言的自信，如果出了错，正好可以重新梳理一下。

#### 目标

如果静下心来看完了 BNR 的这本书，以及斯坦福的 iOS 开发视频，那么对 OC 应该比较了解了，一些常用的 UIKit 用起来也没什么问题了，比如 UIViewController / UIView / UIScrollView / UIImageView / UITableView。也熟悉一些概念，如 KVO / MVC / Delegate / DataSource。

这个阶段下来，应该会有：哦，iOS 开发也就这样嘛，多翻翻文档，熟悉 Cocoa Touch 的一些 Class，差不多也能做出一个简单的 App 了。

### 进阶

入门之后，接下来可以折腾的东西还会有不少。

#### 书籍

[Effective Objective-C 2.0](http://www.amazon.com/Effective-Objective-C-2-0-Specific-Development/dp/0321917014)，里面提到了 52 种提高 iOS App 质量的途径。涉及了 API 设计、protocols / category 的使用、写出更模块化的代码等，读下来应该会有不少收获。

[iOS Programming: The Big Nerd Ranch Guide (4th Edition)](http://www.amazon.com/iOS-Programming-Ranch-Guide-Guides/dp/0321942051)，又是一本 BNR 的书，这本书的特点是通过 Demo 来引出知识点，然后提一些问题，并且会细说解题思路。看书的过程中，对于元学习能力的提升也会有一定帮助。

--- update ---

发现巧哥的 [iOS开发进阶](http://item.jd.com/11598468.html) 已经可以在京东买到了，虽然没有细看，但巧哥出品质量肯定有保障。

#### 其他资源

进入这个阶段后，可以去探索更大的世界了，现在的资源已经很丰富了，但还是要遵循「少而精」的原则。以下是我觉得挺不错的源

* [iOS Dev Weekly](https://iosdevweekly.com/) 每周一期，内容多为这一星期里值得关注的Github项目、文章、工具等。
* [iOS 移动开发周报](http://blog.devtang.com/) 这是唐巧大大整理的每周不错的 iOS 开发相关的内容，多为中文。
* [RayWenderlich](http://www.raywenderlich.com/tutorials) 很多详细又全面的教程，不容错过。
* [iOS Dev Slack](https://iosdev.slack.com/home) 国内不少 iOS 开发（包括大大们）都在这里，不过现在好像不怎么能拿到邀请了。
* [中文 iOS/Mac 开发博客列表](https://github.com/tangqiaoboy/iOSBlogCN)，打开工具订阅吧。

还有，如果可能的话，多去分享自己学到的东西，教是最好的学，我试过几次，效果真的很不错。

#### 目标

这个阶段下来，对于常用的设计模式、内存管理、Blocks 的使用、图像操作、网络请求和管理、多线程应该比较熟悉了。对于 CALayer、Animation、UIScrollView、UITableView、UICollectionView、ViewController Container 则非常熟悉，对「非常熟悉」的定义是：不打开 Xcode，脑子里就能把相应的知识点复述出来 80% ，比如这个类有哪些方法，Delegate / DataSource 有哪些方法，怎么使用，如果要实现某个效果，应该怎么做（好吧， UICollectionView 除外）。

### 高级

其实高级、进阶、入门并没有严格的界限，在入门阶段也可以探究高级阶段的一些东西。我觉得支撑我们不断探索和前进的动力不是兴趣，而是永不满足的好奇心，和对优雅代码的追求。

> If your standards are low, you're going to stop pretty early on in the process.

BNR 的这篇 [Leveling Up](http://www.bignerdranch.com/blog/leveling-up/) 已经讲得很好了，也更加细致。

#### 书籍

[iOS 7 Programming Pushing the Limits](http://www.amazon.com/iOS-Programming-Pushing-Limits-Applications/dp/1118818342) 这本书对 iOS 7 的一些特性会讲解地比较深入，当然也不仅仅是 iOS 7。只叹 iOS 更新实在太快，书籍往往跟不上，一本好书往往需要很长时间来撰写，等书可以出版了，iOS 又出新版本了。

#### 源码

看优秀的源码，可以学到很多东西，使用过程中遇到问题也更容易解决。这些是我觉得值得细看的源码：[AFNetworking](https://github.com/AFNetworking/AFNetworking)(NSOperation, HTTP, Block), [SDWebImage](https://github.com/rs/SDWebImage)(Image Handle, Cache, NSOperation, Block), [SVPullToRefresh](https://github.com/samvermette/SVPullToRefresh)(UIScrollView, State Handle), [JSONModel](https://github.com/icanzilb/JSONModel)(runtime)

如果有兴趣，也可以翻翻 [CoreFoundation](http://www.opensource.apple.com/source/CF/CF-855.17/) / [OC runtime](http://www.opensource.apple.com/source/objc4/objc4-646/) 的源码。

#### 资源

* [oleb](http://oleb.net/blog/)
* [NSHipster](http://nshipster.com/)
* [objc.io](http://objc.io) || [objcio.cn](http://objcio.cn)
* [WWDC 视频](https://developer.apple.com/videos/wwdc/2014/)

#### 工具

* [chisel](https://github.com/facebook/chisel) Facebook 出品的 LLDB 助手，用于调试很方便
* [Reveal](http://revealapp.com/) 每当好奇某个 App 的实现时，都会打开它一窥究竟，用于调试自己的 App 也很方便
* [Aspects](https://github.com/steipete/Aspects) steipete 大大出品的一款方便使用 method swizzling 的工具，可以在运行时动态添加代码到某个方法
* [class-dump](https://github.com/nygard/class-dump) 从 Mach-O 文件生成 OC 头文件，有时想看看某个 App 大概是如何组织的会比较方便
* [Hopper](http://www.hopperapp.com/) 可以对二进制文件进行反编译，甚至可以生成伪代码！有时想看看 UIViewController 里某个方法大概是怎么实现的，就可以用它。
* Instruments 这个内置的工具对于发现 App 的各种问题很有帮助，如内存占用、泄露，渲染问题等。

#### 目标

这个阶段，对于底层的实现会有更深入的了解，各种 Core 开头的 Framework 至少可以说出个大概，工具也能熟练使用，「正经的代码」写过数万行，可能天天在翻 [Dash](http://kapeli.com/dash)。如果别人让你实现某个功能，能在较短的时间内给出不错的实现方案，并且足够细致，甚至精细到如何使用 Core Graphic 去画某个图像。


### 其他

我觉得无论学习什么，「速成」的心态是最要不得的，这只会让自己变得浮躁，一知半解，整个过程也很难让自己的元学习能力得到提升。慢慢来，攻占一个城后，再去打下一个，这时心态也会平和许多。
