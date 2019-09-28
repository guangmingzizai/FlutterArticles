# Widget基础系列 - StatefulWidget

在Flutter中，Widget可以说是第一基础概念。Widget是对用户界面的不可变描述，可被膨化为管理底层渲染树的Element。

理解Widget原理是掌握Flutter编程至关重要的一步，本系列主要介绍Widget的基础知识，本文是第二篇：

## 在Flutter框架中，Widget处于中心位置。Widgets是对用户界面的不可变描述，可被膨胀为管理底层渲染树的elements。

理解Widget原理是掌握Flutter编程至关重要的一步，本系列主要介绍Widget的基础知识，本文是第二篇：

- StatelessWidget
- **StatefulWidget**
- InheritedWidget
- Key

## StatefulWidget

本篇介绍有状态的widget -- 与无状态widget的区别， `State`对象的工作原理等。

看过本系列第一篇文章的，应该对无状态widget有所了解了，如果还没有看，或者还不了解无状态widget，建议先看第一篇。

我们知道，`Element`表示屏幕上的实际内容，而widget是element的不可变配置或蓝本。你肯定会有这样的疑问，我们的app不可能只是展示静态不变的内容，如何处理可变数据呢？如何跟踪数据并刷新UI你？这就需要用到`StatefulWidget`了。它除了提供不可变的配置信息之外，还提供了一个可随时间变化以及触发UI刷新的state对象。

我们通过一段简单的代码来看一下它是如何工作的。

```dart
class ItemCount extends StatelessWidget {
  final String name;
  final int count;

  ItemCount({this.name, this.count});

  @override
  Widget build(BuildContext context) {
    return Text('$name: $count');
  }
}
```

这是一个非常简单的无状态widget，构造函数接受两个参数：name和count，并构建了一个文本widget将它们显示出来。现在，我们希望`count`可以变化。在无状态widget中，我们无法修改任何东西，对吧？`count`是`final`类型的。

因此，我们将它转变为一个有状态的widget：

```dart
class ItemCounter extends StatefulWidget {
  final String name;

  ItemCounter({this.name});

  @override
  _ItemCountState createState() => _ItemCountState();
}

class _ItemCountState extends State<ItemCounter> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return Text('${widget.name}: $count');
  }
}
```

现在有了两个类，一个widget类和一个state类。Widget类有两个职责：持有不可变的`name`，以及创建`state`对象。而state对象呢，持有`count`，可以注意到，`count`不再是`final`的，而是可变的，并且子widget此时由state对象负责创建。`Text`将widget中不可变的`name`和state中可变的`count`组合之后显示出来。

要了解这是如何工作的，让我们来看一下widget和element树。

![stateless_widget_element_tree](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/stateless_widget_element_tree.jpg)

看过第一篇文章的应该知道，element树表示屏幕上的实际显示内容，widget仅仅是element的蓝本。对于无状态widget来说，显示过程是非常直接的。你给Flutter一个无状态widget，Flutter向这个widget请求一个element，并将其挂载到element树上。如果这个无状态widget构建了子widget，则同样向它们请求element对象，并挂载到element树上。

对于有状态widget来说，有一个额外的步骤。同无状态widget一样，首先从widget开始，Flutter请求有状态widget创建一个element对象，有状态widget返回一个`StatefulElement`对象。然后，这个`StatefulElement`对象向widget对象请求一个state对象，这就是`createState`方法的用途。此方法返回一个新的state对象，element对象会持有这个对象。

![widget_element_state](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/widget_element_state.jpg)

开始构建子widget，`StatefulElement`调用state对象的`build`方法。再看一下前面的代码，为了构建文本，我们需要widget中的`name`属性和state对象的`count`属性。因为state对象维护了一个对widget对象的引用，因此它可以访问这两个值以构建文本。就是这样：

![item_counter_widget_element_tree](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/item_counter_widget_element_tree.jpg)

`Text`是无状态的，因此它创建了一个`StatelessElement`，并挂载到element树上。从技术上说，`Text`还有一些自己的子widget，以提供辅助选项、渲染文本等功能。不过，对于这个简单的例子来说，我们不再深入，keep it simple。

一切都已创建，element树可以开始工作了。不过，再看一下state对象，此时还没有对象更新state，没有什么改变`count`属性。

