---
layout: post
title: Architecture Flutter App the Bloc_Redux Way
category: tech
tags: 技术
---

这是[项目地址](https://github.com/lzyy/bloc_redux)，下面来阐述下产生背景和它的一些特点。

接触 Flutter 也有一段时间了，在如何管理状态和处理数据流这块，并没有一个可以直接拿来用的现成方案。好吧，其实有，一个是 [flutter_redux](https://github.com/brianegan/flutter_redux)，一个是 [flutter_bloc](https://github.com/felangel/bloc)。先来说说 flutter_redux，这个可以算是 redux 在 flutter 的官方实现了，主要由两部分组成: `StoreProvider` 和 `StoreConnector`，前者用来保存 store，后者用来响应新的 state，看一个代码片段：

```dart
// Every time the button is tapped, an action is dispatched and
// run through the reducer. After the reducer updates the state,
// the Widget will be automatically rebuilt with the latest
// count. No need to manually manage subscriptions or Streams!
new StoreConnector<int, String>(
  converter: (store) => store.state.toString(),
  builder: (context, count) {
    return new Text(
      count,
      style: Theme.of(context).textTheme.display1,
    );
  },
)
```

这段代码的问题在于只要 reducer 有更新 state，那么所有消费该 Store 的 Connector 就会被 rebuild，哪怕这个 state 有 10 个属性，而 reducer 只是改了其中的一个 bool 值。

```dart
// Creates the base [NextDispatcher].
//
// The base NextDispatcher will be called after all other middleware provided
// by the user have been run. Its job is simple: Run the current state through
// the reducer, save the result, and notify any subscribers.
NextDispatcher _createReduceAndNotify(bool distinct) {
  return (dynamic action) {
    final state = reducer(_state, action);

    if (distinct && state == _state) return;

    _state = state;
    _changeController.add(state);
  };
}
```

这是 redux 这个 library 里的 Notify 机制，采用的是 `==` 判断，这就是问题。在 [react-redux](https://github.com/reduxjs/react-redux) 中，这块是有优化的，通过 `connect` 的 `mapStateToProps`，可以让 Component 指定关心 State 的哪些属性，然后在 react-redux 内部会对 `mapStateToProps` 的返回值和上次保存的进行比较，如果不一样再 rebuild，这样的好处是只有当 Component 关心的哪些属性真的变化时才进行 render。而 flutter_redux 无法做到这点(可能跟 flutter 不让用反射有关)，效率上就会打折扣。

再来看看 [flutter_bloc](https://github.com/felangel/bloc)，这也是关注度蛮高的一个项目，说这个之前先说说 bloc，这是 flutter 提的一个概念，运行机制大致如下：

![](/image/movie250-bloc.png)

它更像一个提案，缺少标准和实现。flutter_bloc 就是对这个提案的一个实现。这个实现本质上没觉得跟 flutter_redux 有太大的区别，而复杂度倒是增加了不少，还提出了一些新的概念(比如 BlocSupervisor, BlocDelegate, Transation)，增加了理解上的困难。在处理核心的 state 问题上依旧跟 flutter_redux 一样，甚至都没有做 `==` check。

```dart
void _bindStateSubject() {
  Event currentEvent;

  (transform(_eventSubject) as Observable<Event>).concatMap((Event event) {
    currentEvent = event;
    return mapEventToState(_stateSubject.value, event);
  }).forEach(
    (State nextState) {
      final transition = Transition(
        currentState: _stateSubject.value,
        event: currentEvent,
        nextState: nextState,
      );
      BlocSupervisor().delegate?.onTransition(transition);
      onTransition(transition);
      _stateSubject.add(nextState);
    },
  );
}
```

可以看到在往 `_stateSubject` 里塞 nextState 时甚至都没有跟之前的 state 进行判断。同时从作者的意图上是希望多个 bloc 一起使用的，这也会造成使用上的不便（比如我这个 Event 到底应该 dispatch 给哪个 bloc？）。

```dart
return BlocBuilder<LoginEvent, LoginState>(
  bloc: widget.loginBloc,
  builder: (
    BuildContext context,
    LoginState loginState,
  ) {
    if (_loginSucceeded(loginState)) {
      widget.authBloc.dispatch(Login(token: loginState.token));
      widget.loginBloc.dispatch(LoggedIn());
    }
  }
);
```

综上，这两格 Library 都无法满足我，只能再造一个轮子了。

## Bloc_Redux

其实只要让 flutter_redux 能够更高效地把状态变化传递给 widgets，问题就解决了。那如何做呢？返回一个新的 state，也就是 reducer 之路，应该是行不通了，因为无法高效地找到变化过的属性，即使可以，还要维护一个属性跟 widgets 的 map，太复杂了。换一个想法，Flutter 不是提供了 `StreamBuilder` 么，那让 Widget 自己选择 listen 哪些 stream，然后当一个 action dispatch 过来后，这些 stream 获得相应的改变不就行了么？

![](https://raw.githubusercontent.com/lzyy/bloc_redux/master/lib/bloc_redux/bloc_redux.png)

其中处理 action 的 reducer 被替换成了 bloc，来看一下核心代码。

```dart
/// Action
///
/// every action should extends this class
abstract class BRAction<T> {
  T payload;
}

/// State
///
/// Input are used to change state.
/// usually filled with StreamController / BehaviorSubject.
/// handled by blocs.
///
/// implements disposable because stream controllers needs to be disposed.
/// they will be called within store's dispose method.
abstract class BRStateInput implements Disposable {}

/// Output are streams.
/// followed by input. like someController.stream
/// UI will use it as data source.
abstract class BRStateOutput {}

/// State
///
/// Combine these two into one.
abstract class BRState<T extends BRStateInput, U extends BRStateOutput> {
  T input;
  U output;
}

/// Bloc
///
/// like reducers in redux, but don't return a new state.
/// when they found something needs to change, just update state's input
/// then state's output will change accordingly.
typedef Bloc<T extends BRStateInput> = void Function(BRAction action, T input);

/// Store
///
/// widget use `store.dispatch` to send action
/// store will iterate all blocs to handle this action
///
/// if this is an async action, blocs can dispatch another action
/// after data has received from remote.
abstract class BRStore<T extends BRStateInput, U extends BRState>
    implements Disposable {
  List<Bloc<T>> blocs;
  U state;

  void dispatch(BRAction action) {
    blocs.forEach((f) => f(action, state.input));
  }

  dispose() {
    state.input.dispose();
  }
}
```

其中 State 被分成了 StateInput 和 StateOutput，其中 Input 部分给 Bloc，方便更新 Stream；Output 部分给 Widgets，方便接收最新数据。同时 Store 也有一个 dispose 方法，因为到时 store 会被放到 StoreProvider 里，当它被 dispose 时，可以让 store 也 dispose，让那些 stream 可以被 close。

就这么简单，我们来看一个 demo：

```dart
/// Actions
class ColorActionSelect extends BRAction<Color> {}

/// State
class ColorStateInput extends BRStateInput {
  final BehaviorSubject<Color> selectedColor = BehaviorSubject();
  final BehaviorSubject<List<ColorModel>> colors = BehaviorSubject();

  dispose() {
    selectedColor.close();
    colors.close();
  }
}

class ColorStateOutput extends BRStateOutput {
  StreamWithInitialData<Color> selectedColor;
  StreamWithInitialData<List<ColorModel>> colors;

  ColorStateOutput(ColorStateInput input) {
    selectedColor = StreamWithInitialData(
        input.selectedColor.stream, input.selectedColor.value);
    colors = StreamWithInitialData(input.colors.stream, input.colors.value);
  }
}

class ColorState extends BRState<ColorStateInput, ColorStateOutput> {
  ColorState() {
    input = ColorStateInput();
    output = ColorStateOutput(input);
  }
}

/// Blocs
Bloc<ColorStateInput> colorSelectHandler = (action, input) {
  if (action is ColorActionSelect) {
    input.selectedColor.add(action.payload);
    var colors = input.colors.value
        .map((colorModel) => colorModel
          ..isSelected = colorModel.color.value == action.payload.value)
        .toList();
    input.colors.add(colors);
  }
};

/// Store
class ColorStore extends BRStore<ColorStateInput, ColorState> {
  ColorStore() {
    state = ColorState();
    blocs = [colorSelectHandler];

    // init
    var _colors = List<ColorModel>.generate(
        30, (int index) => ColorModel(RandomColor(index).randomColor()));
    _colors[0].isSelected = true;
    state.input.colors.add(_colors);
    state.input.selectedColor.add(_colors[0].color);
  }
}
```

Store 就像人的大脑，负责接收信息做出决策，而信息的处理者就是一个个的 Bloc。再来看看 Widget 是如何接收数据，发送 action 的。

```dart
class ColorsWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final store = StoreProvider.of<ColorStore>(context);
    final colors = store.state.output.colors;

    return StreamBuilder<List<ColorModel>>(
      stream: colors.stream,
      initialData: colors.initialData,
      builder: (context, snapshot) {
        final colors = snapshot.data;
        return SliverGrid.count(
            crossAxisCount: 6,
            children: colors.map((colorModel) {
              return GestureDetector(
                onTap: () {
                  store.dispatch(
                      ColorActionSelect()..payload = colorModel.color);
                },
                child: Container(
                  decoration: BoxDecoration(
                      color: colorModel.color,
                      border: Border.all(width: colorModel.isSelected ? 4 : 0)),
                ),
              );
            }).toList());
      },
    );
  }
}
```

通过 `StreamBuilder` 来消费 state output，通过 `store.dispatch` 来发送 action，It's that simple.

最后，附上项目地址：[https://github.com/lzyy/bloc_redux](https://github.com/lzyy/bloc_redux)
