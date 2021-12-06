---
layout: post
title: Alamofire - 你真的会传递请求参数吗？
category:
  - iOS
tags:
  - Swift
  - Alamofire
---

今天一起来研究下`Alamofire`中请求参数相关内容。我们最熟悉的应该是使用字典来传递参数，向下面这样：

```swift
let parameters: [String: Any] = ["a": 1, "b": true]
AF.request("https://httpbin.org/get", parameters: parameters)
```

其实，我们还可以通过模型的方式传递。但前提是该模型遵循`Encodable`协议，如下：

```swift
let parameters = struct { a: 1, "b": true }
AF.request("https://httpbin.org/get", parameters: parameters)
```

两种方式，对于调用者来说，基本上是无感的。下面就一起走进其背后的故事。

## Alamofire 支持的参数格式

### 字典类型

支持使用`字典类型`创建请求的方法如下：
![-w773](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/12/06/16384950344120.jpg)

> 1、2 是今天需要研究的；3 是拦截器，具体参考[这篇文章]({{ site.url }}/2021/11/26/alamofire-request-interceptor/)。

其实这里的`Parameters`就是字典的别名：

```swift
public typealias Parameters = [String: Any]
```

该类型的编码器是`ParameterEncoding`。它负责将`Parameters`类型的参数合适的放入请求中。`ParameterEncoding`是个协议：

```swift
/// 编码器，将参数编码至URLRequest中。
/// 注意：参数是字典类型。
public protocol ParameterEncoding {
    func encode(_ urlRequest: URLRequestConvertible, with parameters: Parameters?) throws -> URLRequest
}
```

该协议的要求就一个方法，它接收`URLRequestConvertible`和`字典参数`，需要返回`URLRequest`。该方法的职能很明确：负责使用传入的`字典参数`加工`URLRequest`并返回加工结果。能满足这个功能的都可以叫做`ParameterEncoding`。

> `URLRequestConvertible`代表了`可转换成URLRequest的类型`，可以从该类型中获取`URLRequest`。

我们可以传递`字典类型`参数，正是因为有了该协议的直接支持。该部分的类图如下：![encoding-w2235](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/12/06/encoding.png)

对于`URLEncoding`：

1. `destination`控制将参数放在请求的什么位置。默认是根据请求方法自动判断，`[.get, .head, .delete]`是将参数放在 URL 的 query 中，其他方法放在 httpBody 中。
2. 参数中有数组类型的话，如何编码通过`arrayEncoding`属性控制；同时`boolEncoding`属性是控制如何编码`Bool`类型的。

对于`JSONEncoding`，它只会将参数放在请求的 Body 中。你要问为啥？那就是`JSONEncoding`编码后的参数会变成`Data`，只能通过 body 传输，并设置`Content-Type`。

### 可编码类型

支持使用`可编码类型`创建请求的方法如下：
![-w871](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/12/06/16384981336549.jpg)

其实这里的`Parameters`是一个泛型。该类型参数对应的编码器为`ParameterEncoder`，同样它也是协议：

```swift
/// 编码器，将参数编码至URLRequest中。
/// 注意：该方法为泛型方法，参数要求为可编码类型。
public protocol ParameterEncoder {
    func encode<Parameters: Encodable>(_ parameters: Parameters?, into request: URLRequest) throws -> URLRequest
}
```

`Alamofire`也默认实现了两种编码器。如下：![encoder](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/12/06/encoder.png)

这里的设计和上一部分类似，不做细讲。

### 更进一步

对于两种参数格式，他们的处理流程大致如下：
![why-encoding-encoder](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/12/06/whyencodingencoder.png)

我们以两种格式的参数为起点，以及两种格式的输出为终点，可以看到：

1. 对于`JSON`类型的输出，由于系统内建的`JSONEncoder`原本就是为了编码该类型而生，所以对于两种类型的输入，后面的支持者都是它。
2. 对于`key=vale`格式的输出，系统没有此种功能的支持。所以这部分的实现都是由框架提供。具体的：
   1. 字典类型的输入：直接在`URLEncoding`类中实现。
   2. 可编码类型的输入：背后的支持者为`URLEncodedFormEncoder`，由它实现具体的编码过程。

### 总结

今天我们了解到了`Alamofire`中处理请求参数背后的故事。总结来说，就是支持`Encodable`类型的参数。说到这里，其实字典也是支持`Encodable`的，为什么这种类型单独领出来实现？(欢迎评论区讨论)
