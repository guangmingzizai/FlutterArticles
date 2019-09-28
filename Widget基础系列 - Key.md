# Widget基础系列 - Key

在Flutter中，Widget可以说是第一基础概念。Widget是对用户界面的不可变描述，可被膨化为管理底层渲染树的Element。

理解Widget原理是掌握Flutter编程至关重要的一步，本系列主要介绍Widget的基础知识，本文是第四篇：

- StatelessWidget
- StatefulWidget
- InheritedWidget
- **Key**

## 引言

你应该已经注意到了，每个widget的构造函数都有一个`key`参数，这个参数的作用是什么呢？**Key用于在widget的位置改变时保留其状态。**比如，保留用户的滑动位置，或者在保留widget状态的情况下修改一个widget集合，如Row、Column等。本篇文章，我们会讨论以下几点：

- Key是什么？
- 何时需要使用Key？
- 应该在何处使用Key？
- 应该使用什么类型的Key？

## Key是什么？

简而言之，Key是Widget、Element和SemanticsNode的标识符。

只有当新widget的key值与element当前关联widget的key值相等时，新widget才会被用于更新element。

拥有相同父节点的所有Element的key值必须是唯一的。

Flutter中常见的Key有：

```yaml
- LocalKey
  - ObjectKey
  - UniqueKey
  - ValueKey
    - PageStorageKey
- GlobalKey
  - GlobalObjectKey
  - LabeledGlobalKey
```

## 何时需要使用Key？

大多数时候不需要，不过当需要在一个相同类型的、有状态的widget集合中添加、删除或调整顺序时，就需要使用Key了。比如，在一个待办事项App中，我们需要可以执行添加新事项、根据优先级调整事项顺序、在完成事项后移除它们等操作。

先通过一个简单的例子来了解一下为什么需要使用Key。下面是一个随机数列表：

![random_num_stateless](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/random_num_stateless.png)

目前`RandomNum`是一个`StatelessWidget`：

```dart
class RandomNum extends StatelessWidget {
  final int num;
  RandomNum(): num = Random().nextInt(1000 * 1000), super();

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: EdgeInsets.all(8),
      child: Text('$num'),
    );
  }
}
```

当我们点击`Reorder`按钮时，随机数列表会重新排序，一切正常。然后我们将`RandomNum`改为`StatefulWidget`。

```dart
class RandomNum extends StatefulWidget {
  @override
  State<StatefulWidget> createState() => RandomNumState();
}

class RandomNumState extends State<RandomNum> {
  int num;

  @override
  void initState() {
    num = Random().nextInt(1000 * 1000);
    super.initState();
  }

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: EdgeInsets.all(8),
      child: Text('$num'),
    );
  }
}
```

此时，再点击`Reorder`按钮，屏幕上的数字没有变化。怎么回事呢？

我们知道，在Flutter中，每个Widget都对应一个Element。Element树其实是非常简单的，仅保存了关联的widget的类型以及指向子element的连接。可以将element树看作app的骨架，它展示了app的结构，但是所有的附加信息需要通过指向widget的引用来查找。

![random_num_stateless_tree](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/random_num_stateless_tree.jpg)

这是无状态版本的widget和element树。当我们改变`items`的顺序时，Flutter会遍历element树以确认结构是否改变，从`Column`对应的element开始，一直到所有的子孙结点。对于每个element，Flutter会检查新widget的类型和key与当前引用的widget的类型和key是否一致，如果一致就将引用指向新的widget。对于`RandomNum`来说，由于没有key，因此Flutter只会检查其类型。

![random_num_stateless_tree_swapped](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/random_num_stateless_tree_swapped.jpg)

当我们改变`items`的顺序后，widget树会重建，但由于结构与之前一致，所以element树结构并不会改变，只不过element指向的widget引用发生了变化。对于`StatelessWidget`来说，这并没有什么问题，所有的信息都保存在widget中，只要改变widget树就可以了。

![random_num_widget_element_tree_stateful](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/random_num_widget_element_tree_stateful.jpg)

当`RandomNum`变为`StatefulWidget`后，widget树、element树与之前一样，但是增加了关联的State对象。此时，`num`不再保存在widget中，而是保存在State对象中。当我们调整widget的顺序后，Flutter依然会遍历element树，检查树结构是否改变，并更新指向widget的引用。

![random_num_widget_element_tree_stateful_swapped](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/random_num_widget_element_tree_stateful_swapped.jpg)

Flutter根据element树以及关联的State对象来确定屏幕上的实际显示内容。因此，屏幕显示内容不会改变。

现在，我们给`RandomNum`增加一个Key：

