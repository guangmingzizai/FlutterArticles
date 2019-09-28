# 如何实现Flutter混合开发框架 - FlutterBoost深度解析

> 由于FlutterBoost的代码质量比较差，顶多中级工程师水平，没有规范、没有注释、没有设计模式...，因此不会对其代码做深入分析，也不建议大家直接学习其源码，以防被误导。
>
> 忘其形而得其神，吸收并超越它。如果我们能从头构建一个混合开发框架，具体的代码实现也就不重要了。
>
> 要想从代码级别彻底搞懂Flutter混合开发，需要先掌握一些基础知识，在`基础知识`环节列出了关键的知识点以及相关资料的链接，请务必在掌握基础知识的基础上再看正文。

## 前言

自从移动App出现以来，业界从来没有停止过对跨平台开发方案的探索，毕竟一套代码库搞定多端诱惑力实在太大了。

在Flutter出现之前，跨平台开发主要围绕着Javascript和WebView，前端技术的优势在于开发和迭代速度快，满足小步快跑、快速试错的精益创业要求。而缺点在于，性能和体验相对比较差。尽管React Native这样的技术，通过使用OEM Widget渲染在一定程度上解决了用户体验差的问题，但也导致系统复杂度升高，各种实现问题层出不穷。以致于跨平台开发的收益可能还比不上直接开发iOS、Android原生应用，对于小团队来说尤其如此。感兴趣的可以看一下[Airbnb：我们为什么会选择放弃 React Native](https://www.oschina.net/news/100376/why-airbnb-drop-reactnative?p=2&from=singlemessage)，笔者之前开发过一年左右的React Native，那些被各种框架自身和第三方库的bug折磨的惨痛经历依然历历在目。

Flutter的实现思路不同于以往的跨平台方案，以往的方案都依赖于OEM已经提供的技术，iOS、Android都支持WebView、Javascript，那就用WebView和Javascript来实现跨平台，但是WebView、Javascript带来的跨平台优势和OEM Widget的性能、体验优势恐怕永远也无法完美融合。而Flutter呢，既然无法完美融合，那就干脆不融合了，连OEM Widget也抛开，从零开始实现一套**原生**的跨平台方案。得益于Dart同时支持JIT和AOT编译的特性，Flutter实现了前端的开发效率和原生的运行效率的完美融合。希望进一步了解的可以看：[What’s Revolutionary about Flutter](https://medium.com/hackernoon/whats-revolutionary-about-flutter-946915b09514)。

从技术上说，Flutter是近乎完美的跨平台方案，抛开各种因素干扰，从第一性原理出发，跨平台方案理应如此。以往的都是浅层的跨平台，对平台依赖度比较高，而Flutter是深度的跨平台，对平台依赖度很低。不过，iOS、Android十余年的生态积累是不可能短时间内被超越的，彻底抛开iOS、Android原生SDK开发移动App也是不现实的和划不来的，很多功能还是需要依赖iOS和Android的系统SDK。

从应用的角度来说，抛弃业界、公司十余年积累的代码库也是不现实的。对于迭代了几年的大应用来说，重构可能都颇费周折，完全了解业务的产品和开发可能早就离职了，优先级也很难挤进项目排期，完全重写是不可能的。 对于大多数快速迭代中的产品，应用Flutter的最现实方案是渐进地集成，先在个别页面小范围尝试，然后根据团队的技术水平和业务发展情况做调整，毕竟Flutter还远未成熟，恐怕并不适合所有的公司和团队。对于小公司来说，这一点尤其需要注意，任何技术的应用都是有成本和门槛的，新技术带来的复杂性和隐形成本往往是小团队难以承受的。保证业务稳定、快速迭代是第一目标，技术只是手段。

因此，Flutter和iOS、Android的混合开发是无法避免的，混合开发又分为两种情况：

- Flutter为主，只在迫不得已时调用平台SDK。通过`flutter create`命令创建的工程即是如此，对于一些必须使用平台SDK的场景，可以使用Flutter插件，比如获取设备信息、推送通知等。
- 原生工程为主，某些页面使用Flutter开发。

对于第一种情况，Flutter已经提供了相对完善的支持，很多常用功能已经有对应的插件可供使用，自己实现一个插件的难度也不高，只要熟悉iOS、Android原生开发，相信遵照文档都可以完成。

对于第二种情况，在已有的原生工程中逐步嵌入Flutter功能则有些复杂，因为Flutter天生就不是为这种场景而设计的，因此官方并没有提供完善的解决方案，这也是本篇文档要讨论的主题。

## 混合开发方案

由于官方没有提供完善、成熟的混合开发方案，而混合开发又不可避免，那么我们只好自己探索可行的混合开发方案了。下面的讲述以iOS为主，Android原理类似。

参考下面的基础知识部分，混合开发需要打交道的无非`FlutterEngine`和`FlutterViewController`。根据两者的组合，可能的方案有：

- 单Engine，单VC
- 单Engine，多VC
- 多Engine，多VC
- 多Engine，单VC

### 单Engine还是多Engine？

一个`FlutterEngine`负责协调一个`FlutterDartProject`实例的运行，一般的App通常只需要一个Dart工程实例，因此绝大多数情况只会使用一个`FlutterEngine`。

也有例外情况，如`ios_add2app`的例子`DualFlutterViewController`所演示的，一个`FlutterEngine`同时只能驱动一个`FlutterViewController`的运行，如果需要同时展示多个`FlutterViewController`就需要多个`FlutterEngine`。

FlutterEngine是一个重对象，对内存消耗很大，应该尽可能使用一个实例，下面的讨论也只针对单Engine的情况。

### 单VC还是多VC？

使用一个`FlutterViewController`还是使用多个呢？这个问题似乎有点儿奇怪，直觉上说，多个页面当然需要多个`FlutterViewController`了，为什么会有这个问题呢？

首先看一下[FlutterViewController.h](https://github.com/flutter/engine/commits/master/shell/platform/darwin/ios/framework/Headers/FlutterViewController.h)的源码，`initWithEngine:`方法是2018年10月27号的[版本](https://github.com/flutter/engine/commit/2bfb893cf39ef8455bf5cb87fbf3e14c410ad71a#diff-3e89a54bd23fd01730fbeabc76f26b26)才添加上去的，之前只能通过`initWithProject:`方法初始化，也就是说，在这个之前`FlutterEngine`根本不支持在多个`FlutterViewController`之前共享。每创建一个`FlutterViewController`，就会自动创建一个`FlutterEngine`。因此，包括闲鱼在内的绝大多数混合开发方案最初都是通过在多个页面之间共用一个`FlutterViewController`来实现页面切换的。

另外一个原因，正如上面刚提到的，一个`FlutterEngine`同时只能驱动一个`FlutterViewController`的运行，Flutter从一开始就不是为多VC的场景而设计的，因此即便`FlutterEngine`存在严重的内存泄漏也依然敢于发布。感兴趣的可以了解一下FlutterEngine内存泄漏的历史，由于FlutterEngine采用MRC，调用关系又错综复杂，很容易出现内存泄漏。

FlutterBoost之前的版本一直是单个`FlutterViewController`，最近的版本才更新为了多个。不过，由于原始设计问题，所以有一些蹩脚的实现。无论单个VC还是多个VC，整体的实现思路是不变的。

### 整体实现思路

重温一下业务场景：在原生App中嵌入Flutter页面，从外部视角来看，嵌入的Flutter页面与其它原生页面没有什么区别，路由跳转时不需要关心一个页面是否为Flutter实现的。

因此，整体的页面切换是原生驱动的，Flutter页面只是整个页面体系的一个子集。在进一步讲解之前，我们需要定义一下“页面”。后面讲到的Flutter页面指的都是对应于一个`UIViewController`或`Activity`的页面，在Flutter页面内部，通过`Navigator`管理的页面不属于框架管理范畴，在框架视角下，这些通过`Navigator`管理的页面都属于页面的内容。

在Dart部分，如何与原生页面做对应和关联呢？从理论上说，一个Flutter页面包含两部分：Dart部分和原生部分，Dart和原生一一对应。由于路由是原生驱动的，因此Dart端的页面结构最好与原生的一致。原生部分自然是一个ViewController或Activity、Fragement，Dart部分呢？一个Navigator的Route吗？或者就是一个Navigator？

首先来分析一下iOS、Android原生页面的结构，总的来说都可以理解为一个栈，同一时刻只有一个主内容页，即栈顶页面。对于NavigationController来说这一点很好理解，可对于TabBarController稍微有点儿绕，不过只要想一下TabBarController的实现就容易理解了，可以将viewControllers理解为层叠关系，切换Tab就是调整层叠的相对位置。

然后来看Dart端，应该用什么Widget才能更直接的对应原生的页面结构呢？一般的Flutter应用都用一个Navigator来管理页面，每个Flutter页面称为一个`Route`。这种方式能对应原生的页面结构吗？考虑一个常见的场景，在TabBarController中嵌NavigationController，当切换Tab时，Dart端的`Navigator`应该如何操作呢？先pop当前页面，然后再push？别搞笑了，这也太挫了。当然，我们可以将原生端改为在NavigationController中嵌入TabBarController来避免这种情况，但我们并不想因为Flutter而改变原生页面的结构。对于这种类似的平级页面切换的场景，`Navigator`显然无法支撑。那么Flutter有没有栈结构的Widget呢？当然，Overlay。如果Flutter端用Overlay维护页面呢？一切都自然多了，切换页面只需要改变Entry的顺序即可。由于Flutter端很多功能依赖需要依赖Navigator，因此我们可以将`Navigator`作为`OverlayEntry`。

Dart页面与原生页面的对应关系：

![flutter_page_relation](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/flutter_hybrid_page_relation.jpg)

总结一下，框架需要做的事情：协调原生端和Dart端的页面结构，使Flutter页面的原生端和Dart端能一一对应。将原生页面的生命周期同步给Dart，Dart根据接收到的原生页面生命周期事件创建、显示、销毁页面。

## 混合开发框架的实现

下面并不会特别详细的介绍源码，但会给出实现思路和思考过程，带着思路去源码里面找实现会比直接看源码要有效率的多。何况，FlutterBoost的代码质量也不高，还到处都是装逼的Boost，也没有必要细细研究。

### iOS

iOS端需要管理原生页面切换并同步到Dart端，接收Dart端的打开、关闭页面请求并同步给平台。

首先，我们需要一个对象管理`FlutterEngine`，`FlutterEngine`在整个应用中是唯一的。

然后，我们需要一个ViewController来展示Flutter页面。前面说过，有两种实现方案：

- 使用一个FlutterViewController，页面本身的ViewController作为parentViewController，页面切换时挪动`FlutterViewController`。为了避免挪动之后造成的页面空白，在挪动之前可以将当前页面截屏。对应于FlutterBoost的0.1.5之前版本。
- 每个页面就是一个FlutterViewController，不再需要截屏。对应于FlutterBoost的0.1.5及之后版本。

为了使Dart端页面能够与原生页面同步，我们需要将原生页面的生命周期事件同步给Dart。FlutterBoost之前用了装逼的xservice_kit，模板代码一大堆，初看起来一头雾水，最新版本已经改为了直接使用`MethodChannel`。

显然，我们需要与Dart端通讯，MethodChannel或者MessageChannel。

由于页面切换会改变`AppLifecyleState`的状态，因此会手动调用`inactive`、`pause`和`resume`等维护`AppLifecycleState`的状态。不过根据笔者的实践，直接使用`FlutterViewController`的情形并不需要这么做。还是那句话，FlutterBoost绝对不是精心设计和实现的代码，里面有很多不必要和多余的代码，观其大略即可。

下面列出FlutterBoost中的核心对象和职责：

- FLBFlutterEngine：管理`FlutterEngine`对象。
- FLBFlutterViewContainer：负责展示Flutter页面，通过页面标识与Dart端页面对应。
- FLBFlutterViewContainerManager：维护页面层次结构，辅助类，不需要特别关注。
- FLBFlutterApplication：一个大杂烩，职责不太清晰，糟糕的设计。
- BoostMessageChannel：负责与Dart通讯。
- FlutterBoostPlugin：负责与Dart通讯，功能同样乱七八糟，不知道放在哪儿的就放在这儿了。

### Android

与iOS一样，管理原生页面切换并同步到Dart端，接收Dart端的打开、关闭页面请求并同步给平台。

核心对象及职责：

- BoostFlutterActivity: 

### Dart

Dart端接受Native端的页面生命周期事件，并创建、展示和销毁页面。当需要打开和关闭页面时，通过Channel请求Native端来完成。

如之前的分析，Dart端的页面结构为：用Overlay维护了一个页面的栈结构，每个页面是一个`Navigator`。接收Native端页面的生命周期事件，并创建、展示和销毁页面，以保持与Native端页面结构一致。

因此，首先需要实现`Navigator`和`Route`的子类，以维护页面的基本信息，并实现自定义功能，如pop时关闭页面。然后需要一个管理`Overlay`的对象，可以添加、删除、调整页面顺序。接下来，需要一个MethodChannel或MessageChannel与Native通讯，接收Native页面的生命周期事件，以及发送打开、关闭页面的请求等。下面还需要一个对象充当Channel和Overlay管理器的中间者，以保持Channel和Overlay管理器的职责单一。这样下来，框架就出来了，只要将`Overlay`管理器对象作为Widget显示出来即可。

FlutterBoost中的核心对象及职责：

- BoostPageRoute：继承自`MaterialPageRoute`，对应一个Native页面。
- BoostContainer：继承自`Navigator`，对应一个Native页面。
- BoostContainerSettings：`BoostContainer`的基本信息，通过此信息与Native页面关联。
- BoostContainerManager：核心类，用`Overlay`维护页面的栈结构，每个`BoostContainer`是其一个`OverlayEntry`。
- ContainerCoordinator：接收Native页面的生命周期和其它事件，并调用`BoostContainerManager`的方法以同步页面。
- BoostChannel：与Native通讯的管道。
- FlutterBoost：职责比较乱，管理其它对象的生命周期，提供外部接口。

## 基础知识

要想彻底理解FlutterBoost这样的混合开发方案，我们必须有一定的知识背景，下面把关键知识点罗列一下，但不会介绍太详细，感兴趣的请看参考资料。

### Flutter是如何工作的？

不同于其它的跨平台框架，Flutter采用了一种全新的类似游戏引擎的实现方式。笼统来说，App由widget组成，它们会被渲染到Skia画布上，然后发送给平台（iOS、Android等）。平台将画布展示给用户，并发送回用户的交互事件和其它App事件。

![how_flutter_row](https://blog-image-1252287090.cos.ap-beijing.myqcloud.com/flutter/FlutterPlatformOverview.png)

App以原生的方式在平台运行，AOT编译。

在实现混合开发框架时，以iOS为例，我们至少需要了解`FlutterEngine`和`FlutterViewController`。

### FlutterEngine

`FlutterEngine`负责协调一个Dart项目的运行，在同一时刻最多只能关联一个`FlutterViewController`。`FlutterViewController`的`initWithEngine`方法会自动调用`FlutterEngine`的`setViewController:`方法，而`FlutterViewController`的`initWithProject`方法会自动创建一个关联的`FlutterEngine`。

`FlutterEngine`可以独立于`FlutterViewController`，以headless的方式运行。也就是说，不考虑性价比，我们完全可以创建一个`FlutterEngine`，在没有用户界面的情况下运行Dart项目，以实现类似JavaScriptCore的跨平台。`FlutterEngine`的生命周期可以跨越多个`FlutterViewController`，以维护状态、执行异步任务等（如下载一个大文件）。

需要调用一个新创建的`FlutterEngine`的`-runWithEntrypoint:`或`-runWithEntrypoint:libraryURI`方法以启动Dart的运行。这两个方法可以在`-setViewController:`之前被调用。

简而言之，`FlutterEngine`负责Dart的运行、channel通讯、纹理注册和插件注册等。

### FlutterViewController

可以简单理解为，负责上图中的`Canvas`和`Events`，Flutter视图的`UIViewController`实现。监听系统通知，以及在生命周期方法中，调用`FlutterEngine`的各种Channel。

### AppLifecycleState

*Flutter应用*的状态。请注意是Flutter应用的状态，而不是应用的状态，对于混合开发来说尤其如此。在开发时，不要期望能够接收到所有的通知。比如，如果用户直接把电池拔掉，应用会随操作系统一起被突然停止。

可以通过`WidgetsBinding.instance.addObserver`监听状态改变的通知。

有四种状态：

#### inactive

- 应用处于前台，对用户可见
- 应用无法接收用户输入

常见情形：

- 应用在前台时来电话了
- 进入App选择器
- 进入控制中心
- FlutterViewController正在切换中
- 正在响应TouchID请求
- 其它的Activity获取了焦点，例如出现了一个系统弹窗

#### paused

- 应用对用户不可见
- 无法响应用户输入
- 运行在后台

在此状态下，引擎不会调用`Window.onBeginFrame`和`Window.onDrawFrame`回调。

#### resumed

应用对用户可见，且可以响应用户输入

#### suspending

应用随时会被系统挂起。在此状态下，引擎不会调用`Window.onBeginFrame`和`Window.onDrawFrame`回调。

## 问题盘点

### Flutter的UI无法刷新

#### 现象描述

iOS：直接调用`FlutterViewController`的`initWithEngine:`方法创建实例，首个页面功能正常，正如官方的例子`ios_add2app`中的一样。但当再Push一个新的界面后，Flutter可以响应用户事件，但界面却不刷新了。

Android：直接继承`FlutterActivity`，打开两个页面，用户事件可以响应，但UI无法刷新。

#### 原因分析

出现UI不刷新的问题，原因可能不止一个。目前可以确定的有两个：

- 页面切换导致AppLifecycleState处于非`resumed`状态：在`FlutterViewController`和`FlutterActivity`的生命周期方法中都会通过`FlutterEngine`的`lifecycleChannel`改变`AppLifecycleState`的状态。主要表现在Android平台上，从页面A切换到页面B，页面B的`onResume`方法先于页面A的`onStop`方法被调用，导致页面切换完成之后`paused`状态。具体可以看[FlutterActivityDelegate.java](https://github.com/flutter/engine/blob/master/shell/platform/android/io/flutter/app/FlutterActivityDelegate.java)以及[FlutterViewController.mm](https://github.com/flutter/engine/blob/master/shell/platform/darwin/ios/framework/Source/FlutterViewController.mm)的源码。
- 页面切换导致`displayingFlutterUI`为NO：在iOS上，页面的生命周期方法不会导致`AppLifecycleState`状态混乱，但是连续的`FlutterViewController`切换会导致`displayingFlutterUI`混乱。在`viewWillAppear:`方法中将其置为YES，在`viewDidDisappear:`方法中将其置为NO。因此，`FlutterViewController`与普通ViewController之间的切换并不会出问题，但当连续的`FlutterViewController`之间切换时，由于多个VC共享同一个`FlutterEngine`，当`viewWillAppear:`方法被调用时，可能与`FlutterEngine`处于解绑状态。

### 再次创建FlutterViewController闪退

#### 现象描述

先通过`initWithEngine:`方法创建一个实例，返回之后再创建一个，在调用`FlutterEngine`的`setViewController:`方法时闪退。

#### 解决方法

参考`ios_add2app`：

```objective-c
- (void)viewWillDisappear:(BOOL)animated {
    [super viewWillDisappear:animated];
    
    if (self.isMovingFromParentViewController) {
        // Note that if we were doing things that might cause the VC
        // to disappear (like using the image_picker plugin)
        // we shouldn't do this.  But in this case we know we're
        // just going back to the navigation controller.
        // If we needed Flutter to tell us when we could actually go away,
        // we'd need to communicate over a method channel with it.
        [self.engine setViewController:nil];
    }
}
```

### FlutterViewController.dealloc闪退

#### 现象描述

通过`initWithProject:`方法创建，dealloc时闪退，见[Github #37225](https://github.com/flutter/flutter/issues/37225)。

#### 解决方法

临时解决方法：

```objective-c
- (void)dealloc {
    [EAGLContext setCurrentContext:nil];
    ...
}
```

## 参考资料

- [码上用它开始Flutter混合开发——FlutterBoost](https://www.yuque.com/xytech/flutter/hhnyho)
- [How Flutter Works](https://buildflutter.com/how-flutter-works/)
