---
title: 【Swift】无主引用unowned、隐式解析可选类型
categories:
  - iOS
date: 2017-08-30 16:32:10
tags:  
 - Swift  
 - unowned  
 - 隐式解析可选类型
---
<!-- more -->

## 类实例之间的循环强引用

我们先来举个简单循环强引用的栗子，定义两个类Person和Apartment，分别代表公寓和它的居民。Person类有一个可选属性apartment，表示一个人可能拥有公寓。Apartment类有一个可选属性tenant，表示一栋公寓可能拥有居民。

接下来这段代码就会产生循环强引用了：
``` swift
let john = Person(name: "John Appleseed")
let unit4A = Apartment(unit: "4A")

john!.apartment = unit4A
unit4A!.tenant = john
```

Swift 提供了两种办法用来解决你在使用类的属性时所遇到的循环强引用问题：

- **弱引用 weak reference**
- **无主引用 unowned reference**

## 弱引用
上例的解决办法就是将apartment或tenant属性声明为弱引用即可，如果有对此例不理解的同学，可以参考[官方文档](https://github.com/numbbbbb/the-swift-programming-language-in-chinese/blob/gh-pages/source/chapter2/16_Automatic_Reference_Counting.md#strong_reference_cycles_between_class_instances)，有配图，讲解得很详细。

关于弱引用大家多少都有所了解，本文重点也不在此，故不再赘述，只提一点官方文档中的注意事项：
> 注意
> 
当 ARC 设置弱引用为nil时，属性观察不会被触发。

## 无主引用
可以在声明属性或者变量时，在前面加上关键字unowned表示这是一个无主引用。

和弱引用类似，无主引用不会牢牢保持住引用的实例。和弱引用不同的是，无主引用在其他实例有相同或者更长的生命周期时使用。

无主引用通常都被期望拥有值。不过 **ARC 无法在实例被销毁后将无主引用设为nil**，因为非可选类型的变量不允许被赋值为nil。

> 重要
>
使用无主引用，你必须确保引用始终指向一个未销毁的实例。
如果你试图在实例被销毁后，访问该实例的无主引用，会触发运行时错误。

下面的例子定义了两个类，Customer和CreditCard，模拟了银行客户和客户的信用卡。这两个类中，每一个都将另外一个类的实例作为自身的属性。这种关系可能会造成循环强引用。
``` swift
class Customer {
    let name: String
    var card: CreditCard?
    init(name: String) {
        self.name = name
    }
    deinit { print("\(name) is being deinitialized") }
}

class CreditCard {
    let number: UInt64
    unowned let customer: Customer
    init(number: UInt64, customer: Customer) {
        self.number = number
        self.customer = customer
    }
    deinit { print("Card #\(number) is being deinitialized") }
}
```

在这个数据模型中，一个客户可能有或者没有信用卡，但是一张信用卡总是关联着一个客户。为了表示这种关系，Customer类有一个可选类型的card属性，但是CreditCard类有一个非可选类型的customer属性。
此外，只能通过将一个number值和customer实例传递给CreditCard构造函数的方式来创建CreditCard实例。这样可以确保当创建CreditCard实例时总是有一个customer实例与之关联。

下面的代码片段定义了一个叫john的可选类型Customer变量，用来保存某个特定客户的引用：
```swift
var john: Customer?
```

然后我们就可以创建Customer类的实例，用它初始化CreditCard实例，并将新创建的CreditCard实例赋值为客户的card属性：
```swift
john = Customer(name: "John Appleseed")
john!.card = CreditCard(number: 1234_5678_9012_3456, customer: john!)
```

在你关联两个实例后，它们的引用关系如下图所示：
![](unowned0.png)

Customer实例持有对CreditCard实例的强引用，而CreditCard实例持有对Customer实例的无主引用。
由于customer的无主引用，当你断开john变量持有的强引用时，再也没有指向Customer实例的强引用了：
![](unowned1.png)

由于再也没有指向Customer实例的强引用，该实例被销毁了。其后，再也没有指向CreditCard实例的强引用，该实例也随之被销毁了：
```swift
john = nil
// 打印 "John Appleseed is being deinitialized"
// 打印 "Card #1234567890123456 is being deinitialized"
```

最后的代码展示了在john变量被设为nil后Customer实例和CreditCard实例的构造函数都打印出了“销毁”的信息。

> 注意
>
上面的例子展示了如何使用安全的无主引用。对于需要禁用运行时的安全检查的情况（例如，出于性能方面的原因），Swift还提供了不安全的无主引用。与所有不安全的操作一样，你需要负责检查代码以确保其安全性。 你可以通过unowned(unsafe)来声明不安全无主引用。如果你试图在实例被销毁后，访问该实例的不安全无主引用，你的程序会尝试访问该实例之前所在的内存地址，这是一个不安全的操作。

## 无主引用以及隐式解析可选属性
- Person和Apartment的例子展示了**两个属性的值都允许为nil**，并会潜在的产生循环强引用。这种场景最适合用弱引用来解决。
- Customer和CreditCard的例子展示了**一个属性的值允许为nil，而另一个属性的值不允许为nil**，这也可能会产生循环强引用。这种场景最适合通过无主引用来解决。
- 然而，存在着第三种场景，在这种场景中，**两个属性都必须有值，并且初始化完成后永远不会为nil**。在这种场景中，需要**一个类使用无主属性，而另外一个类使用隐式解析可选属性**。

下面的例子定义了两个类，Country和City，每个类将另外一个类的实例保存为属性。每个国家必须有首都，每个城市必须属于一个国家。所以Country类拥有一个capitalCity属性，而City类有一个country属性：
```swift
class Country {
    let name: String
    var capitalCity: City!
    init(name: String, capitalName: String) {
        self.name = name
        self.capitalCity = City(name: capitalName, country: self)
    }
}
class City {
    let name: String
    unowned let country: Country
    init(name: String, country: Country) {
        self.name = name
        self.country = country
    }
}
```

为了建立两个类的依赖关系，City的构造函数接受一个Country实例作为参数，并且将实例保存到country属性。

Country的构造函数调用了City的构造函数。然而，只有Country的实例完全初始化后，Country的构造函数才能把self传给City的构造函数。在[两段式构造过程](https://github.com/numbbbbb/the-swift-programming-language-in-chinese/blob/gh-pages/source/chapter2/14_Initialization.html#two_phase_initialization)中有具体描述。

为了满足这种需求，通过在类型结尾处加上感叹号（City!）的方式，将Country的capitalCity属性声明为隐式解析可选类型的属性。这意味着像其他可选类型一样，capitalCity属性的默认值为nil，但是不需要展开它的值就能访问它。在[隐式解析可选类型](https://github.com/numbbbbb/the-swift-programming-language-in-chinese/blob/gh-pages/source/chapter2/01_The_Basics.html#implicityly_unwrapped_optionals)中有描述。

由于capitalCity默认值为nil，一旦Country的实例在构造函数中给name属性赋值后，整个初始化过程就完成了。这意味着一旦name属性被赋值后，Country的构造函数就能引用并传递隐式的self。Country的构造函数在赋值capitalCity时，就能将self作为参数传递给City的构造函数。

以上的意义在于你可以通过一条语句同时创建Country和City的实例，而不产生循环强引用，并且capitalCity的属性能被直接访问，而不需要通过感叹号来展开它的可选值：
```swift
var country = Country(name: "Canada", capitalName: "Ottawa")
print("\(country.name)'s capital city is called \(country.capitalCity.name)")
// 打印 "Canada's capital city is called Ottawa"
```

在上面的例子中，使用隐式解析可选值意味着满足了类的构造函数的两个构造阶段的要求。capitalCity属性在初始化完成后，能像非可选值一样使用和存取，同时还避免了循环强引用。

## 闭包引起的循环强引用
在定义闭包时同时定义捕获列表作为闭包的一部分，通过这种方式可以解决闭包和类实例之间的循环强引用。捕获列表定义了闭包体内捕获一个或者多个引用类型的规则。

## 定义捕获列表
捕获列表中的每一项都由一对元素组成，一个元素是weak或unowned关键字，另一个元素是类实例的引用（例如self）或初始化过的变量（如delegate = self.delegate!）。这些项在方括号中用逗号分开。

如果闭包有参数列表和返回类型，把捕获列表放在它们前面：
```swift
lazy var someClosure: (Int, String) -> String = {
    [unowned self, weak delegate = self.delegate!] (index: Int, stringToProcess: String) -> String in
    // 这里是闭包的函数体
}
```

如果闭包没有指明参数列表或者返回类型，即它们会通过上下文推断，那么可以把捕获列表和关键字in放在闭包最开始的地方：
```swift
lazy var someClosure: Void -> String = {
    [unowned self, weak delegate = self.delegate!] in
    // 这里是闭包的函数体
}
```

## 弱引用和无主引用
在闭包和捕获的实例总是互相引用并且总是同时销毁时，将闭包内的捕获定义为无主引用。相反的，在被捕获的引用可能会变为nil时，将闭包内的捕获定义为弱引用。弱引用总是可选类型，并且当引用的实例被销毁后，弱引用的值会自动置为nil。这使我们可以在闭包体内检查它们是否存在。

>注意
>
如果被捕获的引用绝对不会变为nil，应该用无主引用，而不是弱引用。

#### 参考链接：
[the-swift-programming-language-in-chinese](https://github.com/numbbbbb/the-swift-programming-language-in-chinese/blob/gh-pages/source/chapter2/16_Automatic_Reference_Counting.md#arc_in_action)