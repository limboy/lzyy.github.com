---
layout: post
title: 关于 Flutter Layout 你应该知道的
category: tech
tags: 技术
---

这篇文章首发于 [Medium](https://medium.com/@limboy/flutter-layout-in-a-nutshell-f2ed3cb66d72)，略显生硬的英文看来并不太妨碍理解。

与 Flutter 的布局系统搏斗一段时间之后，感觉终于找到了点门道，于是花了点时间整理了下。

### 核心概念

#### Unbounded Constraints

> either the maximum width or the maximum height is set to double.INFINITY

![](/image/flutter-scrollview.png)

`ScrollView` 和它的子类比如 `ListView` 或 `GridView` 是常见的 Unbounded Constraints. 也就是在某一个方向没有限制大小。其他的 widget 只要能够设置 `width` 或 `height` 为 `double.INFINITY` 的也算。有时也会用 **as big as possible** 来描述这些 widgets。

#### Flex

> when in bounded constraints, try to be as big as possible in that direction.
>
> when in unbounded constraints, try to fit their children in that direction.

当在有限的空间内，会撑满整个空间；如果在一个 unbounded constraints 容器里，就匹配子 widget 的大小。

![](/image/flutter-row-column.png)

最常见的是 `Row` 和 `Column`，如果不嫌麻烦的话，也可以使用 `Flex` widget。里面可以放 `Flexible` widget，也可以不是。如果有 `Flexible` widgets 会把剩余空间计算出来分配给这些 widgets。

#### Flexible

跟 `Flex` 搭配使用，`Flexible` 可以用来声明使用百分之多少的空间。比如 `flex = 1` 也就是 `1/all`，如果有两个 widgets，另一个也是 1，那么 `all = 2`，每个 widget 分配到 50% 的空间。

![](/image/flutter-expanded.png)

`Expanded` 是最常见的 `Flexible` widget，它会填满主方向上可用的空间（比如 Row 的水平空间或 Column 的垂直空间）。

### 主要 Widgets

#### Container

> Containers with no children try to be as big as possible unless the incoming constraints are unbounded, in which case they try to be as small as possible.
>
> Containers with children size themselves to their children.
>
> The width, height, and constraints arguments to the constructor override this.

这是 Container 的三个主要表现：当没有子 widgets 且没有指定 constraints 时，尽可能地充满可用空间，如果有 constraints 就以 constraints 为准（除非跟 parent constraints 冲突）；如果有子 widgets 则以 children 的 size 为准；可以设置 `width`, `height`， `constraints` 来约束 size。

```dart
return MaterialApp(
  home: Scaffold(
    body: Container(
      color: Colors.yellow,
    ),
  ),
);
```

这是一个没有孩子的 container，因此它会表现地尽量大，就像这样：

![](/image/Flutter-container.png)

如果设置了 `width` 或 `height`，则会根据设置的值来表现：

```dart
return MaterialApp(
  home: Scaffold(
    body: Container(
      color: Colors.yellow,
      width: 100,
      height: 100,
    ),
  ),
);
```

![](/image/Flutter-container-width.png)

如果有 child，则会以 child 的 size 为准：

```dart
return MaterialApp(
  home: Scaffold(
    body: Container(
      color: Colors.yellow,
      child: Text('hello'),
    ),
  ),
);
```

![](/image/Flutter-container-child.png)

除此之外，还可以设置 padding, margin, child 的对齐方式，等等。

#### Stack

`Stack` 有点像 css 的绝对布局，可以在上面盖一些 widgets，比如 profile 页的背景图上放一些个人信息。

> Each child of a Stack widget is either positioned or non-positioned.
>
> Positioned children are those wrapped in a Positioned widget that has at least one non-null property.
>
> The stack sizes itself to contain all the non-positioned children, which are positioned according to alignment.

Stack 的 children 如果没有用 `Positioned` 修饰的话，就会用 Stack 的 `fit` 和 `alighment` 来帮它们找到合适的位置。

```dart
Stack(
  fit: StackFit.loose,
  alignment: Alignment.center,
  children: <Widget>[
    Text('world'),
    Positioned(
     bottom: 10,
     child: Text('hello'),
   )
 ],
),
```

![](/image/flutter-stack-1.png)

`StackFit.loose` 的意思是，如果 child size 不比 Stack 的大，就用 child 的 size。而如果设置为 `StackFit.expand` 则会让所有非 `Positioned` 的 widgets 使用 Stack 的 size。

![](/image/flutter-stack-2.png)

`Text('world')` 现在就跟 Stack 一样大了，所以看起来像是 `alignment.center` 没有生效。

#### Row and Column

它们都是 Flex widgets，Row 可以将 children 横着放，column 可以将 children 竖着放。

`crossAxisAlignment` 表示要如何对齐另一侧，比如横着一排的 widgets，垂直方向上它们应该顶部对齐还是居中对齐呢。

`mainAxisSize` 默认是 `MainAxisSize.max`，如果想让它变成 Row 或 Column 的真实高度，可以将它设置为 `MainAxisSize.min`。

#### SizedBox

使用它可以得到一个确定尺寸的 widget。

#### SafeArea

使用 `SafeArea` 可以让 child widget 在顶部和底部腾出足够的空间方便处理 iPhoneX 这类的手机。

### 原则

#### 不要在 Flex widget 里放置 unbounded constraints

`Column` 是 Flex widget，所以在里面放 `ListView` 的话，系统不会答应的。

```dart
return MaterialApp(
  home: Column(
    children: <Widget>[
      ListView.builder(
        itemBuilder: (context, index){
          return Text('hello');
        },
        itemCount: 3,
      )
    ],
  ),
);
```

系统会给出这样的 error

```
flutter: The following assertion was thrown during performResize():
flutter: Vertical viewport was given unbounded height.
...
```

因为 Column 作为 Flex 它不知道应该如何安放一个 **as big as possible** 的 widget。解决方法也很简单，只要设置 ListView 的 `shrinkWrap=true` 即可。这就是告诉 ListView 把自己尽可能地缩小。

可以在 `Column` 或 `Row` 里使用 `Expanded`，因为它是 `Flexible`，就应该待在 Flex 里面。

#### 不要在 unbounded widgets 里设置 flex 为不等于 0 的数值

因为空间无限，如果两个 `Flexible` 分别为 1 和 2，那么 `Flex` 根本不知道该如何将空间划分成 1:2。如果真这么做的话，会收到这样的 error:

```
...
RenderFlex children have non-zero flex but incoming height constraints are unbounded.
...
```

### 小测验

下面这段代码会让 `Hello World` 被包裹在中间的小方块里吗？

```dart
return MaterialApp(
  home: Container(
    alignment: Alignment.center,
    constraints: BoxConstraints.tight(Size(100, 100)),
    decoration: BoxDecoration(color: Colors.yellow),
    child: Text('Hello World'),
  ),
);
```

![](/image/Flutter-squrebox.png)

答案是，不会，它会变成这样：

![](/image/Flutter-squarebox-real.png)

不是设置了 constraints 系统就要按着这个 constraints 来，在经过计算之后，系统会发现这个 constraints 无法满足需求，而被舍弃，具体过程如下：

`Container` 的 `build` 方法里，发现有设置过 constraints，最终会返回一个 `BoxConstraints`:

```dart
BoxConstraints(
  minWidth: minWidth.clamp(constraints.minWidth, constraints.maxWidth),
  maxWidth: maxWidth.clamp(constraints.minWidth, constraints.maxWidth),
  minHeight: minHeight.clamp(constraints.minHeight, constraints.maxHeight),
  maxHeight: maxHeight.clamp(constraints.minHeight, constraints.maxHeight)
)
```

这里的 `clamp` 方法指的是当 minWidth 值比左边的值小时，取左边值，比右边的值大时，取右边值。因为 parent 的 constraints 也就是 screen size 是固定的，因此，`minWidth` 在跟它们比较之后，还是使用了它们的值。

正确的做法是在外面套一层 `Center` 或 `Align` widget。

#### 如何得到父 widget 的 constraints？

使用 `LayoutBuilder`。有时会需要这些信息来做一些显示上的调整。

```dart
// borrowed from https://stackoverflow.com/a/41558369/94962

var container = Container(
  // Toggling width from 100 to 300 will change what is rendered
  // in the child container
  width: 100.0,
  // width: 300.0
  child: LayoutBuilder(
    builder: (BuildContext context, BoxConstraints constraints) {
      if(constraints.maxWidth > 200.0) {
        return Text('BIG');
      } else {
        return Text('SMALL');
      }
    }
  ),
);
```

#### 如何获取屏幕尺寸

使用 `MediaQuery`，除了 `size` 外，还能拿到 `devicePixelRatio` 之类的 device 信息。

### 小结

差不多就这些了，对于理解 Flutter 的布局应该够用了，希望能带来些帮助，如果有什么错误欢迎指出 :)
