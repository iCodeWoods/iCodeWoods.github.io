---
title: 【译】The Best New Features in Swift 4
categories:
  - iOS
date: 2018-01-23 16:20:03
tags: 
 - 译文
 - Swift
 - New Features
---
*本文翻译自：[Mike Ash's Blog](https://plausible.coop/blog/2017/09/13/best-new-features-in-swift-4)，略有修改*

*译者：[iCodeWoods](https://icodewoods.github.io/)*

Swift 4 来了，并且发生了很多不错的改变。Swift 4 既没有像去年的 Swift 3 那样几乎完全修改了语法，也没有像 Swift 2 那样创建了很多新功能。但是 Swift 4 有一些很不错的改善代码的地方，我们来看一看吧！

<!-- more -->

## 多行字符串文字（Multi-Line String Literals）

有时你的代码中需要使用很长的、多行的字符串，它可能是一个HTML模板，可能是XML的一个块，或者是一个很长的用户信息。无论是哪一种，在 Swift 3 中写起来都很痛苦。

你可以把它们全部写在一行中，但是代码很快就会变得很丑：
``` Swift
let message = "Please disable your Frobnitz before proceeding.\n\nTo do this, visit Settings -> Frobnitz, then toggle the switch to \"off\".\n\nIf you need the Frobnitz to remain enabled, tap \"Proceed Anyway\" below."
```

你可以通过连接字符串将它分成多行：
``` Swift
let message =
    "Please disable your Frobnitz before proceeding.\n\n"
  + "To do this, visit Settings -> Frobnitz, then toggle the switch to \"off\".\n\n"
  + "If you need the Frobnitz to remain enabled, tap \"Proceed Anyway\" below."
```

还有很多其它方法，但是这些方法全都不好。

Swift 4 用多行字符串文字解决了这个问题。 要编写多行字符串文字，请在开头和结尾使用三个引号：
``` Swift
let message = """
    Please disable your Frobnitz before proceeding.

    To do this, visit Settings -> Frobnitz, then toggle the switch to "off".

    If you need the Frobnitz to remain enabled, tap "Proceed Anyway" below.
    """
```

如果你写过 Python 的话，一定很熟悉这种语法。不过它们不完全一样。在 Swift 中这个语法有一些有趣的限制和特性。

这个三引号语法不能写在一行中，像下面这样写编译会不通过：
``` Swift
// Will not compile
label.text = """Put your text in "quotes" to make them look quoted."""
```

可以在代码中缩进多行字符串，而不缩进最终结果。上面的多行字符串会缩进代码中的每一行，但字符串没有前导空格。这真的很棒，但是如果你想要缩进呢？ 这个特性是基于结尾的"""标记的缩进，所有其他行的缩进都将除去和它一样的缩进，如果由于某种原因你需要缩进内容的话，你可以缩进文本，让它们离结尾的"""标记更远：
``` Swift
let message = """
        Please disable your Frobnitz before proceeding.

        To do this, visit Settings -> Frobnitz, then toggle the switch to "off".

        If you need the Frobnitz to remain enabled, tap "Proceed Anyway" below.
    """
```

为避免混淆，每行文本的缩进至少要和结尾的"""标记一样多，缩进比它还少的行会报错。

如果你想在写的时候将文本分成多行，但是输出时还是一行，那么你可以在末尾加上'\'：
``` Swift
let message = """
    Please disable your Frobnitz before proceeding. \
    To do this, visit Settings -> Frobnitz, then toggle the switch to "off". \
    If you need the Frobnitz to remain enabled, tap "Proceed Anyway" below.
    """
```

## 单侧范围（One-Sided Ranges）

这是一个很好的、很小的变化。Ranges 现在可以只有一侧，而空的那一侧则会被自动推断出，代表相应的最小值或最大值。

当为一个容器进行 subscripting 时，这意味着你可以忽略 string.endIndex 或 array.count 之类的东西。

例如，如果你想将一个数组分成两半：
``` Swift
let middle = array.count / 2
let firstHalf = array[..<middle]
let secondHalf = array[middle...]
```

或者你想得到一个从0至某个index的子串：
``` Swift
let index = string.index(of: "e")!
string[..<index]
```

它也可以在 switch 语句中使用：
``` Swift
switch x {
case ..<0:
    print("That's a negative.")
case 0:
    print("Nothing!")
case 1..<10:
    print("Pretty small.")
case 10..<100:
    print("Bigger.")
case 100...:
    print("Huge!")
default:
    // 遗憾的是编译器并不知道上面的 cases 其实已经包含了所有情况，所以还是要写 default
    break
}
```

对于一个从0开始到指定index的单侧范围来说，'..<'和'...'都可以使用。比如 array[..<3]、array[...5]

而对于一个从指定index到结尾的单侧范围来说，只能使用'...'，比如 array[2...]，因为在这种情况下'..<'是没有意义的。

## 组合的类和协议类型（Combined Class and Protocol Types）

有时你需要这样一个对象，它既是一个类的子类，又遵守一个协议。 

例如，你可能需要一个遵守KittenProvider协议的UITableViewController对象。 Swift 3需要各种丑陋的解决方法才能表示这个对象。有趣的是，Objective-C却能够表示：
``` Objective-C
UITableViewController<KittenProvider> *object;
```

Swift 4 现在通过使用 '&' 符号也能表达这个概念了。这个符号可能已经被用来将多个协议组合成一个单一的类型，现在它也可以被用来将协议与一个类相结合：
``` Swift
let object: UITableViewController & KittenProvider
```

注意，在这种 type 中只能有一个 class，因为你不能同时生成多个子类：）

## 范型下标（Generic Subscripts）

Swift 早就支持了范型方法，但是却不支持范型下标。你可以通过实现多个不同类型的下标来重载它，但是不能使用范型。现在可以了！
``` Swift
subscript<T: Hashable>(key: T) -> Value?
```

范型是全方面支持的，所以你也可以在 where 子句中使用它：
``` Swift
subscript<S: Sequence>(key: S) -> [Value] where S.Element == Key
```

就像方法一样，范型也可以被用作返回值：
``` Swift
subscript<T>(key: Key) -> T?
```

这对于处理动态类型的容器而言非常方便，比如在处理JSON对象时。

*译者注：增加一个demo*
``` Swift
struct GenericDictionary<Key: Hashable, Value> {
  private var data: [Key: Value]

  init(data: [Key: Value]) {
    self.data = data
  }

  subscript<T>(key: Key) -> T? {
    return data[key] as? T
  }
}

// Dictionary of type: [String: Any]
var earthData = GenericDictionary(data: ["name": "Earth", "population": 7500000000, "moons": 1])

// 自动推断出返回类型，而不用写 "as? String"
let name: String? = earthData["name"]

// 自动推断出返回类型，而不用写 "as? Int"
let population: Int? = earthData["population"]

extension GenericDictionary {
  subscript<Keys: Sequence>(keys: Keys) -> [Value] where Keys.Iterator.Element == Key {
    var values: [Value] = []
    for key in keys {
      if let value = data[key] {
        values.append(value)
      }
    }
    return values
  }
}

// Array subscript value
let nameAndMoons = earthData[["moons", "name"]]        // [1, "Earth"]
// Set subscript value
let nameAndMoons2 = earthData[Set(["moons", "name"])]  // [1, "Earth"]
```
*译者注：这个例子中，你可以看到传入两个不同序列类型作为下标（Array和Set），会得到他们各自的值。*

## 可编码（Codable）

说到JSON，也许Swift 4中最大的新功能就是Codable协议。 
编译器现在会为你的类型自动生成序列化和反序列化代码，你所要做的就是声明遵守了Codable协议。

想象一下你有一个Person类型：
``` Swift
struct Person {
    var name: String
    var age: Int
    var quest: String
}
```

以前如果你想从JSON中读取Person类的值或将它们写入JSON中，不得不写一堆烦人的重复代码来这样做（*译者注：一堆encode和decode*）。

而在 Swift 4 中，你只需要半行代码就能做到：
``` Swift
struct Person: Codable {
```

如果由于某种原因你只想要支持编码或解码中的一个，而不是同时支持，则可以分别声明遵守Encodable或Decocable：
``` Swift
struct EncodablePerson: Encodable { ... }

struct DecodablePerson: Decodable { ... }
```

遵守 Codable 协议只是对遵守它们俩的一个简写而已。

使用 Codable 类型需要一个编码器或解码器，它决定了序列化格式以及如何将 Swift 值转换为序列化值。Swift 为 JSON 和 property list 提供了编码器和解码器，Foundation 的 archive 也支持 Codable 类型。

如果想将某个东西编码为 JSON，可以创建一个 JSONEncoder 并调用它的 encode 方法：
``` Swift
let jsonEncoder = JSONEncoder()
let data = try jsonEncoder.encode(person)
```

要解码时，创建一个 JSONDecoder 并调用它的 decode 方法，传入你想解码成的类型，和要解码的数据源：
``` Swift
let jsonDecoder = JSONDecoder()
let decodedPerson = try jsonDecoder.decode(Person.self, from: data)
```

注意编码和解码方法被标记为 *throws* 的，因为在处理的过程中可能会发生很多错误，比如类型不匹配或者数据不完整等。所以你需要在这些调用中加上 *try*，并 *catch* 它们抛出的错误。

由于JSON本身不支持日期或二进制数据，因此需要将这些值转换为其他JSON表示。 例如，对二进制数据使用base64编码是很常见的。 JSONEncoder和JSONDecoder可以用不同的策略来定制以处理这些值。 例如，如果要将日期编码为ISO-8601，并将数据编码为base64：
``` Swift
let jsonEncoder = JSONEncoder()
jsonEncoder.dateEncodingStrategy = .iso8601
jsonEncoder.dataEncodingStrategy = .base64
let data = try jsonEncoder.encode(person)

// 反之亦然
let jsonDecoder = JSONDecoder()
jsonDecoder.dateDecodingStrategy = .iso8601
jsonDecoder.dataDecodingStrategy = .base64
let decodedPerson = try jsonDecoder.decode(Person.self, from: data)
```

他们还提供了通过编写自己的代码来提供完全自定义策略的选项。

属性列表（Property list）编码是相似的。使用 PropertyListEncoder 和 PropertyListDecoder 来代替 JSON 编码器。由于属性列表本身可以表示日期和二进制数据，属性列表编码器不提供​​这些选项。

如果你已经在使用 NSCoding 了，那么你可以将它与 Codable 混合搭配使用，不用马上就全部改掉。NSKeyedArchiver 提供了一个新的 encodeEncodable 方法，它可以在指定的 key 下对任何可编码的类型进行编码。NSKeyedUnarchiver 提供了相应的 decodeDecodable，可以解码任何可解码的类型。

Codable 是一个灵活的协议，有很多自定义行为的空间。编译器提供了一个默认实现，但如果你需要不同的行为，则可以提供自己的实现。这使得编写将旧数据迁移到新类型、在序列化表示中使用与源代码中不同的名称或进行其他自定义的实现变得非常简单。

对 Codable 的所有可能性的全面讨论超出了本文的范围，但是你可以在我写的 [Friday Q&A post about Swift.Codable](https://mikeash.com/pyblog/friday-qa-2017-07-14-swiftcodable.html) 中阅读更多内容。

## 总结

Swift 4 并没有像前几年那样发生巨大的变化，但却是一次实打实的进步。Codable 将使一些常见的任务变得非常容易，这可能是我最喜欢的功能之一。像多行字符串和范型下标等其他功能不会有这么大的影响，但它们应该会使我们写的代码有很好的改善。