```dart
class _ItemCountState extends State<ItemCounter> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return Text('${widget.name}: $count');
  }
}
```

如果放入一个`GestureDetector`，就可以通过state对象的`setState`方法触发更新了。

```dart
class _ItemCountState extends State<ItemCounter> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () {
        setState(() {
          count++;
        });
      },
      child: Text('${widget.name}: $count'),
    );
  }
}
```

`setState`是更新属性并刷新UI的一种方式，向它提供一个更新属性的函数，state对象运行此函数并刷新UI。

![item_counter_widget_element_tree_setstate](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/item_counter_widget_element_tree_setstate.jpg)

看一下图表，当`setState`运行时，`count`加1，此处有一个关键点，state对象标记element为dirty，表示下一帧时需要重建子节点。当下一帧时，像之前一样，`StatefulElement`调用state对象的`build`方法重建子节点，输出一个新的`Text`以显示新的`count`值。酷的地方是，由于更新前后widget的类型一样 -- 都是`Text`，`StatelessElement`依然保留在原处，仅更新widget引用到新的widget而已。

这是state对象持有可变数据，以及数据更新时重建子widget的基本例子。

对于state对象来说，还有很重要的一点 -- 它们的生命周期比较长。只要更新前后的widget类型不变（准确地说，还需要Key值相等，不过本文不涉及Key，可以暂时忽略，详见第四篇文章），当widget对象被新的对象替换时，state对象依然会附着于element树上。举例来说，当`ItemCounter`对象由于树上层的变化而重建时（*name变了*），原来的`ItemCounter`被移去，不过由于新的对象类型相同，`StatefulElement`和state对象依然保留在原处。

![item_counter_widget_element_tree_update_name](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/item_counter_widget_element_tree_update_name.jpg)

它们从widget的更新中存活下来，仅标记自己为dirty以重建子节点。然后，state对象的`build`方法给出一个显示其`count`值的新`Text`，但name换成了新widget中的值。旧的`Text`对象被移去，新的被挂载上去，而`Text`对应的element对象依然保留在原处。这就是当创建state对象的widget被替换时状态还能保持的原因。就像热更新，向设备推送新的代码而不改变应用的状态。

> State对象的生命周期与Widget不同，这一点必须谨记在心。

在这儿，我们使用新的属性构建新的widget，但是state对象不变。`State`类还有一个方法，如果state对象希望知道widget对象何时被替换，可以重写`didUpdateWidget`方法。例如，对于上面的`ItemCounter`，有如下代码：

![item_counter_app](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/item_counter_app.png)

当我们点击“Change Name”按钮切换名称时，正如前面介绍的那样，`count`值并不会改变。如果我们希望当widget的`name`改变时将`count`值清空，该如何做呢？

```dart
class _ItemCountState extends State<ItemCounter> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () {
        setState(() {
          count++;
        });
      },
      child: Text('${widget.name}: $count'),
    );
  }

  @override
  void didUpdateWidget(ItemCounter oldWidget) {
    if (oldWidget.name != widget.name) {
        count = 0;
    }
    super.didUpdateWidget(oldWidget);
  }
}
```

Flutter框架在调用`didUpdateWidget`方法之后会调用`build`方法，因此在`didUpdateWidget`方法中调用`setState`是多余的。

我们可以看到，`StatefulWidget`使我们可以很方便地跟踪数据的变化并刷新UI。不过，随着对Flutter的运用愈发熟练，我们会发现越来越少需要自己写`StatefulWidget`。一个原因是，很多常用的功能已经实现过了。例如，如果有一个数据流，我们需要一个当数据流输出新数据时能自动刷新的`StatefulWidget`，OK，Flutter框架中有一个`StreamBuilder`。

```dart
StreamBuilder(
  stream: _myStreamOfStrings,
  builder: (context, snapshot) {
    Text(snapshot.data ?? 'loading...');
  }
)
```

另一个原因是，如果嵌套了很多`StatefulWidget`，通过这些widget的`build`和构造方法传递数据是非常笨重的。

![stateful_widgets](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/stateful_widgets.jpg)

幸运的是，在Flutter还有一种类型的widget，使我们可以轻松访问树中上层的数据，哪怕是隔了100层也没关系。这就是`InheritedWidget`，下篇文章会介绍它。



