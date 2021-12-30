# Encodable

> 关于系统框架部分的代码可以参考[这里](https://github.com/apple/swift-corelibs-foundation/blob/main/Sources/Foundation/JSONEncoder.swift)

## Encoder协议

`Encoder`就是记录当前`encode`的状态，比如当前解析的路径，提供存储值的容器。定义如下：

```swift
public protocol Encoder {
    /// 当前的编码路径，如：videos ->[0] -> id
    var codingPath: [CodingKey] { get }
    /// 存储自定义信息
    var userInfo: [CodingUserInfoKey : Any] { get }
    /// 三种容器，下面会介绍
    func container<Key>(keyedBy type: Key.Type) -> KeyedEncodingContainer<Key> where Key : CodingKey
    func unkeyedContainer() -> UnkeyedEncodingContainer
    func singleValueContainer() -> SingleValueEncodingContainer
}
```
`EncodingContainer`可以理解为数据的存储器，任何满足`Encodable`的数据都可以存在其中，它主要有3种：

1. `KeyedEncodingContainer`：负责`Key-Value`结构。
2. `UnkeyedEncodingContainer`：负责`数组`结构。
3. `SingleValueEncodingContainer`：负责单一值结构，如`Int`、`String`...

一个`EncodingContainer`的内容大致分为几类：
1. 上下文信息
2. 具体类型的编码支持方法
3. 嵌套容器支持方法
4. `superDecoder`

#### KeyedEncodingContainer

var codingPath: [CodingKey] { get }


```swift
public mutating func encodeConditional<T>(_ object: T, forKey key: KeyedEncodingContainer<K>.Key) throws where T : AnyObject, T : Encodable

public mutating func encodeConditional<T>(_ object: T, forKey key: K) throws where T : AnyObject, T : Encodable
```