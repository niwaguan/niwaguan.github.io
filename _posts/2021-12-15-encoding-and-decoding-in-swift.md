---
layout: post
title: 【译】Swift&JSON 从入门到精通
description: 在这个教程中，你将学习到所有使用Swift进行编解码所需要的知识。包括这些： 1. 在`蛇形命名`和`驼峰命名`格式之间转换 2. 自定义`Coding keys` 3. 使用`keyed`, `unkeyed` 和 `nested` 容器 4. 处理`嵌套类型`, `日期类型`以及子类
category:
  - iOS
tags:
  - Swift
  - JSON
  - Codable
---

> 原文链接：https://www.raywenderlich.com/3418439-encoding-and-decoding-in-swift

在iOS中最常见的工作是将数据保存起来并通过网络传输。但是在这之前，你需要将数据通过`编码`或`序列化`转换成合适的格式。
![encode](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/12/23/encode.png)

同样的，在你使用这些数据之前，你也需要将其转换成合适的格式。这个相反的过程被称为`解码`或`反序列化`。![decode](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/12/23/decode.png)
在这个教程中，你将学习到所有使用Swift进行编解码所需要的知识。包括这些：
1. 在`蛇形命名`和`驼峰命名`格式之间转换
2. 自定义`Coding keys`
3. 使用`keyed`, `unkeyed` 和 `nested` 容器
4. 处理`嵌套类型`, `日期类型`以及子类

这确实有点多，是时候开始动手了！

## 开始动手

