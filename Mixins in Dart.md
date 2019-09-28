# Mixins in Dart

[官方文档](https://dart.dev/guides/language/language-tour#classes)如此描述Dart语言：

> Dart is an object-oriented language with classes and mixin-based inheritance. Every object is an instance of a class, and all classes descend from [Object.](https://api.dart.dev/stable/dart-core/Object-class.html) *Mixin-based inheritance* means that although every class (except for Object) has exactly one superclass, a class body can be reused in multiple class hierarchies.

与Java、OC等App开发者常用的面向对象编程语言一样，Dart是单继承的，除了根类，每个类有且只有一个父类；不同的是，Dart的继承是基于`Mixin`的，一个类的代码可以在多个类层次结构中被重用。

对于大多数iOS、Android开发者来说，Mixin是一个陌生的概念。由于其功能强大，Flutter库中大量使用了Mixin，因此有必要研究一番。

## Mixin是什么？

简单来说，`Mixin`是在多个类层次结构中重用代码的一种方式。

> 在面向对象编程语言中，mixin是一个类，它的代码供其它类使用，但无需成为其它类的父类。其它类如何获得mixin方法的使用权取决于具体的编程语言。Mixin有时被描述为”包含的“而不是”继承的“。
>
> Mixin鼓励代码重用，可用于解决多继承的[菱形问题](https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem)，或解决语言中缺乏对多继承支持的问题。Mixin也可以被看作带有默认实现方法的接口。
>
> — 维基百科

在Dart语言中，通过`with`关键字加一个或多个mixin名字的方式使用mixin：

```dart
class Musician extends Performer with Musical {
  // ···
}

class Maestro extends Person
    with Musical, Aggressive, Demented {
  Maestro(String maestroName) {
    name = maestroName;
    canConduct = true;
  }
}
```

定义mixin的方式跟类一样，只不过没有构造方法。除非希望mixin可以像普通类一样使用，否则使用`mixin`关键字而不是`class`。例如：

```dart
mixin Musical {
  bool canPlayPiano = false;
  bool canCompose = false;
  bool canConduct = false;

  void entertainMe() {
    if (canPlayPiano) {
      print('Playing piano');
    } else if (canConduct) {
      print('Waving hands');
    } else {
      print('Humming to self');
    }
  }
}
```

通过`on`关键字可以限定mixin允许被哪些类使用，这样mixin中可以调用非自身定义的方法。

```dart
mixin MusicalPerformer on Musician {
  // ···
}
```

其实，在Swift等现代编程语言中，也有类似Mixin的特性，我们可以使用协议扩展的方式在Swift中实现Mixin效果：

```swift
protocol ErrorDisplayable {
    func error(message:String)
}

extension ErrorDisplayable {
    func error(message:String) {
        // Do what it needs to show an error
        //...
        print(message)
    }
}

struct NetworkManager : ErrorDisplayable {
    func onError() {
        error("Please check your internet Connection.")
    }
}
```

## 历史

Mixin最初出现在Symbolics的面向对象风味系统，该系统是[Lisp Machine Lisp](https://en.wikipedia.org/wiki/Lisp_Machine_Lisp)语言使用的面向对象方法。其名字的灵感来源于冰淇淋：冰淇淋店提供基本口味的冰淇淋（香草、巧克力等），然后混合上其它物品（坚果、饼干、巧克力等），称为"mix-in"。

## 优势

1. 提供了一种多继承机制，允许多个类使用公共功能，但是没有复杂的多继承语义
2. 代码重用性
3. 只继承或使用父类的必要特性，而不是所有特性

### 多重继承的问题

假设有如下抽象类：

```dart
abstract class Performer {
   void perform();
}
```

两个子类Dancer和Singer：

```dart
class Dancer extends Performer {
   void perform() {
      print('Dance Dance Dance ');
   }
}
class Singer extends Performer {
   void perform() {
      print('lalaaa..laaalaaa....laaaaa');
   }
}
```

**假设Dart支持多重继承，有一个Musician类继承了Dancer和Singer。**

> Note: Dart只支持单继承，此处仅是假设，在C++这种支持多重继承的语言中会出现此种情形

```dart
class Musician extends Dancer,Singer {
   void showTime() {
      perform();  
   }
}
```

当我们调用Musician类的`perform()`方法时，具体的哪个方法会被执行：

![deadly_diamond_of_death](https://miro.medium.com/max/1400/1*k28mWAKSCZN-_YWyFrOX6w.jpeg)

困惑！这就是著名的菱形问题（DDD），多重继承的一个核心问题。

### Mixin如何解决菱形问题？

将上面的代码用mixin重写：

```dart
class Performer {
   void perform() {
     print('performing...');
   }
}

mixin Dancer {
   void perform() {
     print('Dance...Dance...Dance..');
   }
}

mixin Singer {
   void perform() {
     print('lalaaa..laaalaaa....laaaaa');
   }
}

class Musician extends Performer with Dancer,Singer {
   void showTime() {
     perform();
   }
}
```

多个Mixin之间是层叠的，而不是平行的，因此不会出现冲突。

为了确定使用mixin时具体的调用方法，可以遵循如下简单的步骤：

- 如果使用mixin的类继承于某个类，首先将这个类放在栈顶。拿`class Musician extends Performer`来说：

![performer](https://miro.medium.com/max/646/1*rUrxDkdla3cBsquM64J2wQ.jpeg)

- 牢记声明mixin的顺序，它绝对了类的优先级。如果多个mixin包含相同的方法，在后面的会被优先执行。对应上面的代码，首先有：

![performer_dancer](https://miro.medium.com/max/656/1*n2xlbk329Hlv29B5Y5CMWw.jpeg)

然后：

![performer_dancer_singer](https://miro.medium.com/max/650/1*WbOzfx7GcUnhebymJZPLEg.jpeg)

- 最后，加上类本身。如果类中定义了父类或mixin中相同的方法，本类中的方法首先被执行。

![mixin_which_method](https://miro.medium.com/max/1400/1*lxRghguALEkvYDNmc0Ykgw.jpeg)

### 只继承或使用父类的必要特性，而不是所有特性

这一点也是面向协议编程的优势，需要用心理解。我们拿Mac开发中的例子来说明，只要关注继承层次结构就好，不涉及具体开发知识。

在OC中，button有如下继承层次结构：

```objective-c
protocol NSObjectProtocol { /* ... */ }
open class NSObject: NSObjectProtocol { /* ... */ }
open class NSResponder: NSObject { /* ... */ }
open class NSView: NSResponder { /* ... */ }
open class NSControl: NSView { /* ... */ }
open class NSButton: NSControl { /* ... */ }
```

`NSButton`与`NSView`是典型的继承：在使用`NSView`的地方都可以使用`NSButton`，`NSButton`共享`NSView`的所有数据和行为。

然而上面的继承结构中，可不仅仅是`NSButton`和`NSView`。

`NSObject`类是`NSObjectProtocol`协议的默认实现，如果协议支持默认实现的话，我们完全可以去掉`NSObject`这个类，我们并不需要实例化`NSObject`类来实现什么具体功能。

`NSView`和`NSResponder`完全是正交的概念 - 显示和事件。因此，`NSView`是一个`NSResponder`并不是那么明显。更重要的是，从理论上说，*任何*对象只要实现相应的接口并插入到响应链中，就应该可以执行响应链的逻辑。

`NSControl`更难区分好坏，它提供了一个交互和数据行为混合包，并扩展了`NSResponder`的一些行为。在某些方面，继承`NSControl`的确有助于代码重用 - 因此有一种观点认为它的设计是合适的。不过，它是继承层次结构中的一个抽象类（不需要直接实例化），这意味着它作为协议可能会更好 - 可能需要拆分为更细小的协议。

而在Swift中，之前OC中的很多类都可以用协议+默认实现来替代。由于协议不是单继承的，因此可以更灵活地组合。`NSButton`的继承结构可以这样实现（粗糙版）：

```swift
protocol NSObjectProtocol { /* ... */ }
protocol NSResponder { /* ... */ }
protocol NSControl { /* ... */ }
protocol NSTwoStateControl: NSControl { /* ... */ }
open class NSView: NSObjectProtocol, NSResponder { /* ... */ }
open class NSButton: NSView, NSTwoStateControl { /* ... */ }
```

`NSObject`被拿掉了（通过`NSObjectProtocol`的默认实现替代），`NSResponder`和`NSControl`变成了协议，同时`NSControl`被拆分为一些更简单的协议（更大的功能通过混合可以很容易地实现）。

概况来说，协议/mixin更灵活、更易于组合。

那么，什么时候应该用继承呢？由于mixin/协议的灵活性和强大功能，我们应该将继承限定在：

1. 想要共享基类的*绝大多数*行为，并且，
2. 希望合并基类的*所有*数据

## 参考资源

- [Dart for Flutter : Mixins in Dart](https://medium.com/flutter-community/https-medium-com-shubhamhackzz-dart-for-flutter-mixins-in-dart-f8bb10a3d341)
- [Protocol vs Subclass](https://www.cocoawithlove.com/blog/protocols-versus-subclasses.html)