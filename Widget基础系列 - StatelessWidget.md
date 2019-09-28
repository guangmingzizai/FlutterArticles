# Widget基础系列 - StatelessWidget

在Flutter中，Widget可以说是第一基础概念。Widget是对用户界面的不可变描述，可被膨化为管理底层渲染树的Element。

理解Widget原理是掌握Flutter编程至关重要的一步，本系列主要介绍Widget的基础知识，本文是第一篇：

- **StatelessWidget**
- StatefulWidget
- InheritedWidget
- Key

## 什么是Widget?

我们知道，Flutter是Google推出的一个跨平台的移动App开发方案，让我们可以用一套代码库同时开发iOS和Android应用。

Widget是Flutter应用的基本构造单元，每个Widget表示对一块用户界面的不可变描述。Widget可以做很多事情，即有按钮、菜单这样的结构化Widget，也有传递字体或者主题颜色的样式Widget，还有padding这样的布局Widget...我们还可以通过组合已有的Widget来构建新的Widget，组合无穷无尽。

看一个简单的例子，在屏幕上显示出我家猫咪“Tigger"的名字：

```dart
import 'package:flutter/material.dart';

void main() => runApp(CatApp());

class CatApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'My Cat',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: Scaffold(
        appBar: AppBar(
          title: Text('My Cat'),
        ),
        body: Center(
          child: Text('Tigger'),
        ),
      ),
    );
  }
}
```

效果如下：

![tigger_name](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/tigger_name.png)

如果我们想给猫的名字加一个背景颜色，可以将`Text`用`DecoratedBox`包裹起来：

```dart
body: Center(
  child: DecoratedBox(
    decoration: BoxDecoration(color: Colors.lightBlueAccent),
    child: Text('Tigger'),
  ),
),
```

![tigger_colored_name](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/tigger_colored_name.png)

如果我们希望在背景颜色和文字之间加一个边距，可以在两个Widget中间加一个`Padding`：

```dart
body: Center(
  child: DecoratedBox(
    decoration: BoxDecoration(color: Colors.lightBlueAccent),
    child: Padding(
      padding: const EdgeInsets.fromLTRB(8, 4, 8, 4),
      child: Text('Tigger'),
    ),
  ),
),
```

![tigger_colored_name_padding](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/tigger_colored_name_padding.png)

将widget装配起来的过程就是我们上面所说的“组合”。用户界面由一系列的widget组合而成，每个widget处理一个特定的任务。`Padding`负责添加内边距，`DecoratedBox`装饰一个box...

## StatelessWidget

假设我又收养了一对猫，需要将它们的名字也显示出来。可以用一个`Column`将它们纵向排列起来：

![three_cat_names](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/three_cat_names.png)

名字挤在了一起，在名字中间加一些空白：

![three_cat_names_margin](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/three_cat_names_margin.png)

看起来好多了，不过重复的代码有点儿多，如果创建一个自定义Widget将细节封装起来，只需要传入一个名字，就像上面使用的`Text`那样是不是更好？

```dart
class CatName extends StatelessWidget {
  final String name;

  const CatName(this.name);

  @override
  Widget build(BuildContext context) {
    return DecoratedBox(
      decoration: BoxDecoration(color: Colors.lightBlueAccent),
      child: Padding(
        padding: const EdgeInsets.fromLTRB(8, 4, 8, 4),
        child: Text(name),
      ),
    );
  }
}
```

如此一来，上图的代码可以简化为：

![three_cat_names_encapsulated](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/three_cat_names_encapsulated.png)

界面显示效果是一样的，通过使用一个`StatelessWidget`和Flutter的组合功能，代码看起来紧凑多了。

我们创建了一个名为“CatName”的`StatelessWidget`，`StatelessWidget`是一个由子widget组合而成的widget，因此它有一个`build`方法，且没有需要跟踪的可变状态。

“可变状态”是什么意思呢？随时间变化的任何属性。例如，一个文本输入框，它有一个可变的字符串属性来跟踪用户的输入。或者一个带动画的widget，某些值随动画而改变。`CatName`没有这些，它只需要一个字符串类型的名字，不会改变，因此`StatelessWidget`非常适合它。我们可以将`name`声明为`final`类型，通过构造函数接受此参数。由于所有的参数都是`final`的，因此我们可以将构造函数也声明为`final`的。

我们已经看到`build`方法是如何工作的，那么它是什么时候被调用的呢？

我们倾向于将用Flutter构建的App想象成一棵widget组成的树，这并不是坏事。不过，正如前面提到的，widget仅仅是一块用户界面的配置信息或蓝本。那么，它们配置的又是什么呢？Element。Element是widget实例化并挂载到屏幕上的对象，element树表示给定时刻屏幕上实际显示的内容，一个element表示用对应的widget配置树中特定位置的元素。

每个widget类都有对应的element类，以及用于创建element实例的方法。例如，`StatelessWidget`对应`StatelessElement`。当widget被挂载到树上时， `createElement`方法会被调用。

```dart
abstract class StatelessWidget extends Widget {
  @override
  StatelessElement createElement() => StatelessElement(this);
}
```

Flutter向widget请求一个element对象，然后将其放到element树上，element会引用创建它的widget对象。

![widget_element](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/widget_element.jpg)

在上面的`CatApp`例子中，每个widget创建自己的element，并挂载到element树上。

![CatApp_widgets_elements](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/CatApp_widgets_elements.jpg)

因此，`CatApp`有两棵树：element树，表示屏幕上的实际内容；widget树，创建element树的蓝本。

> Element并不负责布局和渲染，因此还有一棵树：RenderObject树。说element表示屏幕上的实际内容也说得过去，只不过抽象层次不同而已。

Element的构建过程是如何开始的呢？或者说，是什么开启了这一切？我们看一下最开始的代码：

```dart
void main() => runApp(CatApp());
```

`CatApp`表示整个应用程序，它是一个`StatelessWidget`。在Flutter中，Widget几乎可以做任何事情。我们看到`main`函数，它是应用的入口，`main`调用了`runApp`函数，这就是应用的起点。`runApp`接受一个widget对象，将其装载为应用的根element，大小与屏幕(准确地说是FlutterView)相同。Flutter依次调用所有的`build`方法创建widget对象，并通过它们创建element对象，直到一切都被构建并装载到屏幕上，准备好被布局和渲染 -- 然后我们就看到了屏幕上的内容。

![CatApp](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/CatApp.jpg)

## 参考资料

- [Flutter Widget 101](https://www.youtube.com/results?search_query=flutter+widgets+101)