从[这里下载](https://pan.baidu.com/s/1tdmeSNNJ7BjcU3hMC_6F6w)所需资源后继续（先复制提取码`lgnb`）。

> 下载完成后，starter是该教程使用的版本。final是最终完成的版本。

我们打开本节代码`Nested types`。使`Toy`和`Employee`遵循`Codable`协议：

```swift
struct Toy: Codable {
  ...
}
struct Employee: Codable {
  ...
}
```

`Codable`本身并不是一个协议，它只是另外两个协议的别名：`Encodable`和`Decodable`。你也行已经猜到了，这两个协议就是代表那些可以被编解码的类型。

你无需再做其他事情，因为`Toy`和`Employee`的所有存储属性都是`Codable`的。Swift标准库中大多数类型（例如`String`、`URL`）都是支持`Codable`的。

添加一个`JSONEncoder`和`JSONDecoder`来处理`toys`和`employees`的编解码：

```swift
let encoder = JSONEncoder()
let decoder = JSONDecoder()
```
操作JSON我们只需做这些！下面进入第一个挑战！

## 编解码嵌套类型

`Employee`包含了一个`Toy`属性（这是个嵌套类型）。编码后的JSON结构和`Employee`结构体保持一致：

```json
{
  "name" : "John Appleseed",
  "id" : 7,
  "favoriteToy" : {
    "name" : "Teddy Bear"
  }
}
```
```swift
public struct Employee: Codable {
  var name: String
  var id: Int
  var favoriteToy: Toy
}
```

JSON数据将`name`嵌套在`favoriteToy`之中，并且所有的JSON字段名与`Toy`和`Employee`的存储属性名相同，所以基于结构体的类型体系，JSON的结构很容易理解。如果属性名称和JSON的字段名都相同，并且属性都是`Codable`的，那么我们可以很容易的将JSON转换为数据模型，或者反过来。现在来试一试：

```swift
// 1
let data = try encoder.encode(employee)
// 2
let string = String(data: data, encoding: .utf8)!
```

这里做了2件事：
1. 将`employee`使用`encode(_:)`编码成JSON。是不是很简单！
2. 从上一步的`data`中创建String，一遍可以查看其内容。

这里的编码过程会产生合法的数据，所以我们可以使用它重新创建`employee`：

```swift
let sameEmployee = try decoder.decode(Employee.self, from: data)
```

好了，可以开始下一个挑战了！

## 在`蛇形命名`和`驼峰命名`格式之间转换

现在，假设JSON的键名从驼峰格式（这样`looksLikeThis`）转换成了蛇形格式（这样`looks_like_this_instead`）。但是，`Toy`和`Employee`的存储属性只能使用驼峰格式。幸运的是`Foundation`考虑到了这种情况。

打开本节代码`Snake case vs camel case`，在编解码器创建之后使用之前的位置添加下面的代码：

```swift
encoder.keyEncodingStrategy = .convertToSnakeCase
decoder.keyDecodingStrategy = .convertFromSnakeCase
```
运行代码，检查`snakeString`，编码后的`employee`产生下面的内容：

```json
{
  "name" : "John Appleseed",
  "id" : 7,
  "favorite_toy" : {
    "name" : "Teddy Bear"
  }
} 
```
![announcement](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/12/23/announcement.png)
## 自定义Coding keys

现在，假设JOSN的格式再一次改变，其使用的字段名和`Toy`和`Employee`中存储属性名不一致了：

```json
{
  "name" : "John Appleseed",
  "id" : 7,
  "gift" : {
    "name" : "Teddy Bear"
  }
}
```
可以看到，这里使用`gift`代替了原来的`favoriteToy`。这种情况我们需要自定义`Coding keys`。在我们的类型中添加一个特殊的枚举类型。打开本节代码`Custom coding keys`，在`Employee`中添加下面的代码：

```swift
enum CodingKeys: String, CodingKey {
  case name, id, favoriteToy = "gift"
}
```
这个特殊的枚举遵循了`CodingKey`协议，并使用`String`类型的原始值。在这里我们可以让`favoriteToy`和`gift`一一对应起来。

在编解码过程中，只会操作出现在枚举中的`cases`，所以即使那些不需要指定一一对应的属性，也需要在枚举中包含，就像这里的`name`和`id`。

运行`playground`，然后查看`string`的值，你会发现JSON字段名不在依赖存储属性名称，这得益于`自定义的Coding keys`。

继续下一个挑战！

## 处理`扁平化`的JSON

现在，JSON的格式变成下面这样：

```json
{
  "name" : "John Appleseed",
  "id" : 7,
  "gift" : "Teddy Bear"
}
```

这里不在有`嵌套`结构，和我们的模型结构不一致了。这种情况我们需要自定义编解码过程。

打开本节代码`Keyed containers`。这里有个`Employee`类型，它遵循了`Encodable`。同时我们使用`extension`让它遵循了`Decodable`。这样做的好处是，可以保留结构体的`逐一成员构造器`。如果我们在定义`Employee`时让它遵循`Decodable`，它将失去这个构造器。添加下面的代码到`Employee`中：

```swift
// 1
enum CodingKeys: CodingKey {
  case name, id, gift
}

func encode(to encoder: Encoder) throws {
  // 2
  var container = encoder.container(keyedBy: CodingKeys.self)
  // 3  
  try container.encode(name, forKey: .name)
  try container.encode(id, forKey: .id)
  // 4
  try container.encode(favoriteToy.name, forKey: .gift)
}
```
在之前简单(指属性名和键名一一对应且嵌套层级相同)的示例中，`encode(to:)`方法由编译器自动实现了。现在我们需要手动实现。

1. 创建`CodingKeys`表示JSON的字段。因为我们没有做任何的关系映射，所以不必声明它的原始类型为`String`。
2. 从`encoder`中获取`KeyedEncodingContainer`容器。这就像一个字典，我们可以存储属性的值到其中，这样就进行了编码。
3. 编码`name`和`id`属性到容器中。
4. 使用`gift`键，直接将`toy`的名字编码到容器中。

运行`playground`，然后查看`string`的值，你会发现它符合上面JSON的格式。我们可以选择使用什么字段名编码一个属性值，这给了我们很大的灵活性。

和编码过程类似，简单版本的`init(from:)`方法可以由编译器自动实现。但是这里我们需要手动实现，使用下面的代码替换`fatalError("To do")`：

```swift
// 1
let container = try decoder.container(keyedBy: CodingKeys.self)
// 2
name = try container.decode(String.self, forKey: .name)
id = try container.decode(Int.self, forKey: .id)
// 3
let gift = try container.decode(String.self, forKey: .gift)
favoriteToy = Toy(name: gift)
```
然后添加下面的代码，就可以从JSON中重新创建`employee`：

```swift
let sameEmployee = try decoder.decode(Employee.self, from: data)
```

## 处理多级嵌套的JSON

现在，JSON的格式变成下面这样：

```JSON
{
  "name" : "John Appleseed",
  "id" : 7,
  "gift" : {
    "toy" : {
      "name" : "Teddy Bear"
    }
  }
}
```
`name`字段在`toy`字段中，而`toy`又在`gift`字段中。如何解析成我们定义的数据模型呢？

打开本节代码`Nested keyed containers`，添加下面的代码到`Employee`：

```swift
// 1  
enum CodingKeys: CodingKey {  
  case name, id, gift
}
// 2
enum GiftKeys: CodingKey {
  case toy
}
// 3
func encode(to encoder: Encoder) throws {
  var container = encoder.container(keyedBy: CodingKeys.self)
  try container.encode(name, forKey: .name)
  try container.encode(id, forKey: .id)
  // 4  
  var giftContainer = container
    .nestedContainer(keyedBy: GiftKeys.self, forKey: .gift)
  try giftContainer.encode(favoriteToy, forKey: .toy)
}
```
这里做了几件事：
1. 创建顶层的`CodingKeys`
2. 创建用于解析`gift`字段的`CodingKeys`，后续使用它创建容器
3. 使用顶层容器编码`name`和`id`
4. 使用`nestedContainer(keyedBy:forKey:)`方法获取用于编码`gift`字段的容器，并将`favoriteToy`编码进去
运行并查看`string`的值，你会发现JSON的格式符合预期。

解码过程也很类似。添加下面的代码：

```swift
extension Employee: Decodable {
  init(from decoder: Decoder) throws {
    let container = try decoder.container(keyedBy: CodingKeys.self)
    name = try container.decode(String.self, forKey: .name)
    id = try container.decode(Int.self, forKey: .id)
    let giftContainer = try container
      .nestedContainer(keyedBy: GiftKeys.self, forKey: .gift)
    favoriteToy = try giftContainer.decode(Toy.self, forKey: .toy)
  }
}

let sameEmployee = try decoder.decode(Employee.self, from: nestedData)
```
好了，我们已经搞定了嵌套类型的容器。并从其中解码出了`sameEmployee`。

## 处理日期类型

现在，JSON里添加了日期字段，就像下面这样：

```json
{
  "id" : 7,
  "name" : "John Appleseed",
  "birthday" : "29-05-2019",
  "toy" : {
    "name" : "Teddy Bear"
  }
}
```

JSON中并没有标准的日期格式。在`JSONEncoder`和`JSONDecoder`使用日期类的`timeIntervalSinceReferenceDate`方法去处理(`Date(timeIntervalSinceReferenceDate: interval)`)。

这里我们需要指定`日期转换策略`。打开本节代码`Dates`，在`try encoder.encode(employee)`之前添加下面的代码：

```swift
// 1
extension DateFormatter {
  static let dateFormatter: DateFormatter = {
    let formatter = DateFormatter()
    formatter.dateFormat = "dd-MM-yyyy"
    return formatter
  }()
}
// 2
encoder.dateEncodingStrategy = .formatted(.dateFormatter)
decoder.dateDecodingStrategy = .formatted(.dateFormatter)
```
这里主要做了2件事：
1. 在`DateFormatter`的扩展中添加了`格式化器`，它的格式化形式满足JSON中日期的格式，并且是可以重用的。
2. 设置`dateEncodingStrategy`和`dateDecodingStrategy`为`.formatted(.dateFormatter)`，这样编解码时就会使用它去处理日期

运行并检查`dateString`的内容，你会发现它符合预期。

## 处理子类

现在，JSON格式变成了下面这样：

```json
{
  "toy" : {
    "name" : "Teddy Bear"
  },
  "employee" : {
    "name" : "John Appleseed",
    "id" : 7
  },
  "birthday" : 580794178.33482599
}
```
这里将`Employee`所需信息分开了。我们打算使用`BasicEmployee`去解析`employee`。打开本节代码`Subclasses`，使`BasicEmployee`遵循`Codable`：

```swift
class BasicEmployee: Codable {
```
不出意外，编译器报错了，因为`GiftEmployee`并没有遵循`Codable`。我们继续添加下面的代码，就可以修正错误了：

```swift
// 1              
enum CodingKeys: CodingKey {
  case employee, birthday, toy
}  
// 2
required init(from decoder: Decoder) throws {
  let container = try decoder.container(keyedBy: CodingKeys.self)
  birthday = try container.decode(Date.self, forKey: .birthday)
  toy = try container.decode(Toy.self, forKey: .toy)
  // 3
  let baseDecoder = try container.superDecoder(forKey: .employee)
  try super.init(from: baseDecoder)
}  
```
这里做了3件事：
1. 在`GiftEmployee`中添加了`CodingKeys`。和`JSON`中的字段名对应。
2. 从`decoder`解码出子类的属性值。
3. 创建用于解码父类属性的`Decoder`，然后调用父类的方法初始化父类属性。

下面我们继续完成`GiftEmployee`的编码方法：

```swift
override func encode(to encoder: Encoder) throws {
  var container = encoder.container(keyedBy: CodingKeys.self)
  try container.encode(birthday, forKey: .birthday)
  try container.encode(toy, forKey: .toy)
  let baseEncoder = container.superEncoder(forKey: .employee)
  try super.encode(to: baseEncoder)
}
```
和解码过程类似，我们先编码了子类的属性，然后获取用于编码父类的`encoder`。下面测试下结果：

```swift
let giftEmployee = GiftEmployee(name: "John Appleseed", id: 7, birthday: Date(),  toy: toy)
let giftData = try encoder.encode(giftEmployee)
let giftString = String(data: giftData, encoding: .utf8)!
let sameGiftEmployee = try decoder.decode(GiftEmployee.self, from: giftData)
```
运行并检查`giftString`，你会发现其内容符合预期。学习了本节，你就可以处理更复杂的继承数据模型了。

## 处理混合类型的数组

现在，JSON格式变成了下面这样：

```swift
[
  {
    "name" : "John Appleseed",
    "id" : 7
  },
  {
    "id" : 7,
    "name" : "John Appleseed",
    "birthday" : 580797832.94787002,
    "toy" : {
      "name" : "Teddy Bear"
    }
  }
]
```
这是个JSON数组，但是其内部元素格式并不一致。打开本节代码`Polymorphic types`，可以看到这里使用枚举定义了不同类型的数据。

首先，我们让`AnyEmployee`遵循`Encodable`协议：

```swift
enum AnyEmployee: Encodable { ... }
```
继续在`AnyEmployee`中添加下面的代码：

```swift
  // 1
enum CodingKeys: CodingKey {
  case name, id, birthday, toy
}  
// 2
func encode(to encoder: Encoder) throws {
  var container = encoder.container(keyedBy: CodingKeys.self)
  
  switch self {
    case .defaultEmployee(let name, let id):
      try container.encode(name, forKey: .name)
      try container.encode(id, forKey: .id)
    case .customEmployee(let name, let id, let birthday, let toy):  
      try container.encode(name, forKey: .name)
      try container.encode(id, forKey: .id)
      try container.encode(birthday, forKey: .birthday)
      try container.encode(toy, forKey: .toy)
    case .noEmployee:
      let context = EncodingError.Context(codingPath: encoder.codingPath, 
                                          debugDescription: "Invalid employee!")
      throw EncodingError.invalidValue(self, context)
  }
}
```
这里我们主要做了两件事：
1. 定义了所有可能的键。
2. 根据不同类型，对数据进行编码。

在代码的最后添加下面的内容来进行测试：

```swift
let employees = [AnyEmployee.defaultEmployee("John Appleseed", 7), 
                 AnyEmployee.customEmployee("John Appleseed", 7, Date(),toy)]
let employeesData = try encoder.encode(employees)
let employeesString = String(data: employeesData, encoding: .utf8)!
```

接下来的编码过程有点复杂。继续添加下面的代码：

```swift
extension AnyEmployee: Decodable {
  init(from decoder: Decoder) throws {
    // 1
    let container = try decoder.container(keyedBy: CodingKeys.self) 
    let containerKeys = Set(container.allKeys)
    let defaultKeys = Set<CodingKeys>([.name, .id])
    let customKeys = Set<CodingKeys>([.name, .id, .birthday, .toy])
   
    // 2
   switch containerKeys {
      case defaultKeys:
        let name = try container.decode(String.self, forKey: .name)
        let id = try container.decode(Int.self, forKey: .id)
        self = .defaultEmployee(name, id)
      case customKeys:
        let name = try container.decode(String.self, forKey: .name)
        let id = try container.decode(Int.self, forKey: .id)
        let birthday = try container.decode(Date.self, forKey: .birthday)
        let toy = try container.decode(Toy.self, forKey: .toy)
        self = .customEmployee(name, id, birthday, toy)
      default:
        self = .noEmployee
    }
  }
}
// 3
let sameEmployees = try decoder.decode([AnyEmployee].self, from: employeesData) 
```
解释下上面的代码：
1. 获取`KeydContainer`，并获取其所有键。
2. 根据不同的键，实行不同的解析策略
3. 从`employeesData`中解码出`[AnyEmployee]`

> 个人感觉若数组中的元素可以用同一模型来表示，只是字段可能为空时，直接将模型字段设为可选。当然这里也提供了解析不同模型的思路。

## 处理数组

现在，我们有如下格式JSON：

```json
[
  "teddy bear",
  "TEDDY BEAR",
  "Teddy Bear"
]
```
这里是一个数组，并且其大小写各不相同。此时我们不需要任何`CodingKey`，只需使用`unkeyed container`。

打开本节代码`Unkeyed containers`，添加下面的代码到`Label`结构体中:

```swift
func encode(to encoder: Encoder) throws {
  var container = encoder.unkeyedContainer()
  try container.encode(toy.name.lowercased())
  try container.encode(toy.name.uppercased())
  try container.encode(toy.name)
}
```
`UnkeyedEncodingContainer`和之前用到的`KeyedEncodingContainer`相似，但是它不需要`CodingKey`，因为它将编码数据写入JSON数组中。这里我们编码了3中不同的字符串到其中。

继续解码：

```swift
extension Label: Decodable {
  // 1
  init(from decoder: Decoder) throws {
    var container = try decoder.unkeyedContainer()
    var name = ""
    while !container.isAtEnd {
      name = try container.decode(String.self)
    }
    toy = Toy(name: name)
  }
}
let sameLabel = try decoder.decode(Label.self, from: labelData)
```
这里主要是获取`decoder.unkeyedContainer`，获取容器中最后一个值来初始化`name`。

## 处理嵌套在对象中的数组

现在我们有如下格式JSON：

```json
{
  "name" : "Teddy Bear",
  "label" : [
    "teddy bear",
    "TEDDY BEAR",
    "Teddy Bear"
  ]
}
```
这次，标签对应在了`label`字段下。我们需要使用`nested unkeyed containers`去进行编解码。

打开本节代码`Nested unkeyed containers`，在`Toy`中添加下面的代码：

```swift
func encode(to encoder: Encoder) throws {
  var container = encoder.container(keyedBy: CodingKeys.self)
  try container.encode(name, forKey: .name)
  var labelContainer = container.nestedUnkeyedContainer(forKey: .label)                   
  try labelContainer.encode(name.lowercased())
  try labelContainer.encode(name.uppercased())
  try labelContainer.encode(name)
}
```
这里我们创建了一个`nested unkeyed container`，并填充了3个字符串。运行代码，并查看`string`的值，可以看到预期结果。

继续添加下面的代码进行解码：

```swift
extension Toy: Decodable {
  init(from decoder: Decoder) throws {
    let container = try decoder.container(keyedBy: CodingKeys.self)
    name = try container.decode(String.self, forKey: .name)
    var labelContainer = try container.nestedUnkeyedContainer(forKey: .label)
    var labelName = ""
    while !labelContainer.isAtEnd {
      labelName = try labelContainer.decode(String.self)
    }
    label = labelName
  }
}
let sameToy = try decoder.decode(Toy.self, from: data)
```
这里，我们像之前一样，使用`unkeyed container`的最后一个值初始化`label`字段，只不过获取的是嵌套的容器。

## 处理可选字段

最后，我们的模型中的属性也可以是可选类型，`container`也提供了对应的编解码方法：

```swift
encodeIfPresent(value, forKey: key)
decodeIfPresent(type, forKey: key)
```

## 总结

今天我们由浅入深的学习了如何在`Swift`中处理`JSON`。其中`自定义Coding keys`、`处理子类`等部分需要重点理解。希望对大家有所帮助。