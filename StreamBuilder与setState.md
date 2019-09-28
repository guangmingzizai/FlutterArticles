# StreamBuilder与setState

## StreamBuilder简介

### Stream

如今的app是高度异步的，事件可能在任意时刻、以任意顺序发生。如用户点击了某个按钮、网络中断了、推送消息到了、内存不足了、网络请求数据返回来了等等。现代的GUI编程框架，基本上都是基于事件驱动模型的，我们通过Target-Action、Notification、Listener等方式处理各种事件。近几年，响应式编程兴起，以Stream将各种事件统一抽象起来，将各种事件看作数据流，这样使得我们可以用一种模式处理所有的事件。

Dart对异步数据流提供了很好的支持，此处不做过多介绍。

```dart
Stream<int> count() async* {
  int i = 1;
  while (ture) {
    yield i++;
  }
}
```

### StreamBuilder

在Flutter中，我们如何使用Stream呢？使用*StreamBuilder*这个widget。

```dart
StreamBuilder<int>(
  stream: _myStream,
  builder: _myBuilderFunction,
);
```

`StreamBuilder`监听来自数据流的事件，每当新事件到来时重建widget子树，并将最新的事件传递给子树的`builder`。

> **注意：**
>
> 上面这句话是不准确的，每个事件并不严格对应一次`build`，使用不当会引发严重问题，这也是此篇文章的缘起。

给`StreamBuilder`一个`Stream`对象和`builder`函数，对于给定`snapshot`返回相应的widget子树。

```dart
StreamBuilder(
  stream: _myStream,
  builder: (context, snapshot) {
    return MyWidget(snapshot.data);
  },
);
```

可以给它一个初始值，以在第一个事件到来之前显示某些数据。

```dart
StreamBuilder(
  stream: _myStream,
  initialData: 42,
  builder: (context, snapshot) {
    return MyWidget(snapshot.data);
  },
);
```

确保使用`snapshot`之前检查是否有数据，如果还没有，可以展示一个Indicator。

```dart
StreamBuilder<int>(
  stream: _myStream,
  builder: (context, snapshot) {
    if (!snapshot.hasData) {
      return CircularProgressIndicator();
    }
    return MyWidget(snapshot.data);
  },
);
```

如果想做更细粒度的控制，还可以检查`ConnectionState`。

```dart
StreamBuilder(
  stream: _myStream,
  builder: (context, snapshot) {
    switch (snapshot.connectionState) {
      case ConnectionState.waiting:
      case ConnectionState.none:
        return LinearProgressIndicator();
      case ConnectionState.active:
        return MyWidget(snapshot.data);
      case ConnectionState.done:
        return MyFinalWidget(snapshot.data);
    }
  }
);
```

另外，我们还应该一如既往地检查错误，对吧？

```dart
StreamBuilder(
  stream: _myStream,
  builder: (context, snapshot) {
    if (snapshot.hasError) {
      return UhOh(snapshot.error);
    }
    ...
  }
);
```

#### 实现原理

`StreamBuilder`的实现很简单，监听数据流的变化，并调用`setState`重建widget子树。

`StreamBuilder`是一个`StatefulWidget`，下面是`State`的实现代码：

```dart
class _StreamBuilderBaseState<T, S> extends State<StreamBuilderBase<T, S>> {
  StreamSubscription<T> _subscription;
  S _summary;

  @override
  void initState() {
    super.initState();
    _summary = widget.initial();
    _subscribe();
  }

  @override
  void didUpdateWidget(StreamBuilderBase<T, S> oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (oldWidget.stream != widget.stream) {
      if (_subscription != null) {
        _unsubscribe();
        _summary = widget.afterDisconnected(_summary);
      }
      _subscribe();
    }
  }

  @override
  Widget build(BuildContext context) => widget.build(context, _summary);

  @override
  void dispose() {
    _unsubscribe();
    super.dispose();
  }

  void _subscribe() {
    if (widget.stream != null) {
      _subscription = widget.stream.listen((T data) {
        setState(() {
          _summary = widget.afterData(_summary, data);
        });
      }, onError: (Object error) {
        setState(() {
          _summary = widget.afterError(_summary, error);
        });
      }, onDone: () {
        setState(() {
          _summary = widget.afterDone(_summary);
        });
      });
      _summary = widget.afterConnected(_summary);
    }
  }

  void _unsubscribe() {
    if (_subscription != null) {
      _subscription.cancel();
      _subscription = null;
    }
  }
}
```

在`StreamBuilder`中，只是将`Stream`的数据、错误等用`AsyncSnapshot`包装一下：

