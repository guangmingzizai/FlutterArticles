# Widget基础系列 - InheritedWidget

在Flutter中，Widget可以说是第一基础概念。Widget是对用户界面的不可变描述，可被膨化为管理底层渲染树的Element。

理解Widget原理是掌握Flutter编程至关重要的一步，本系列主要介绍Widget的基础知识，本文是第三篇：

- StatelessWidget
- StatefulWidget
- **InheritedWidget**
- Key

## 简介

前面讲过了StatelessWidget和StatefulWidget，当我们的App变得越来越大、Widget树越来越复杂时，传递和访问数据会变得非常繁琐。

![widget_tree](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/widget_tree.jpg)

假设每个Widget嵌套了四、五个Widget，我们希望在最底层的Widget访问顶层Widget的数据，如果只使用StatelessWidget和StatefulWidget，我们不得不将数据添加在每个Widget的构造函数里，如果需要添加、修改、删除数据，那简直就是噩梦，一串构造函数都需要修改。你应该已经猜到了，`InheritedWidget`就是解决这个问题的。

![widget_tree_inherited](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/widget_tree_inherited.jpg)

当我们将`InheritedWidget`添加到Widget树中时，在其下面的所有Widget都可以得到一个指向它的引用，因此我们称其为Inherited Widget。

假设有一个InheritedWidget的子类：InheritedAppState：

```dart
class InheritedAppState extends InheritedWidget {
  final AppState appState;
  
  InheritedAppState({this.appState, Widget child}): super(child: child);
  
  @override
  bool updateShouldNotify(_InheritedStateContainer old) => true;
  
  static InheritedAppState of(BuildContext context) =>
    context.inheritFromWidgetOfExactType(InheritedAppState);
}
```

在子孙Widget中，可以直接通过`context.inheritFromWidgetOfExactType(InheritedAppState)`获取`InheritedAppState`对象。为了提升代码简洁性和易读性，InheritedWidget一般会包含一个静态的`of`方法。

我们注意到，InheritedWidget是**不可变**的，因此上面的`appState`属性是`final`类型的。当InheritedWidget的数据改变时，我们只能重建整个widget。请谨记这一点，所有的Widget都是不可变的，不要改变widget中保存的数据。

`final`仅表示属性不能被再次赋值，不代表内部不能改变。比如，我们可以将附加的数据模型改为一个service对象，这个service可以访问数据库、网络请求、本地assets资源等。

```dart
class InheritedAppState extends InheritedWidget {
  final AppStateService service;
  
  InheritedAppState({this.service, Widget child}): super(child: child);
  
  @override
  bool updateShouldNotify(_InheritedStateContainer old) => true;
  
  static InheritedAppState of(BuildContext context) =>
    context.inheritFromWidgetOfExactType(InheritedAppState);
}
```

简单来说，`InheritedWidget`提供了一种在widget树中从上到下传递、共享数据的方式。Flutter SDK正是通过InheritedWidget来共享应用主题和Locale等信息的。

## 原理

对于`InheritedWidget`来说，需要实现：

- 在子孙widget中可以方便地获取到`InheritedWidget`或者保存在`InheritedWidget`中的数据
- 当`InheritedWidget`中的数据改变时，更新所有依赖的子孙widget

下面我们分别介绍实现原理。

### 获取InheritedWidget对象

我们已经知道，通过`context.inheritFromWidgetOfExactType`可以在子孙widget中获取上层的InheritedWidget对象，具体是如何实现的呢？

`BuildContext`实际上就是`Element`，其相关实现代码如下：

```dart
Map<Type, InheritedElement> _inheritedWidgets;

@override
  InheritedWidget inheritFromElement(InheritedElement ancestor, { Object aspect }) {
    assert(ancestor != null);
    _dependencies ??= HashSet<InheritedElement>();
    _dependencies.add(ancestor);
    ancestor.updateDependencies(this, aspect);
    return ancestor.widget;
  }

  @override
  InheritedWidget inheritFromWidgetOfExactType(Type targetType, { Object aspect }) {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    final InheritedElement ancestor = _inheritedWidgets == null ? null : _inheritedWidgets[targetType];
    if (ancestor != null) {
      assert(ancestor is InheritedElement);
      return inheritFromElement(ancestor, aspect: aspect);
    }
    _hadUnsatisfiedDependencies = true;
    return null;
  }

  void _updateInheritance() {
    assert(_active);
    _inheritedWidgets = _parent?._inheritedWidgets;
  }
```

可以看到，在Element中，有一个Map类型的属性`_inheritedWidgets`，键是`Type`，而值是`InheritedElement`。当我们调用`inheritFromWidgetOfExactType`方法时，直接从这个Map中查找对应类型的`InheritedElement`，并调用其`updateDependencies`方法注册依赖，然后返回其关联的widget对象。

那么`_inheritedWidgets`的值从哪儿来呢？从parent那儿来。那parent的值又从哪儿来呢？从距离最近的`InheritedElement`祖先那儿来。`InheritedElement`中的`_inheritedWidgets`又是如何维护的呢？

```dart
@override
  void _updateInheritance() {
    assert(_active);
    final Map<Type, InheritedElement> incomingWidgets = _parent?._inheritedWidgets;
    if (incomingWidgets != null)
      _inheritedWidgets = HashMap<Type, InheritedElement>.from(incomingWidgets);
    else
      _inheritedWidgets = HashMap<Type, InheritedElement>();
    _inheritedWidgets[widget.runtimeType] = this;
  }
```

很简单，获取parent中的值，并注册自身。

因此，一个普通`Element`中的`_inheritedWidgets`，保存的就是其所有`InheritedElement`祖先节点的信息，通过这个属性，我们就可以方便地、高效率地获取到树上层的InheritedWidget了。

### 更新子孙widget

这个相对简单，不涉及新的知识，当上层的`InheritedWidget`变化时，其widget子树会重建，子孙widget在其`build`方法中就可以获取到新的值了。

为了确保子孙widget中的数据能及时更新，只应该在build方法、layout和paint回调以及`State.didChangeDependencies`方法中调用`inheritFromWidgetOfExactType`方法。

不应该在widget的构造方法或者`State.initState`方法中调用`inheritFromWidgetOfExactType`方法，因为当数据变更时，这些方法不会被再次调用。也不应该在`State.dispose`方法中调用，因为此时element树是不稳定的。如果必须要在`State.dispose`中访问上层的InheritedWidget，可以提前在`State.didChangeDependencies`方法中保存一个引用。在`State.deactivate`方法中调用是安全的，此方法在widget从树中移除时被调用 。

如果我们希望在`InheritedWidget`更新时做一些处理怎么办？比如从网络获取数据，如果放在`build`中代价显然太高了，从设计模式的角度来说也极其不合理，`build`方法最好只负责构建widget树。

`State`有一个`didChangeDependencies`方法，当其依赖的对象发生改变时会被调用，最主要的场景就是`InheritedWidget`依赖。此方法在`initState`方法结束之后也会被马上调用一次，可以在这个方法中安全地调用`BuildContext.inheritFromWidgetOfExactType`方法。子类很少需要重写此方法，因为框架总会在依赖改变时调用`build`方法。只有当需要执行网络请求这样的昂贵操作时，才需要重写此方法。

