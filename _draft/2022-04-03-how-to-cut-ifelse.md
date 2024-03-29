写代码的程序员很多，写好代码的程序员却不是那么多（自认为在写好代码的路上🤣）。今天分享一个避免大量`if-else`的案例，和大家共同进步。

> 该篇是笔者在`Flutter`项目中遇到的问题，所以示例代码是`Dart`语言。请读者不要有负担，语言不是重点，重点是思想。

## 问题的由来

下面是两段实际的业务代码：

1. `购买`操作，再执行真正的购买流程之前，需要有必要的条件校验
    ```dart
    _onBuyButtonClick() {
      /// 1. 用户封禁校验
      final user = getUserInfo();
      if (user.isForbidden) {
        showForbiddenDialog();
        return;
      }
      /// 2. 未支付订单数量校验
      final orders = getUserWaitingPayOrders();
      if (orders.length >= limit) {
        showTooMuchOrderWaitingPayDialog();
        return;
      }
      /// 3. xxx
      /// 4. xxx
    
      /// 购买流程
    }
    ```

2. `出售`操作，再执行真正的出售流程之前，需要有必要的条件校验
    ```dart
    _onSellButtonClick() {
      /// 1. 用户封禁校验
      final user = getUserInfo();
      if (user.isForbidden) {
        showForbiddenDialog();
        return;
      }
      /// 2. 店铺封禁校验
      /// 3. xxx
      /// 4. xxx
      
      /// 售卖流程
    }
    ```

这样需要校验的流程，我们一共有`10个`！每个流程需要校验的项目`2~5`个不等。大家体验下需求文档：
![-w1825](media/16493102770149.jpg)
![-w1218](media/16493103032126.jpg)


这里的每一张图代表一个流程，每个黄色方块是一个校验项目。当然，流程是不可能重复的，但每个校验项可能是重复的。所以我们可以将问题抽象为：如何在`N`个操作加入`M`个前置验证？例如：

1. 操作`N1（购买）`需要检查`M1（用户是否被封禁）、M2（等待付款的订单不能太多）...`
2. 操作`N2（上架出售）`需要检查`M1、M3（店铺是否被封禁）、M4（正在售卖的商品是否达到数量上限）...`
3. 操作`N3`需要验证`M5、M6、M8`....

对于这种`非常合理的需求`，我们怎么能反驳呢？😁 So, let's kill it!

## 解决方案

`Show me the code`。先给出目前正在使用的方案，再分析里面的具体细节。

我们最终实现的效果如下（以`购买流程`为例）：
```dart
_onBuyButtonClick() {
  /// 使用CheckController来控制哪些条件是需要被检查的
  final anyChecker = CheckController().check([
    Requirement.account,
    Requirement.orderBuffer,
  ]);
  /// 若存在需要处理的，就处理它
  if (anyChecker != null) {
    anyChecker.handle();
    return;
  }
  /// 之前的购买流程
}
```

可以看到，我们将原来的几十行的校验代码（最长的8个校验项目，也就是8个if判断），缩短为短短的几行。相比之下，该方案有很多优点：
1. 没有重复代码。之前`N2`流程中的校验代码，完全是`N1`的`Copy`。现在即使两个流程拥有相同的校验项，也只体现在枚举的相同`case`上。
2. 可读性增强，可维护性大大提高。在大量的`if-else`中搞懂它是做什么的，虽然不是很有挑战，但它确实需要一定时间。特别是在一段时间之后，加上没有详细注释的情况下。
3. 可维护性大大提高。一个流程的校验项，完全对应数组的元素，包括校验项的增删改查。假设在一个流程上改变两个项目的优先级，之前你需要读懂哪两个`if`是你关心的，然后才能调整。现在，你只需要在数组中找到对应的`case`就可以。并且现在它们是绝对聚集的，之前的代码可能一部分在屏幕可见范围，另一部分完全不在！

## 如何实现

如果你对上面的实现感兴趣的话，这里我们一起分析它是如何实现的。

### 第一阶段 - 减少重复性

若想复用`M`个检查，我们必须将检查部分的代码独立出来。以`购买`的检查为例，我们可以发现整个过程可以分为两步：
1. 条件校验
2. 结果处理
所有`M`个检查都可以看做，校验`xxx`条件，不满足的话就`xxx`。这里我们将每个检查封装成独立的类，以`购买`中的`用户是否被封禁`检查为例：

```dart
class AccountForbiddenChecker {
  /// 根据条件返回用户是否被封禁
  bool match() {
    return false;
  }
  /// 用户被封禁的具体操作，如弹窗警告
  void handle() {}
}
```

再比如`等待付款的订单不能太多`的检查：

```dart
class OrderWaitingPayBufferChecker {
  /// 判断用户未支付的订单是否太多
  bool match() {
    return false;
  }

  /// 未支付订单过多的具体操作，如弹窗警告
  void handle() {}
}
```

像这样，我们可以将这`M`个检查，都封装在具体的类中。避免了多处流程条件检查中的复制粘贴。但对于使用者来说，他需要记住每一种`Checker`的名字，最起码需要有印象，这是一种负担。所以，我们使用枚举，来表示每一个检查项：

```dart
/// 需要校验的项目
enum Requirement {
  account,
  orderBuffer,
  // ...
}
```

由枚举到具体的检查类，我们还需要有个转换过程。这里使用了`switch`：

```dart
extension Mapper on Requirement {
    RequirementChecker toChecker() {
        switch(this) {
            case Requirement.account: return AccountForbiddenChecker();
            case Requirement.orderBuffer: return OrderWaitingPayBufferChecker();
            // ...
        }
    }
}
```

### 第二阶段 - 增加可复制性

当需要协调多个类的时候，我们就需要一个管理者了。

```dart
/// 检查项管理器
class CheckController {
  /// 根据传入的枚举，判断具体的项目是否匹配，若匹配，则返回对应的检查者。
  RequirementChecker? check(List<Requirement> items) {
    for (final item in items) {
      final checker = item.toChecker();
      if (checker.match()) {
        return checker;
      }
    }
    return null;
  }
}
```
`RequirementChecker`也是必要的，它是一个接口，负责标准化每个`Checker`：

```dart
abstract class RequirementChecker {
  bool match();
  void handle();
}
```

然后每个具体的`Checker`实现该接口，这样管理者，以及外部才能统一使用多个`Checker`。

到这里，我们就实现了上面的解决方案。对于每个流程，我们只需要`CV`大法，然后对校验项稍作修改，即可达到效果。`bingo`！

## 相关推荐

今天的解决方案并不是笔者初创🤣。其思想来源于`设计模式中的责任链模式`。[墙裂推荐这个网站](https://refactoringguru.cn/design-patterns/catalog)。

对于每种模式，都配有大量的图解、问题以及解决方案。例如`责任链模式`一章：
![-w670](media/16497292902793.jpg)
![-w681](media/16497293054966.jpg)
![-w673](media/16497293518452.jpg)
简直不要太赞！

好了，秘籍都奉上了。希望大家早日登峰造极！
