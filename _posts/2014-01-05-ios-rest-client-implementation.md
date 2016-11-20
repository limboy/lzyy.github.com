---
layout: post
title: 基于AFNetworking2.0和ReactiveCocoa2.1的iOS REST Client
category: tech
tag: 技术
---

在开发iOS App时经常会遇到跟后端REST API通信的情况。这就涉及到错误处理，NSDictionary与Model的映射，用户登录与登出，权限验证，Archive/UnArchive，Copy，AccessToken过期处理等等，如果没有很好地处理这些点，就容易出现代码复杂度增大，结构散乱，不方便后期维护的现象。

正好最近在看AFNetworking2.0和ReactiveCocoa2.1，参考了github的[octokit](https://github.com/octokit/octokit.objc)，重写了花瓣的iOS REST API，分享些心得。

### 基本结构

{% highlight objc %}
|- HBPAPI.h
|- Classes
	|- HBPAPIManager.h
	|- HBPAPIManager.m
	|- Models
		|- HBPObject.h
		|- HBPObject.m
		|- HBPUser.h
		|- HBPUser.m
		...
{% endhighlight %}

使用时，直接引用`HBPAPI.h`即可，里面包含了所有的Class。因为使用了AFNetworking2.0，所以不再是HBPClient，而是HBPManager。 HBPAPIManager包含了所有的跟服务端通信的方法，通过Category来区分。

{% highlight objc %}
#pragma mark - HBPAPIManager (Private)

@interface HBPAPIManager (Private)

// 内部统一使用这个方法来向服务端发送请求
//
// resultClass - 从服务端获取到JSON数据后，使用哪个Class来将JSON转换为OC的Model
// listKey - 如果不指定，表示返回的是一个object，如user，如果指定表示返回的是一个数组，listKey就表示这个列表的keyname，如{'users':[]}, 那么listName就为'user'
- (RACSignal *)requestWithMethod:(NSString *)method relativePath:(NSString *)relativePath parameters:(NSDictionary *)parameters resultClass:(Class)resultClass listKey:(NSString *)listKey;
@end


#pragma mark - HBPAPIManager (User)

@interface HBPAPIManager (User)

// signal会send user，如果没有user，就会sendError
// 必须当前用户已经登录的情况下调用
- (RACSignal *)fetchUserInfo;

// ...
@end


#pragma mark - HBPAPIManager (Friendship)
// ...
{% endhighlight %}

Models Group包含了所有跟服务端API对应的Model，比如`HBPComment`

HBPComment.h

{% highlight objc %}
#import "HBPObject.h"

@class HBPUser;

@interface HBPComment : HBPObject
@property (nonatomic, assign) NSInteger commentID;
@property (nonatomic, copy) NSString *createdAt;
@property (nonatomic, strong) HBPUser *user;
//...
@end

{% endhighlight %}

HBPComment.m

{% highlight objc %}
#import "HBPComment.h"

@implementation HBPComment

- (NSDictionary *)JSONKeysToPropertyKeys
{
    return @{
             @"comment_id": @"commentID",
             @"user_id": @"userID",
             @"created_at": @"createdAt",
			 //...
             };
}

@end
{% endhighlight %}

### Archive / UnArchive / Copy

每一个Object都要支持Archive / UnArchive / Copy，也就是要实现`<NSCoding>`和`<NSCopying>`协议，这两个协议的内容其实就是对Object的Property做些处理，所以如果可以在基类里把这些事都统一处理，就会方便许多。octokit使用[Mantle](https://github.com/MantleFramework/Mantle)来做这些事情，不过我觉得Mantle还是有些麻烦，于是写了个通过运行时来获取property，并实现`<NSCoding>` 和 `<NSCopying>`的基类，只有两个公共方法：

{% highlight objc %}
#import <Foundation/Foundation.h>

@interface HBPObject : NSObject <NSCopying, NSCoding>

// 解析API返回的JSON，返回对应的Model
- (id)initWithDictionary:(NSDictionary *)JSON;

// JSON key到property的映射关系
- (NSDictionary *)JSONKeysToPropertyKeys;
@end

{% endhighlight %}

其中`- (id)initWithDictionary:(NSDictionary *)JSON`的作用是遍历Object的Property，如果Property的Class是`HBPObject`，那么就调用`- (id)initWithDictionary:(NSDictionary *)JSO`，不然就通过KVC的`setValue:forKey:`来设定值。

`- (NSDictionary *)JSONKeysToPropertyKeys`的内容大概是这样：

{% highlight objc %}
- (NSDictionary *)JSONKeysToPropertyKeys
{
    return @{
             @"id": @"ID",
             @"nav_link": @"navLink",
             };
}
{% endhighlight %}

这样通过一个`HBPObject`基类就完成了 Archive / UnArchive / Copy 。

### 用户的登录与登出

先来说说登录，由于使用RAC，在构造API时，就不需要传入Block了，随之而来的一个问题就是需要在注释中说明`sendNext`时会发送什么内容。

{% highlight objc %}
+ (RACSignal *)signInUsingUsername:(NSString *)username passowrd:(NSString *)password
{
    NSAssert(API_CLIENT_ID && API_CLIENT_SECRET, @"API_CLIENT_ID and API_CLIENT_SECRET must be setted");
    NSDictionary *parameters = @{
                                 @"grant_type": @"password",
                                 @"username": username,
                                 @"password": password,
                                 };
    HBPAPIManager *manager = [self createManager];
    
    return [[manager fetchTokenWithParameters:parameters]
            setNameWithFormat:@"+signInUsingUsername:%@ password:%@", username, password];
}
{% endhighlight %}

看着还挺简单的吧，因为主要工作都是`+fetchMoreData:parameters`在做，看看它的实现

{% highlight objc %}
- (RACSignal *)fetchTokenWithParameters:(NSDictionary *)parameters
{
    return [[[[[[[self rac_POST:@"oauth/access_token" parameters:parameters]
             // reduceEach的作用是传入多个参数，返回单个参数，是基于`map`的一种实现
             reduceEach:^id(AFHTTPRequestOperation *operation, NSDictionary *response){
			     // 拿到token后，就设置token property
				 // setToken:方法会被触发，在那里会设置请求的头信息，如Authorization。
                 HBPAccessToken *token = [[HBPAccessToken alloc] initWithDictionary:response];
                 self.token = token;
                 return self;
             }]
             catch:^RACSignal *(NSError *error) {
			     // 对Error进行处理，方便外部识别
                 NSInteger code = error.code == -1001 ? HBPAPIManagerErrorConnectionFailed : HBPAPIManagerErrorAuthenticatedFailed;
                 NSError *apiError = [[NSError alloc] initWithDomain:HBPAPIManagerErrorDomain code:code userInfo:nil];
                 return [RACSignal error:apiError];
             }]
             then:^RACSignal *{
			     // 一切正常的话，顺便获取用户信息
                 return [self fetchUserInfo];
             }]
             doNext:^(HBPUser *user) {
			     // doNext相当于一个钩子，是在sendNext时会被执行的一段代码
                 self.user = user;
             }]
			 // 把发送内容换成self
             mapReplace:self]
			 // 避免side effect
             replayLazily];
}
{% endhighlight %}

这里对signal进行了chain / modify / hook 等操作，主要作用是获取access token和用户信息。

用户的登出就简单了，直接设置`user`和`token`为`nil`就行了。

### 设置超时时间和缓存策略

因为AF2.0使用了新的架构，导致要设置Request的超时和缓存稍微有些麻烦，需要新建一个继承自`AFHTTPRequestSerializer`的Class

{% highlight objc %}
@interface HBPAPIRequestSerializer : AFHTTPRequestSerializer @end

@implementation HBPAPIRequestSerializer

- (NSMutableURLRequest *)requestWithMethod:(NSString *)method URLString:(NSString *)URLString parameters:(NSDictionary *)parameters
{
    NSMutableURLRequest *request = [super requestWithMethod:method URLString:URLString parameters:parameters];
    request.timeoutInterval = 10;
    request.cachePolicy = NSURLRequestReloadIgnoringLocalCacheData;
    return request;
}

@end
{% endhighlight %}

然后将这个class设置为manager.requestSerializer

{% highlight objc %}
HBPAPIManager *manager = [[HBPAPIManager alloc] initWithBaseURL:[NSURL URLWithString:API_SERVER]];
manager.requestSerializer = [HBPAPIRequestSerializer serializer];
{% endhighlight %}

这样就行了

### 权限验证

这个比较简单些，直接在方法里面加上判断

{% highlight objc %}
- (RACSignal *)createCommentWithPinID:(NSInteger)pinID text:(NSString *)text
{
    if (!self.isAuthenticated) return [RACSignal error:[self.class authenticatedError]];
 
    return [self requestWithMethod:@"POST" relativePath:[NSString stringWithFormat:@"pins/%d/comments", pinID] parameters:@{@"text": text} resultClass:[HBPComment class] listKey:@"comment"];
}
{% endhighlight %}

### AccessToken过期的处理

AccessToken过期和获取新的AccessToken可以交给使用者来做，但是会比较麻烦，最好的方法是过期后自动去获取新的AccessToken，拿到Token后自动去执行之前失败的请求，这块我是这么处理的

{% highlight objc %}
- (RACSignal *)requestWithMethod:(NSString *)method relativePath:(NSString *)relativePath parameters:(NSDictionary *)parameters resultClass:(Class)resultClass listKey:(NSString *)listKey
{
    RACSignal *requestSignal;
	// create requestSignal
	// ...

	return [requestSignal catch:^RACSignal *(NSError *error) {
        if (error.code == HBPAPIManagerErrorInvalidAccessToken) {
            return [[[self refreshToken] ignoreValues] concat:requestSignal];
        }
        return [RACSignal error:error];
    }];
}
{% endhighlight %}

### HBPObject SubClass

那些继承自`HBPObject`的子类，有些事情是`HBPObject`无法处理的，比如NSArray的Property，因为Objective-C不支持generic，所以无法知道这个数组包含的究竟是怎样的Class，这时就需要在子类对这些property做处理。

比如画板(HBPBoard)有一个叫`pins`的NSArray属性，因为在`HBPObject`中使用了KVC，所以如果子类有类似`setXXX:`的方法的话，那么该方法就会被调用，利用这一点，就可以处理那些特殊情况。

{% highlight objc %}
@implementation HBPBoard

- (void)setPins:(NSArray *)pins
{
    _pins = [[pins.rac_sequence map:^id(id value) {
        return [[HBPPin alloc] initWithDictionary:value];
    }] array];
}

@end
{% endhighlight %}

再比如，返回的JSON内容中，有一个叫`content`的key，其中有type / date / color 等sub key，而你只想要`type`信息，只需添加一个`type` property，然后在`setContent`时，设置一下`type`即可。

{% highlight objc %}
- (void)setContent:(NSDictionary *)content
{
    _type = content[@"type"];
}
{% endhighlight %}

---

以上就是我在使用AFNetworking2.0和ReactiveCocoa2.1构建iOS REST Client时的一些小心得，确实能感觉到RAC带了不少方便，虽然也同时带来了一些弊端（如返回的内容不明确，学习成本高），但还是利大于弊。

有什么问题和想法，欢迎交流。
