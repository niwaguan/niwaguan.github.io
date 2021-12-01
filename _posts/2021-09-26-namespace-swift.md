---
layout: post
title: 如何在Swift中轻松扩展现有类而不需要考虑冲突
category:
  - iOS
tags:
  - Swift
  - RxSwift
---

不知各位同学是否有感觉，类似：`RxSwift`中的`xxx.rx.xxx`以及`Kingfisher`中的`image.kf.xxx`这种`api`使用起来就很爽。那么这种类似命名空间的东西是怎么实现的呢？今天一起来扒扒。我们的目标是实现字符串的截取，可以像下面这样调用：

```swift
let string = "Hello, World!"
string.ns.substring(from: 1) // ello, World!
```

## 啥也不管，就这样来行不行？

`string.ns.substring(from: 1)`从表面看，就是调用了`String`的一个属性，然后得到一个对象，这个对象上面有一个方法`substring(from:)`。所以，只管梭哈：

```swift
class StringApi {
    /// 由于后续需要操作String，这里通过构造函数将其传递进来
    let string: String
    init(_ string: String) {
        self.string = string
    }

    func substring(from: Int) -> String? {
        let start = string.index(string.startIndex, offsetBy: from)
        let x = string[start..<string.endIndex]
        return String(x)
    }
}

extension String {
    /// 通过ns将支持的Api都返回
    var ns: StringApi {
        return StringApi(self)
    }
}

```

nice！很简单的嘛。但是，我们要想扩展其他类，怎么办？比如扩展`CGSize`。是不是要向下面这样：

```swift
class CGSizeApi {
    let size: CGSize
    init(_ size: CGSize) {
        self.size = size
    }
    func rotate() -> CGSize {
        return CGSize(width: size.height, height: size.width)
    }
}

extension CGSize {
    var ns: CGSizeApi {
        return CGSizeApi(self)
    }
}
```

看，还是很简单嘛！好像，只要我们想扩展某种类型，只需定义`api`所依赖的对象，然后再通过`extension`将其返回就可以了。真的是这样吗？让我们试试扩展`Array`：
![-w799](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/09/27/16327103515773.jpg)

可以看到当我们需要操作具体类型的时候，就无法搞定了。当然你可能说使用类型约束，可以搞定。但是这种方式还是有太多的弊端：大量的重复代码、不方便扩展，每次新扩展类型，都需要写对应的`api`包装类、不易维护。

## 新思路

观察上面的`StringApi`、`CGSizeApi`和`ArrayApi`他们本质上就是对后续想操作的数据的包装。所以我们实现一个包装类：

```swift

class Wrapper<T> {
    let value: T
    init(_ value: T) {
        self.value = value
    }
}

```

这样我们就可以包装任意值。

下一步就是通过`.pns`（为了和之前的`ns`区别）返回这个`Wrapper`对象了。这次我们选择`protocol`：

```swift
protocol NamespaceCompatible {
    associatedtype Value
    var pns: Wrapper<Value> { get set }
}
```

在提供下默认实现：

```swift
extension NamespaceCompatible {
    var pns: Wrapper<Self> {
        get { return Wrapper(self) }
        set { }
    }
}
```

这样，我们就可以很方便的扩展其他类型了。比如`String`:

```swift
extension String: NamespaceCompatible {}
```

再比如`CGSize`

```swift
extension CGSize: NamespaceCompatible {}
```

最后，我们的`api`要放在何处？又该如何组织？答案还是`extension`！

> 我们通过`.pns`返回的是`Wrapper`，所以这里应该扩展`Wrapper`。但是需要加上约束。

```swift
/// String
extension Wrapper where T == String {
    func substring(from: Int) -> String? {
        let start = value.index(value.startIndex, offsetBy: from)
        let x = value[start..<value.endIndex]
        return String(x)
    }
}
/// CGSize
extension Wrapper where T == CGSize {
    func rotate() -> CGSize {
        return CGSize(width: value.height, height: value.width)
    }
}
/// Sequence
extension Wrapper where T: Sequence, T.Element == String {
    func concat() -> String {
        return value.reduce("") { result, element in
            return result + element
        }
    }
}
```

好了，这样就可以快乐的玩耍了。下次我们再扩展其他类型时，只需：

1. `extension Type: NamespaceCompatible {}`
2. `extension Wrapper where T == Type` 或者 `extension Wrapper where T: Type`

## RxSwift & Kingfisher

最后我们一起来看看第三方库的实现方式，是否和我们的思路一样：

### RxSwift

![-w902](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/09/27/16327143891877.jpg)

整个实现都在`Reactive.swift`文件中：

1. `public struct Reactive<Base> {}`相当于我们这里的`class Wrapper<T> {}`。选择`struct`而不是`class`是一个优化的点。额外的`subscript`是`Rx`的内容了，这里不再分析。
2. `ReactiveCompatible`相当于我们这里的`NamespaceCompatible`。这里多了通过类型访问`.rx`的支持。可以按需加入。

### Kingfisher

该框架的实现在`Kingfisher.swift`文件中：
![-w598](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/09/27/16327170993430.jpg)

1. `KingfisherWrapper`跟我们的`Wrapper`相同
2. 我们的`NamespaceCompatible`协议在这里被分成`KingfisherCompatible`和`KingfisherCompatibleValue`。前者专门负责引用类型，后者负责值类型。不知这样区分有何考究？
3. 这里的`KingfisherCompatible`和`KingfisherCompatibleValue`都是空协议。这里将`Wrapper`的泛型规定成了遵循协议的那个类型。 而`R小Swift`中在协议中明确规定了`ReactiveBase`，更加灵活。

## 我还是不理解。怎么办？

在如何实现这里，需要一定的抽象能力。需要多看多思考多动手。

还有一部分可能是因为 Swift 的语法问题：

1. `Self` & `self`的使用方法。 参考[这里](https://www.cnswift.org/types#Self)
2. `Swift`的协议约束不熟悉。参考[这里](https://www.cnswift.org/protocols#spl-22)

好了，再也不担心我的扩展和别人的扩展冲突了！