```dart
class _RandomNumAppState extends State<RandomNumApp> {
  List<RandomNum> items;
  
  @override
  void initState() {
    items = [
      RandomNum(key: UniqueKey()),
      RandomNum(key: UniqueKey()),
      RandomNum(key: UniqueKey()),
      RandomNum(key: UniqueKey()),
      RandomNum(key: UniqueKey()),
      RandomNum(key: UniqueKey()),
    ];
    super.initState();
  }
  ...
}

class RandomNum extends StatefulWidget {
  RandomNum({Key key}): super(key: key);

  @override
  State<StatefulWidget> createState() => RandomNumState();
}
```

![random_num_widget_element_tree_stateful_key](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/random_num_widget_element_tree_stateful_key.jpg)

当我们改变`items`的顺序时，Flutter遍历element树以检查是否需要更新。`Column`与之前一样，直接将widget引用指向新的widget即可。对于`RandomNum`的element来说，由于element的Key值与对应widget的Key值不同，因此Flutter会使这个element暂时失效，并移除对它的引用以及其对widget的引用。

![random_num_widget_element_tree_no_reference](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/random_num_widget_element_tree_no_reference.jpg)

然后，从第一个不匹配的widget开始，Flutter会在所有失效的子结点中查找具有对应Key值的element，如果找到了，则将这个element的widget引用指向到这个widget。接着对第二个widget做相同的操作，以此类推。

![random_num_widget_element_tree_with_reference](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/random_num_widget_element_tree_with_reference.jpg)

当遍历结束之后，更新element树的引用，此时widget、element、state就对应起来了。

![random_num_widget_element_tree_stateful_updated](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/random_num_widget_element_tree_stateful_updated.jpg)

总结来说，当我们需要更新一个有状态的、同类型的widget组成的集合时需要使用Key来保留widget的状态。

## 应该在何处使用Key？

当我们需要使用Key时，应该将Key用在widget树的什么位置呢？**需要保存状态的widget子树的顶层。**我们刚才一直在谈论state，你或许会认为应该用在第一个`StatefulWidget`上，不过这是错误的!！

将上面的例子略作修改，我们将`RandomNum`这个`StatefulWidget`包裹在一个`Padding`里面：

```dart
class _RandomNumAppState extends State<RandomNumApp> {
  List<Padding> items;

  @override
  void initState() {
    items = [
      Padding(
        padding: EdgeInsets.all(4),
        child: RandomNum(key: UniqueKey()),
      ),
      Padding(
        padding: EdgeInsets.all(4),
        child: RandomNum(key: UniqueKey()),
      ),
      Padding(
        padding: EdgeInsets.all(4),
        child: RandomNum(key: UniqueKey()),
      ),
      Padding(
        padding: EdgeInsets.all(4),
        child: RandomNum(key: UniqueKey()),
      ),
    ];
    super.initState();
  }
}
```

再次运行程序，我们发现所有的数字每次都重新生成了一遍，怎么回事呢？先来看一下添加Padding之后的widget和element树。

![random_num_padding_widget_element_tree](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/random_num_padding_widget_element_tree.jpg)

当我们改变`RandomNum`结点的位置时，Flutter的widget-to-element匹配算法每次在树中查找一级。我们先看第一层，即`Padding`层，暂时忽略其它结点，每次只看一层。

![random_num_padding_only_widget_element_tree](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/random_num_padding_only_widget_element_tree.jpg)

可以看到，对于`Padding`这一层来说，调整`RandomNum`的顺序之后，匹配关系并没有发生什么变化，`Padding`还没有Key，因此只需要比较其类型，显然类型都一样，更新element指向widget的引用即可。

然后来看第二层，即第一个`Padding`对应的子树：

![random_num_padding_subtree](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/random_num_padding_subtree.jpg)

Element的key值与widget的key值不匹配，因此Flutter会使这个element失效并移除对它的连接。我们在例子中使用的是本地Key，这意味着Flutter只会在一个层级中使用这个Key值来匹配widget和element。由于无法在同级找到拥有相同Key值的element，因此Flutter会重新创建一个新的element并赋予新的State对象。所以，我们看到所有的数字都重新创建了。

如果我们将Key用在Padding上，Flutter就会感知到变化并正确地更新连接，就像上面的例子一样。