```dart
class StreamBuilder<T> extends StreamBuilderBase<T, AsyncSnapshot<T>> {
  const StreamBuilder({
    Key key,
    this.initialData,
    Stream<T> stream,
    @required this.builder,
  }) : assert(builder != null),
       super(key: key, stream: stream);

  final AsyncWidgetBuilder<T> builder;

  final T initialData;

  @override
  AsyncSnapshot<T> initial() => AsyncSnapshot<T>.withData(ConnectionState.none, initialData);

  @override
  AsyncSnapshot<T> afterConnected(AsyncSnapshot<T> current) => current.inState(ConnectionState.waiting);

  @override
  AsyncSnapshot<T> afterData(AsyncSnapshot<T> current, T data) {
    return AsyncSnapshot<T>.withData(ConnectionState.active, data);
  }

  @override
  AsyncSnapshot<T> afterError(AsyncSnapshot<T> current, Object error) {
    return AsyncSnapshot<T>.withError(ConnectionState.active, error);
  }

  @override
  AsyncSnapshot<T> afterDone(AsyncSnapshot<T> current) => current.inState(ConnectionState.done);

  @override
  AsyncSnapshot<T> afterDisconnected(AsyncSnapshot<T> current) => current.inState(ConnectionState.none);

  @override
  Widget build(BuildContext context, AsyncSnapshot<T> currentSummary) => builder(context, currentSummary);
}
```

## setState简介

`setState`相信大家都不陌生了，它是如何触发`build`调用的？每次`setState`都会调用`build`吗？

当`State`对象的内部状态改变时，我们调用`setState`通知Flutter框架预定一次`build`。传入的回调函数是同步调用的，不支持异步函数，因为那样会搞不清楚状态具体的设置时间。如果不调用`setState`而直接修改内部状态，则UI不会刷新。

```dart
setState(() { _myState = newValue; });
```

推荐只把具体的内部状态修改放在`setState`中，相关的计算逻辑或其它操作不要放在里面。比如，有一个计数器，点击之后加1并存储，那么只应该把加1放在`setState`中。

```dart
Future<void> _incrementCounter() async {
  setState(() {
    _counter++;
  });
  Directory directory = await getApplicationDocumentsDirectory();
  final String dirName = directory.path;
  await File('$dir/counter.txt').writeAsString('$_counter');
}
```

在`dispose`之后，不应该再调用此方法，否则会引发错误。

来看一下`setState`的实现代码，对一些assert逻辑做了归纳：

```dart
void setState(VoidCallback fn) {
    assert(fn != null);
  // assert before dispose and after mounted
  ...
    final dynamic result = fn() as dynamic;
  // assert result isn't Future
  ...
      // We ignore other types of return values so that you can do things like:
      //   setState(() => x = 3);
      return true;
    }());
    _element.markNeedsBuild();
  }
```

代码很简单，检查生命周期状态以及result类型，然后标记`element`需要重建。`setState`调用Element的`markNeedsBuild`，标记Element为dirty，并将其添加到一个全局的widget列表中，下一帧时这个列表中的widget会重建。

## StreamBuilder与setState

背景知识介绍完，终于可以进入主题了。只聊一个问题：每次新事件，都会`build`吗？

我们知道，屏幕是有刷新频率的，智能手机的刷新频率目前一般是60次。有如下关系：

- StreamBuilder调用`setState`刷新UI
- `setState`标记Element为dirty，在下一帧时重建widget子树

因此，如果在一帧时间内多次调用`setState`，那么只有最后一次生效。换言之，如果在一帧时间内有多个事件，那么在`builder`函数中只会接收到最后一次。

对于用户交互这样的事件来说，一帧的时间是很长的，这个问题基本可以忽略。但是对于网络请求这样的事件来说，一帧的时间又是很长的，同时发送多个请求，在同一帧时间片返回是很正常的。如果我们将每个接口的数据Push到数据流，`StreamBuilder`可能不会接收到所有的数据序列。

### 注意事项

讨论这个问题的意义何在呢？这涉及到“**如何设计状态？**”这个关键问题。

Flutter与React一样，都是基于响应式编程思想而设计的，UI就是状态的函数：

![flutter_widget_state](https://flutter.cn/assets/development/data-and-backend/state-mgmt/ui-equals-function-of-state-54b01b000694caf9da439bd3f774ef22b00e92a62d3b2ade4f2e95c8555b8ca7.png)

所以，状态的设计质量直接决定代码的质量，请务必花心思设计状态，不要着急写UI。

记住一点：**用于widget的数据流，后面的数据对前面的应该是替代关系，而不能是补充关系。**只有这样，上面的公式才成立。

## 参考资源

- [StreamBuilder (Widget of the Week)](https://www.youtube.com/watch?v=MkKEWHfy99Y)