---
layout: post
title: 一个 Demo 入门 Flutter
category: tech
tags: 技术
---

Flutter 是 Google 研发的一套移动端开发框架，也是 Google 正在研发的下一代操作系统 Fuchsia 的 App 开发框架（Web 和 Desktop 也都在进行积极的尝试），前几天刚发布了 1.0 正式版。关于 Flutter 的原理和介绍可以参考美团的[这篇文章](https://tech.meituan.com/waimai_flutter_practice.html)。

本文希望通过一个 Demo 来更深入地了解 Flutter 的布局、状态管理等细节。这个 Demo 可以获取豆瓣的 Top 250 电影，以自定义列表形式呈现，可以收藏/取消收藏，可以点击查看详情页。

源码: [https://github.com/lzyy/flutter-demo-topmovies](https://github.com/lzyy/flutter-demo-topmovies)

[这里](https://video.twimg.com/ext_tw_video/1070286725656662017/pu/vid/360x640/uoeUgoq2-EigTzi7.mp4?tag=6)可以查看最终的效果。

![](/image/movie250-demo.png)

### 目录划分

接到这个需求后首先要考虑的是目录结构应该怎么划分，这也是架构的一部分，我是这么分的:

```
blocs/
widgets/
models/
pages/
routes/
services/
main.dart
env.dart
```

#### BLoC

`BLoC` 是 Business Logic of Components 的缩写，也就是负责所有业务逻辑的，跟 `ViewModel` 的职能挺像的。是一个纯洁的 Dart 类，跟 UI 完全解耦，更加详细的说明可以参见[这篇文章](https://juejin.im/post/5bb6f344f265da0aa664d68a)

#### Widgets

这个目录下面放的是所有的 Widget，Widget 位于 Flutter Framework 的最上层，用来描述 UI 元素的 Layout / Animation 等，或者非 UI 元素(如 DataProvider)，最终这些 Widget 会被组装起来形成 Page。

#### Models

服务端的 JSON 过来后，需要转换成对应的 Model 方便使用，Model 不需要包含业务逻辑，但可以有一些简单的 Model 相关的操作，比如 CRUD。

#### Pages

通常一个 App 会有多个页面组成，这些页面就可以放到这个目录下。

#### Routes

虽然 Flutter 也支持直接 push 一个 Widget，但这样不方便页面管理，对于像「外部的 URL 可以直接跳转到某个页面」这样的功能处理起来也会麻烦些。因此，需要有一个地方可以对 URL 进行注册。

#### Services

这个是放 Remote API 相关的，理论上来说，加一层 Repository 抽象会更加合适，出于方便，就去掉了这一层。

#### main.dart

入口文件，用来初始化 App

#### env.dart

用来设置一些环境变量，类似于 Config。

设置好了目录后，接下来进行任务分解，首先要完成的就是布局，进行布局之前需要有数据源，方便模拟，最好跟正式接口一致，那就先来完成这一项工作。

### 设置数据接口

我们希望从模拟环境到真实环境只需改一个配置即可，为了达到这个目的，我们先把协议定好，到时只要换一个实现就行。这里会用到两个接口

```dart
import 'package:topmovies/models/movie.dart';

abstract class API {
  Future<MovieEnvelope> getMovieList(int start);
  Future<Movie> getMovie(String movieID);
}
```

然后新建一个 Mock API 类来实现这个接口

```dart
class MockAPI extends API {
  @override
  Future<Movie> getMovie(String movieID) {
    return createMockMovieWithTitle('我是电影 $movieID');
  }

  @override
  Future<MovieEnvelope> getMovieList(int start) {
    // 现在还用不着
    return null;
  }
}
```

然后在 `env.dart` 里新建一个 Env 类

```dart
import 'services/api.dart';

class Env {
  static API apiClient;
}
```

其实就是提供一个全局的 apiClient 注入接口，App 初始化时，设置好 apiClient，使用时不需要关心是哪个 apiClient 实例，这样也方便单元测试。

经过这两步后，数据接口就准备好了，接下来需要设置 BLoC。

### 设置首页的 BLoC

BLoC 其实就是 ViewModel，有了 API 实现之后，接下来就要让 Widget 可以用上这些数据，这就是 BLoC 做的事。

![](/image/movie250-bloc.png)

Widget 告诉 BLoC 发生了什么，BLoC 告诉 Widget 哪些数据更新了。

```dart
class MoviesBloc extends BlocBase {
  final BehaviorSubject<MovieEnvelope> _movieEnvelope = BehaviorSubject();
  var _currentStart = 0;

  Observable<MovieEnvelope> get movieEnvelope => _movieEnvelope.stream;

  MoviesBloc() {
    _getMovies();
  }

  _getMovies() {
    Env.apiClient.getMovieList(_currentStart).then((movieEnvelope) {
      var newMovieEnvelope = movieEnvelope;
      if (_movieEnvelope.value != null) {
        newMovieEnvelope.movies = _movieEnvelope.value.movies
          ..addAll(movieEnvelope.movies);
      }
      _movieEnvelope.add(newMovieEnvelope);
      _currentStart = movieEnvelope.movies.length;
    });
  }
}
```

对于 Widget 来说，只要 listen `bloc.movieEnvelope` 就能第一时间拿到最新的 movie 数据，然后把它们展示出来即可。

### StreamBuilder

如果直接用 `setState` 的话，使用姿势是在 state 的 `initState` 时 listen `bloc.movieEnvelope`，当收到新的内容时，setState，这样系统就会 rebuild widget，然后用上最新的数据。Flutter 很贴心地提供了便捷的类 `StreamBuilder`，只要告诉它 `stream`，那么当这个 stream 有新数据时，`itemBuilder` 就会自动 rebuild。

```dart
class Movies extends StatefulWidget {
  @override
  State<StatefulWidget> createState() {
    return _MoviesState();
  }
}

class _MoviesState extends State<Movies> {
  @override
  Widget build(BuildContext context) {

    final bloc = BlocProvider.of<MoviesBloc>(context);

    return StreamBuilder<MovieEnvelope>(
        stream: bloc.movieEnvelope,
        builder: (context, snapshot) {
        // snapshot.data 就是最新的内容
        // 返回一个新的 widget 即可
        })
  }
}
```

注意到这里有一个 `BlocProvider`，这是何物？它其实也是个 Widget（是的，Flutter 的 Widget 并不局限于 UI）。

为什么通过`of`方法能拿到这个 bloc？因为在构建 Tree 时，`BlocProvider` 套在了当前视图的上层(只要从当前节点向上追溯能找到就行)，就像这样：

```dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocProvider(
        bloc: MoviesBloc(),
        child: MaterialApp(
          title: 'Douban Movie Top 250',
          home: Home(),
        ));
  }
}
```

而这个 `of` 方法也很简单：

```dart
static T of<T extends BlocBase>(BuildContext context) {
  final type = _typeOf<BlocProvider<T>>();
  // 这句话的意思就是从当前 context 向上找，找到第一个该类型的实例为止，找不到就返回 null
  BlocProvider<T> provider = context.ancestorWidgetOfExactType(type);
  return provider.bloc;
}
```

BLoC 差不多有了之后，接下来就可以进入布局阶段了。

### 首页布局

![](/image/movie250-home-layout.png)

Flutter 提供了两种列表布局方式: `ListView` 和 `GridView`，如果页面里除了列表还有其他元素（比如顶部的 SlideView 等），就需要用到 `CustomScrollView` 或者 `NestedScrollView`，首页除了列表没有其他元素，同时一行可能包含多个视图，因此我们选择 `[GridView](https://docs.flutter.io/flutter/widgets/GridView-class.html)`。

GridView 的构建可以使用 `GridView.builder`, 它需要提供几个关键信息:

```dart
GridView.builder(
  gridDelegate: // 提供最终的布局信息，x,y,width.height
  itemCount: // 一共有多少元素
  itemBuilder: (context, index) {} // 每个 item 具体长啥样
);
```

如果是固定的每行几列或固定宽度，且每个 item 的呈现形式几乎一样，可以直接使用自带的 `SliverGridDelegateWithFixedCrossAxisCount` 或 `SliverGridDelegateWithMaxCrossAxisExtent`，我们这个 case 中，每一行的列表不都是一样的，因此不能直接拿来用，这就需要进入到高级模式了，自己实现 `gridDelegate`。 `SliverGridDelegate` 的核心方法是 `SliverGridLayout getLayout(SliverConstraints constraints);` 也就是返回一个 `SliverGridLayout`，系统拿到这个 layout 之后，就知道该怎么布局了。

```dart
abstract class SliverGridLayout {
  /// 针对某个 scroll 的偏移量，最小的 index 是多少
  int getMinChildIndexForScrollOffset(double scrollOffset);

  /// 针对某个 scroll 的偏移量，最大的 index 是多少
  int getMaxChildIndexForScrollOffset(double scrollOffset);

  /// 给一个 index，告诉我它的 x,y,width,height
  SliverGridGeometry getGeometryForChildIndex(int index);

  /// 这些 childcount 一共能产生多大的偏移量
  /// 知道了这个信息后，系统就可以展示滚动条的长短了
  double computeMaxScrollOffset(int childCount);
}
```

具体实现这里就不贴出来了，感兴趣的可以在源码里看。还是有点小复杂的，尤其是加上 loadmore 的逻辑后，不过知道了系统想要什么，想尽办法满足它就是了。

### 载入更多

第一次请求的布局是完成了，如何实现载入更多的效果呢？在 iOS 中会通过判断是否拉到了底部来触发加载更多的逻辑，在 Flutter 中我们可以换一种方式来达到效果。

什么时候需要加载更多？当前视图的 item 都展示完了的时候，而在展示 item 时，builder 会传入一个 `index`，用来告知当前 item 处于哪个 `index`，我们可以把这个信息告诉 BLoC。比如第一页一共展示 20 部电影，当 BLoC 收到 index 19 时，就知道这 20 部都已经被展示过了，就可以通过 API 去拿更多的数据，等拿到后，跟原先的数据进行合并，然后作为新的值告诉 widget，widget 的 StreamBuilder 发现有新数据后，自动刷新 widget，这样新的电影就被展示出来了。

稍微复杂的一点是加载更多时，需要展示一个 indicator，当没有更多数据时，又是另一个样式，这又该如何处理呢？可以思考下。

### Real API 接入

前面这几步都走完后，列表的布局应该没问题了，接下来就要接入真实的数据了。这个接入过程还是挺简单的，新建一个实现了 `API` 的类，然后在 App 入口处，用它替换掉 MockAPI

```dart
class RealAPI extends API {
  @override
  Future<MovieEnvelope> getMovieList(int start) async {
    var client = HttpClient();
    var request = await client.getUrl(Uri.parse(
        'https://api.douban.com/v2/movie/top250?start=$start&count=40'));
    var response = await request.close();
    var responseBody = await response.transform(utf8.decoder).join();
    Map data = json.decode(responseBody);
    return MovieEnvelope.fromJSON(data);
  }

  @override
  Future<Movie> getMovie(String movieID) async {
    // 暂时还用不到，先忽略
    return null;
  }
}
```

Dart 内置了对异步请求的支持，分为两大块，`async + await + Future` 和 `Stream + async* + yield`，前者用来处理单次或少量的异步请求，后者用来实现异步的 `iterator`，也就是数据可能会源源不断地冒出来。

在这个例子中，通过 await 来拿异步数据就可以了，拿到之后把它转换为 model(方便起见，错误处理就先忽略)。 然后在入口处设置

```dart
void main() {
  Env.apiClient = RealAPI();
  runApp(MyApp());
}
```

### item 的内容展示

这块是个细致活，对于 Widget 元素的使用可以参考官方的[这个教程](https://flutter.io/docs/development/ui/layout)，需要提一点的是 `LayoutBuilder` 这个 widget，默认 child widget 是拿不到 parent 的布局信息的，但有时又需要拿它来进行一些计算，这时就可以在外面套一层 `LayoutBuilder`，它的 `builder` 属性是一个 function，会传一个 `BoxConstraints` 进来，通过它就能拿到 parent 的布局信息。

### 收藏影片

点击影片 title 旁边的 `...`， 如果是 iOS 平台，则弹出一个 `ActionSheet`，Android 则弹出 `BottomSheet`，选择收藏的话，这个 `...` 会变成红色。

如果要针对不同平台进行不同的展示，可以使用 `Platform.isIOS` 来区分(这个类在 `dart:io` package 下)，关于 `ActionSheet` 和 `BottomSheet` 的使用，查看相应的 API 即可。比较麻烦的是 `...` 的变色处理，倒不是变色麻烦，而是要让这个状态得到保持，不然下次再滑到该 item 时，又会回到原先的颜色。

大多数的教程里，对这部分的处理都是更新 list 中该 item 对应的 model，然后让更新后的 list 触发 StreamBuilder 重新渲染 widget，我觉得这样实在有点小题大做了。我的做法是给每个 item 配一个对应的 bloc，item 的 model 和状态都保存在这个 bloc 中，在这个例子中，当某个 movie item widget 需要更新状态时，从它对应的 bloc 中拿即可。

如果这时要新加一个 Feature，当收藏电影时，顶部 AppBar 的右边要有对应的数字显示。在 Google 官方的 [Demo](https://www.youtube.com/watch?v=RS36gBEp8OI) 里是直接在 widget 的 `onTap` callback 里调用另一个 bloc 的方法(`CartBloc.addition`)，这样其实把业务逻辑也耦合到了 UI 里面（如果 CartBloc 的 addition 逻辑变了，或者在添加到 Cart 的同时，还要更新用户状态等，就需要在这个 callback 里调整这些逻辑），因此更好的方法是自己的 bloc 处理完后向上抛 Notification，外层接收到 Notification 后再去更新其他 Bloc。就好像要进行跨部门沟通时，最好让共同的上级知道这件事，或者由他来协调。

这里简单说下 Flutter 里的 Notification 机制，它不像 iOS 里的 `NotificationCenter` 可以全局接收，而是只有在 widget 上层路径中的 `NotificationListener` 才能收到通知，这样可以避免通知泛滥的情况。而且使用时，必须继承系统的 `Notification`，这样每一个通知就是一个特定类，阅读代码或排查起来也会很方便。

至此，首页的电影列表页基本完成了，该进入详情页了，在这之前，还有一件事情要做，那就是 Router 的注册。

### Router 注册

`MaterialApp` 自带了 Router 注册，乍一看还挺方便的，不过有一个坑就是不支持通过 URL 传递动态参数，比如 `/movie/1024`，必须完全匹配才可以。如果真要实现 pattern 匹配就要设置 `onGenerateRoute` 参数，当通过 Navigator 进行 push 时，这个参数对应的方法就会被触发，可以在这个方法里面进行 URL 的解析。

我在这里实现了个简单的通过 URL 注册 Widget 的方法，URL 支持 pattern，比如 `/movie/{id}` 就能匹配 `/movie/1024`，使用时，通过 URL 来拿 widget，再把这个 widget push 出去即可。

```dart
class Router {
  static var _routerEntry = _Router(name: '');

  // param should wrap with {}, eg: /movie/{id}
  static register(String pattern, WidgetBuilder widgetBuilder) {
    final patternSections = pattern.split('/');
    var routerEntry = _routerEntry;
    for (int i = 0; i < patternSections.length; i++) {
      final _pattern = patternSections[i];
      final _router = _Router(name: _pattern);
      if (i == patternSections.length - 1) {
        _router.widgetBuilder = widgetBuilder;
      }
      routerEntry.addChild(_router);
      routerEntry = _router;
    }
  }

  static Widget getWidget(String url, BuildContext context, {Map params}) {
    final urlSections = url.split('/');
    var routerEntry = _routerEntry;
    Widget widget;
    Map urlParams = {};

    for (int i = 0; i < urlSections.length; i++) {
      final _urlSection = urlSections[i];
      var found = false;
      for (_Router _router in routerEntry.children) {
        if (_router.name == _urlSection || _router.name.startsWith('{')) {
          if (_router.name.startsWith('{')) {
            final param = _router.name.substring(1, _router.name.length - 1);
            urlParams[param] = _urlSection;
          }
          found = true;
          routerEntry = _router;
          if (i == urlSections.length - 1) {
            if (_router.widgetBuilder != null) {
              widget =
                  _router.widgetBuilder(context, urlParams, params: params);
            }
          }
        }
      }

      if (!found) {
        break;
      }
    }

    return widget;
  }
}
```

路由的注册

```dart
class Routes {
  static String root = '/';
  static String detail = '/movie/{id}';

  static configureRoutes() {
    Router.register(root, (BuildContext context, Map urlParams, {Map params}) {
      return Home();
    });

    Router.register(detail, (BuildContext context, Map urlParams,
        {Map params}) {
      return MoviePage(
        movieID: urlParams['id'],
        movie:
            params != null && params['movie'] != null ? params['movie'] : null,
      );
    });
  }
}
```

### 详情页-布局选择

详情页主要分为 4 个部分，顶部的海报图，中间的概要、横向可滚动的影人列表页和 Tab 标签，以及最下面的列表页。有两个方案可以选，`NestedScrollView` 和 `CustomScrollView`，前者分为 header 和 body 两部分，可以在里面套 scrollView，最开始选择了这个方案，后来发现 `TabbarView` 怎么都处理不好，如果把它单独放到 body 里，那么所有剩下的部分就都要放到 header 里，虽然可行，但跑起来会发现，`TabbarView` 里的 scroll offset 是错的，一部分内容会「钻」进 Tabbar 里，而且底部空了很大一部分。如果把除了海报图，剩下的部分都放到 body 里，也不行，被迫只能选择 `CustomScrollView`。

`CustomScrollView` 方案的一个难点是处理 `TabbarView`，因为不能直接把它放到 `slivers` 里，于是换了个思路，抛弃 `TabbarView` 手动实现 tabbar 点击之后，下面的内容切换效果。

详情页的核心代码大概像这样：

```dart
Widget build(BuildContext context) {
  return Scaffold(
      backgroundColor: Color(0xfff4f4f4),
      body: BlocProvider(
        bloc: bloc,
        child: (() {
          if (movie == null) {
            return Center(
              child: Text('loading'),
            );
          } else {
            return NotificationListener<TabSwitchNotification>(
              onNotification: (notification) {
                setState(() {
                  currentTabType = notification.tabContentType;
                });
              },
              child: DefaultTabController(
                  length: 2,
                  child: SafeArea(
                      top: false,
                      child: CustomScrollView(
                        slivers: <Widget>[
                          MovieHero(),
                          SliverToBoxAdapter(
                            child: MovieSummary(),
                          ),
                          SliverToBoxAdapter(
                            child: MovieActors(),
                          ),
                          SliverPadding(
                            padding: EdgeInsets.all(7.0),
                          ),
                          MovieReviewTabbar(),
                          MovieReviewTabbarContent(
                            tabContentType: currentTabType,
                          ),
                        ],
                      ))),
            );
          }
        }()),
      ));
}
```

对于 tabbar 来说，需要提供一个 controller，要么通过属性设置，要么外面包一层，这里选择了后者，所以可以看到最外面是 `DefaultTabController`，这样就可以在里面通过 `DefaultTabController.of(context)` 来拿到这个 controller，进而了解当前选中的 tab，以及控制 tab 的选中情况。

普通的 Widget 要通过 `SliverToBoxAdapter` 转换才能被放到 slivers 里面，slivers 其实就是 `CustomScrollView` 的组成部分。通过上面的代码，我们可以知道这个 scrollview 是由哪几部分组成的，非常清晰。

### 详情页-顶部海报效果

这个效果看上去满复杂的，实现起来很简单，只要使用 [SliverAppBar](https://docs.flutter.io/flutter/material/SliverAppBar-class.html) 这个 Widget 即可。[这里](https://medium.com/@diegoveloper/flutter-collapsing-toolbar-sliver-app-bar-14b858e87abe) 有比较详细的介绍。

### 详情页-影片介绍

这里主要是使用 Text Widget，一个难点是默认内容是截断和收起的，当点击按钮后，展开。如果是在 iOS 里，需要分别计算两者的高度，然后调用 `reloadRowsAtIndexPaths` 方法，有点麻烦，Flutter 里很简单，widget 自己更新内容即可，不需要考虑高度的计算，也不要显式地调用 reload 方法。

### 详情页-影人列表

这是一个横向滑动列表，把 `ListView` 的 `scrollDirection` 设置为 `horizontal` 即可

### 详情页-Tabbar

Tabbar 需要有一个 controller，我们在外面包了一层 `DefaultTabController`，这里就不需要操心了。比较坑的一点是，`TabBar` 这个 Widget 没有提供 `onTap` 方法，只能通过监听 controller 来获取 tab 的变化，不过这样也好，管理起来更方便。

PS: 这里的 Tabbar 是需要吸顶的，因此要用 `SliverPersistentHeader` 包一下。

### 详情页-点评列表

点评列表是一个 `SliverList`，配置起来还算简单，那么点击 tab 切换内容这个该如何实现呢？可以思考下。

### 小结

我个人很喜欢 Flutter， 用它来写 GUI 感觉非常自然和舒服。不需要借助 JSX 或 XML，用编程语言就能搞定，方便 Local Reasoning，不需要在 JSX / XML 和 Code 之间来回切换。同时通过 Widget 来配置视图的方式也很方便，GUI 非常适合这种声明式编程。

刚开始接触 Flutter 时，容易被那一大堆的类搞晕，其实了解了核心理念后，啃透几个 Demo，就会慢慢找到感觉。自己再多写写，踩踩坑，就熟练了。

至于性能方面，debug 跟 release 模式还是会有些差距，因此如果在 debug 模式下发现不够流畅，可以切换到 release 模式再试下。

我也是接触 Flutter 不久，如果有不对的地方欢迎指正，如果能给你带来些帮助，备感荣幸。