```dart
class _RandomNumAppState extends State<RandomNumApp> {
  List<Padding> items;

  @override
  void initState() {
    items = [
      Padding(
        key: UniqueKey(),
        padding: EdgeInsets.all(4),
        child: RandomNum(),
      ),
      Padding(
        key: UniqueKey(),
        padding: EdgeInsets.all(4),
        child: RandomNum(),
      ),
      Padding(
        key: UniqueKey(),
        padding: EdgeInsets.all(4),
        child: RandomNum(),
      ),
      Padding(
        key: UniqueKey(),
        padding: EdgeInsets.all(4),
        child: RandomNum(),
      ),
    ];
    super.initState();
  }
  ...
}
```

![random_num_padding_widget_element_tree_key](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/random_num_padding_widget_element_tree_key.jpg)

## 应该使用什么类型的Key？

我们已经知道何时需要使用Key，以及应该在何处使用Key。不过，如果我们看一下Flutter的文档，就会发现有很多类型的Key，那么应该使用什么类型的Key呢？

### LocalKey

当我们修改一个widget集合时，就像上面的将一组数字重新排序那样，只需要与其它widget的key区分开来即可。对于这种情况，可以根据widget中保存的信息来做选择。

#### ValueKey

相等性由其`value`属性确定。

在一个待办事项列表中，如果每个列表项的文字是唯一的，那么`ValueKey`是不错的选择，将文字作为其值：

```dart
return TodoItem(
  key: ValueKey(todo.task),
  todo: todo,
  onDismissed: (direction) {
    _removeTodo(context, todo);
  },
);
```

#### ObjectKey

相等性由其Object类似的`value`属性确定。

如果widget中保存的是复杂信息的组合呢？比如在一个通讯录App中，每个人的信息有很多项：

```yaml
AddressBookEntry:
  FirstName: Hob
  LastName: Reload
  Birthday: July 18
  
AddressBookEntry:
  FirstName: Ella
  LastName: Mentary
  Birthday: July 18
  
AddressBookEntry:
  FirstName: Hob
  LastName: Thyme
  Birthday: February 29
```

任何一项信息可能都不是唯一的，姓名、出生日期都可能重复，不过信息的组合是唯一的。对于这种情况，`ObjectKey`或许是最合适的。

#### UniqueKey

只与自己相等。

如果多个widget的值相同，或者想要确保每个Key的唯一性，可以使用`UniqueKey`。在上面的例子中使用的就是`UniqueKey`，因为我们没有在widget中保存任何不变且唯一的数据，数字需要等到`RandomNum`构建或`initState`时才能确定。

#### PageStorageKey

定义[PageStorage](https://api.flutter.dev/flutter/widgets/PageStorage-class.html)的值存放在何处的`ValueKey`。

滚动列表（ScrollPosition）使用`PageStorage`保存滚动位置，每次滚动停止时都会更新`PageStorage`中保存的值。

```dart
void didEndScroll() {
  activity.dispatchScrollEndNotification(copyWith(), context.notificationContext);
  if (keepScrollOffset)
    saveScrollOffset();
}

@protected
void saveScrollOffset(){
  PageStorage.of(context.storageContext)?.writeState(context.storageContext, pixels);
}
```

`PageStorage`用于保存和恢复比widget生命周期长的数据，这些数据保存在一个per-route的Map中，Key由widget的`PageStorageKey`和其祖先结点来决定。要使widget重建时能够找到保存的值，key值的identity在每次widget构建时必须保持不变。

例如，为了确保`TabBarView`重建时每个MyScrollableTabView的滚动位置都能被恢复，我们为其指定了`PageStorageKey`：

```dart
TabBarView(
  children: myTabs.map((Tab tab) {
    MyScrollableTabView(
      key: PageStorageKey<String>(tab.text), // like 'Tab 1'
      tab: tab,
    ),
  }),
)
```

### GlobalKey

在整个App中唯一的Key。

GlobalKey唯一标识了一个element，通过GlobalKey可以访问与此element关联的其它对象，比如`BuildContext`。对于`StatefulWidget`来说，可以通过GlobalKey访问其`State`。

与上面介绍的LocalKey不同，含有`GlobalKey`的widget在移动位置时可以改变父结点。与LocalKey相同的是，位置的移动必须在一个动画帧内完成。

例如，我们想在不同的页面展示同一个widget，同时保持其状态，就需要使用GlobalKey。

GlobalKey的成本比较高，如果不是为了上面的两个目的，即在保持widget状态的情况下更换父节点，或者需要访问在widget树中完全不同部分的widget中的信息，可以考虑使用上面介绍的LocalKey。

## 总结

当在widget树中更换位置时，使用Key来保持状态。最常见的场景是修改一个相同类型widget组成的集合，例如一个列表。将Key放在希望保持其状态的widget子树的顶部，根据widget中保存的信息类型选择合适的Key。