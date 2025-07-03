[ #SwiftData ](https://fatbobman.com/zh/tags/SwiftData/)[ #Swift ](https://fatbobman.com/zh/tags/Swift/) [ #English ](https://fatbobman.com/en/posts/swift-predicate-usage-composition-and-considerations/) 

# Swift Predicate: 用法、构成及注意事项

[东坡肘子](https://x.com/intent/follow?screen%5Fname=fatbobman) 

 发表于 2024 年 2 月 29 日 

NSPredicate 一直是 Apple 提供的一个强大工具，允许开发者通过定义复杂的逻辑条件以自然且高效的方式对数据集合进行筛选和评估。随着时间的推移，Swift 语言的不断成熟和发展，2023 年 Swift 社区着手使用纯 Swift 语言重构 Foundation 框架。在这一重大更新中，引入了基于 Swift 编码的新 Predicate 功能，标志着在数据处理和评估方面迈入了新的阶段。本文旨在探讨 Swift Predicate 的使用方法、构成以及在实际开发中应注意的关键事项。

[ 马上加入 ![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)的开发者社区!  加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长! ](https://t.ly/gzxeh) 

 🚀 马上加入，享受交流的快乐 →

## 什么是谓词

在现代软件开发中，对数据进行高效且精确的筛选和评估是至关重要的。谓词（Predicate）作为一种强大的工具，允许开发者通过定义返回布尔值（true 或 false）的逻辑条件，来实现这一目标。这不仅在筛选集合或查找集合中的特定元素时发挥着核心作用，而且也是数据处理和业务逻辑实现的基石。

尽管苹果的 `NSPredicate` 提供了这种能力，但它依赖于 Objective-C 语法、存在容易出现运行时错误的风险以及面临平台限制等挑战，这些限制了其在不同环境下的应用范围和灵活性。

Swift 

```
class MyObject: NSObject {
  @objc var name: String
  init(name: String) {
    self.name = name
  }
}
let object = MyObject(name: "fat")

// create NSPredicate
let predicate = NSPredicate(format: "name = %@", "fat")
XCTAssertTrue(predicate.evaluate(with: object)) // true

let objs = [object]
// filter object by predicate
let filteredObjs = (objs as NSArray).filtered(using: predicate) as! [MyObject]
XCTAssertEqual(filteredObjs.count, 1)
```

### Swift Predicate 的引入与改进

为了克服这些限制并拓展谓词的应用范围，Swift 社区对 Foundation 框架进行了重构，引入了基于 Swift 语言的 Predicate 功能。这一新特性不仅摆脱了对 Objective-C 的依赖，还通过 Swift 的宏功能，简化了谓词的构建过程，如下所示：

Swift 

```
class MyObject {
  var name:String
  init(name: String) {
    self.name = name
  }
}

let object = MyObject(name: "fat")
let predicate = #Predicate<MyObject>{ $0.name == "fat" }
try XCTAssertTrue(predicate.evaluate(object)) // true

let objs = [object]
let filteredObjs = try objs.filter(predicate)
XCTAssertEqual(filteredObjs.count, 1)
```

在此示例中，我们通过 `#Predicate` 宏构建了一个逻辑条件。这种构建方式非常类似于撰写闭包代码，使得开发者能够以自然的方式构建出更加复杂的逻辑，例如：包含多个条件的谓词：

Swift 

```
let predicate = #Predicate<MyObject>{ object in
  object.name == "fat" && object.name.count < 3
}
try XCTAssertTrue(predicate.evaluate(object)) // false
```

此外，现在的 `MyObject` 无需继承自 `NSObject` 或使用 `@objc` 标注其属性，以支持 KVC。当然，Swift Predicate 同样适用于仍继承自 `NSObject` 的类型。

### NSPredicate 与 Swift Predicate 的比较

相较于 NSPredicate，Swift Predicate 提供了诸多改进：

* **开源性与平台兼容性**：支持跨平台使用，如 Linux 和 Windows。
* **类型安全**：利用 Swift 的类型检查减少运行时错误。
* **开发效率**：受益于 Xcode 支持，提高了代码编写的速度和准确性。
* **语法自由度**：提供更大的表达自由，不受 Objective-C 语法规则的限制。
* **泛用性**：可应用于所有 Swift 类型，不再限于继承自 NSObject 的类。
* **现代 Swift 特性支持**：支持 Sendable 和 Codable 等现代 Swift 特性，使其更适合当下的 Swift 编程范式。

通过这些改进，Swift Predicate 不仅优化了开发者的工作流程，而且为 Swift 生态系统的扩展和成长开辟了新路径。

## Swift Predicate 的主要构成

在深入探讨 Swift Predicate 的使用方法和注意事项之前，首先需要对其结构进行一番了解。具体来说，我们应该明白 Predicate 是由哪些元素构成的，以及 Predicate 宏是如何发挥作用的。

### PredicateExpression 协议

`PredicateExpression` 协议（或者说是遵循该协议的具体类型）定义了表达式的条件逻辑。例如，它能够代表一个“小于”条件，该条件包含具体的逻辑判断，用以决定某个输入值是否小于给定的值。这一协议是构建 Swift Predicate 架构中最为关键的部分。`PredicateExpression` 协议的声明如下：

Swift 

```
public protocol PredicateExpression<Output> {
    associatedtype Output
    
    func evaluate(_ bindings: PredicateBindings) throws -> Output
}
```

Foundation 提供了一系列[预定义](https://developer.apple.com/documentation/foundation/predicateexpressions?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web)的表达式类型，这些类型都遵循 `PredicateExpression` 协议，使得开发者能够直接利用 `PredicateExpressions` 下的类型或类型方法来构造谓词表达式。这为构建灵活而强大的条件评估逻辑铺平了道路。例如，若我们想构造一个代表数字 `4` 的表达式，相应的代码如下：

Swift 

```
let express = PredicateExpressions.Value(4)
```

`PredicateExpressions.Value` 的实现[代码](https://github.com/apple/swift-foundation/blob/b8ef4ce99ca7c5b8f39dbba7acb86721c916fb40/Sources/FoundationEssentials/Predicate/Expression.swift#L143?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web)如下所示：

Swift 

```
extension PredicateExpressions {
  public struct Value<Output> : PredicateExpression {
        public let value: Output
        
        public init(_ value: Output) {
            self.value = value
        }
        
        public func evaluate(_ bindings: PredicateBindings) -> Output {
            return self.value
        }
    }
}
```

`Value` 结构体直接封装了一个值，并在调用其 `evaluate` 方法时，简单地返回该被封装的值。这让 `Value` 成为了一种在谓词表达式中代表常量值的有效方式。

> 需要特别指出的是，`PredicateExpression` 的 `evaluate` 方法可以返回任何类型的值，而不仅限于布尔类型。

进一步，若我们需要定义一个表达 `3 < 4` 条件的表达式，相应的代码示例如下：

Swift 

```
let express = PredicateExpressions.build_Comparison(
  lhs: PredicateExpressions.Value(3),
  rhs: PredicateExpressions.Value(4),
  op: .lessThan
)
```

此代码片段将生成一个遵循 `PredicateExpression` 协议的类型实例：

Swift 

```
PredicateExpressions.Comparison<PredicateExpressions.Value<Int>, PredicateExpressions.Value<Int>>
```

调用此实例的 `evaluate` 方法时，将返回一个布尔值，即判断结果。

通过嵌套表达式的方式，开发者可以构建出极为复杂的逻辑判断。同时，所产生的类型表达式也相应地变得复杂。

### #Expression 宏

在 WWDC 2024 中，Foundation 的谓词系统引入了多项新功能，不仅增加了新的表达式方法，最为显著的改进是新增了 `#Expression` 宏，这大大简化了谓词表达式的构建过程。在之前的版本中，开发者仅在使用 `#Predicate` 宏时才能体验到构建的便捷性。现在，通过 `#Expression` 宏，即使是构建独立的表达式也能实现自然流畅的表达方式。

`#Expression` 宏使得开发者可以通过多个独立的表达式来分别定义谓词，这不仅使构建复杂谓词更为清晰，还增强了表达式的可复用性。

与谓词只能返回布尔值不同，表达式可以返回任意类型。因此，在声明表达式时，开发者需要明确指定输入和输出类型。

Swift 

```
let unplannedItemsExpression = #Expression<[BucketListItem], Int> { items in
    items.filter {
        !$0.isInPlan
    }.count
}

let today = Date.now
let tripsWithUnplannedItems = #Predicate<Trip>{ trip
    // The current date falls within the trip
    (trip.startDate ..< trip.endDate).contains(today) &&

    // The trip has at least one BucketListItem
    // where 'isInPlan' is false
    unplannedItemsExpression.evaluate(trip.bucketList) > 0
}
```

### Predicate 结构体

Swift Predicate，即使通过宏定义，其核心依然是 `Predicate` 结构体。这个结构体负责将逻辑条件（由 `PredicateExpression` 实现）与具体的值相绑定。这种机制使得 `Predicate` 能够实例化具体的条件逻辑，并接受输入值以进行评估。

它的[定义](https://github.com/apple/swift-foundation/blob/b8ef4ce99ca7c5b8f39dbba7acb86721c916fb40/Sources/FoundationEssentials/Predicate/Predicate.swift#L14?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web)如下所示：

Swift 

```
public struct Predicate<each Input> : Sendable {
    public let expression : any StandardPredicateExpression<Bool>
    public let variable: (repeat PredicateExpressions.Variable<each Input>)
    
    public init(_ builder: (repeat PredicateExpressions.Variable<each Input>) -> any StandardPredicateExpression<Bool>) {
        self.variable = (repeat PredicateExpressions.Variable<each Input>())
        self.expression = builder(repeat each variable)
    }
    
    public func evaluate(_ input: repeat each Input) throws -> Bool {
        try expression.evaluate(
            .init(repeat (each variable, each input))
        )
    }
}
```

主要特性包括：

* **布尔值返回限制**：`Predicate` 专门处理返回布尔值的表达式。这意味着复杂的表达式树的最终结果必须是一个布尔值，以便于进行逻辑判断。
* **构造过程**：在构造 `Predicate` 时，必须提供一个闭包，该闭包接收 `PredicateExpressions.Variable` 类型参数，并返回一个遵循 `StandardPredicateExpression<Bool>` 协议的表达式。
* **`StandardPredicateExpression` 协议**：这是对 `PredicateExpression` 协议的扩展，要求表达式同时遵循 `Codable` 和 `Sendable`。目前，官方只允许 Foundation 预置的表达式符合此协议。

Swift 

```
public protocol StandardPredicateExpression<Output> : PredicateExpression, Codable, Sendable {}
```

* **构造闭包和变量属性的高级特性**：利用 Swift 的 [Parameter Packs](https://github.com/apple/swift-evolution/blob/main/proposals/0393-parameter-packs.md?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 特性，`Predicate` 支持创建能同时处理多个泛型参数的谓词，这是 `NSPredicate` 所不具备的功能。

比如，利用 `Predicate` 结构体和 `PredicateExpression` 协议，我们可以构造出一个用于比较两个整数 `n` 和 `m`（`n < m`）的谓词示例：

Swift 

```
// 定义闭包：比较两个整数值是否满足"小于"关系
// 此闭包采用两个 PredicateExpressions.Variable<Int> 类型的参数，
// 并构造一个表示"小于"比较逻辑的 PredicateExpression
let express = { (value1: PredicateExpressions.Variable<Int>, value2: PredicateExpressions.Variable<Int>) in
    PredicateExpressions.build_Comparison(
        lhs: value1,
        rhs: value2,
        op: .lessThan
    )
}

// 使用 express 闭包构造 Predicate 实例，
// 其中 express 定义了评估逻辑，即判断第一个参数是否小于第二个参数
let predicate = Predicate {
    express($0, $1)
}

let n = 3
let m = 4

// 评估 predicate：检查 n 是否小于 m，预期返回 true
try XCTAssertTrue(predicate.evaluate(n, m))
```

### Predicate 宏

与通过字符串构建的 `NSPredicate` 相比，虽然直接使用 `PredicateExpression` 和 `Predicate` 结构体构建谓词能够获得类型安全检查、代码自动完成等优势，但这种方式在效率上较低，编写和阅读的难度也相对较高，这无疑增加了开发者在创建谓词时的心智负担。

为了降低这种复杂性，Foundation 引入了 Predicate 宏（ `#Predicate`），旨在以更简洁、高效的方式帮助开发者构建 Swift Predicate。

仍以构建判断 `n < m` 的谓词为例，通过使用宏可以大大地简化谓词的构建操作：

Swift 

```
let predicate = #Predicate<Int,Int>{ $0 < $1}
let n = 3
let m = 4
try XCTAssertTrue(predicate.evaluate(n,m)) // true
```

在 Xcode 中，通过查看宏展开后生成的代码，我们可以清楚地看到宏如何简化了之前需要大量代码才能实现的逻辑。

![](https://cdn.fatbobman.com/image-20240225182917655-zipic.png) 

Predicate 宏的[实现代码](https://github.com/apple/swift-foundation/blob/5b06c5d5ac15c6eb052ac1f91fc023d4299b5f66/Sources/FoundationMacros/PredicateMacro.swift?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web)大约有 1200 行，其只支持 Foundation 中预置的谓词表达式以及特定可用于谓词中的方法。在转换时，当遇到不支持的表达式类型、方法或找不到对应的表达式时会报错。

通过引入 Predicate 宏，Swift 提供了一种既简洁又强大的方式来构建复杂的谓词逻辑，它允许开发者以几乎原生 Swift 代码的形式直接构建出复杂的逻辑判断，显著提高了代码的可读性和可维护性。更重要的是，Predicate 宏的使用大幅减少了开发者构建复杂查询时的心智负担，使得开发工作流程更为流畅和高效。

## Swift Predicate 构建的技巧与注意事项

在了解了 Swift Predicate 的构成之后，我们可以更准确地掌握构建 Predicate 时的限制与技巧。

### 全局函数的限制

使用 Predicate 宏构建谓词时，需要注意宏的转换逻辑是将闭包代码转换为 Foundation 的预置 `PredicateExpress` 表达式。当前预置的 `PredicateExpress` 实现并不支持直接访问全局函数或类型方法或属性返回的数据。因此，在使用这类数据构建谓词时，应通过 `let` 关键字预先获取所需数据。例如：

Swift 

```
func now() -> Date {
  .now
}
let predicate = #Predicate<Date>{ $0 < now()  } // Global functions are not supported in this predicate
```

正确的方式是先获取函数或属性的值，再构建谓词：

Swift 

```
let now = now()
let predicate = #Predicate<Date>{ $0 < now  }
```

同理，对于类型属性的直接访问也存在限制：

Swift 

```
let predicate = #Predicate<Date>{ $0 < Date.now  }
// Key path cannot refer to static member 'now'

let now = Date.now
let predicate = #Predicate<Date>{ $0 < now  }
```

这是由于当前的谓词表达式仅支持实例属性的 KeyPath，并不支持类型属性。

### 实例方法的限制

与上一条相同，在谓词中直接调用实例方法（如 `.lowercased()`）也不受支持。

Swift 

```
struct A {
  var name:String
}

let predicate = #Predicate<A>{ $0.name.lowercased() == "fat" } // The lowercased() function is not supported in this predicate
```

在这种情况下，应使用 Swift Predicate 支持的内置方法，例如：

Swift 

```
let predicate = #Predicate<A>{ $0.name.localizedLowercase == "fat" }
```

目前可用的内置方法集合是相对有限的，这包括但不限于：`contains`、`allSatisfy`、`flatMap`、`filter`、`subscript`、`starts`、`min`、`max`、`localizedStandardContains`、`localizedCompare`、`caseInsensitiveCompare` 等。开发者应定期查阅苹果的 [官方文档](https://developer.apple.com/documentation/foundation/predicate?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 或直接参考 Predicate 宏的源代码，以获取对最新支持的方法的全面了解。

由于目前内置的方法并不全面，一些在 NSPredicate 中常见的谓词构建方式在 Swift Predicate 中可能尚未得到支持。这意味着，尽管 Swift Predicate 为构建类型安全且表达力强的谓词提供了强大的工具，但开发者可能仍需在某些场景下寻找替代方案或等待未来的扩展以覆盖更广泛的用例。

### 支持创建多种泛型参数的谓词

得益于 Parameter Packs 功能，Swift Predicate 为开发者提供了更高的灵活性，允许定义能够接收多种泛型参数的谓词。这种能力极大地扩展了谓词的适用场景，使得开发者能够轻松应对各种复杂的条件判断需求。

正如前文中构建的 `n < m` 示例所展示的，这种方法不仅可以应用于单一类型的参数比较，还可以扩展到多个不同类型的参数，进一步增强了 Swift Predicate 相比传统 Swift 高阶函数的表达能力和灵活性。这一特性让 Swift Predicate 成为构建复杂逻辑判断的强大工具，同时保持代码的清晰性和类型安全。

Swift 

```
struct A {
  var name:String
}

struct B {
  var age: Int
}

let predicate = #Predicate<A,B>{ a,b in
  !a.name.isEmpty && b.age > 10
}
```

### 通过嵌套机制创建复杂的判断逻辑

Swift Predicate 的设计允许开发者通过嵌套谓词表达式构建出结构复杂的谓词逻辑。这种能力使得在实现那些在 `NSPredicate` 中通常需要依赖子查询来完成的条件判断变得更加直观和简洁。如今，这些复杂的逻辑表达可以更加符合 Swift 语言的编程习惯，提高了代码的可读性和可维护性。

Swift 

```
struct Address {
  var city:String
}
struct People {
  var address:[Address]
}

let predicate = #Predicate<People>{ people in
  people.address.contains { address in
    address.city == "Dalian"
  }
}
```

> 当数据模型包含对多关系且为可选时，上述方法不起作用

### 支持构建包含可选值的谓词

Swift Predicate 支持了可选值类型的使用，这是在处理数据模型中常见的可选属性时的一大优势。这种支持允许开发者直接在谓词逻辑中处理可选值，从而使得谓词表达式的书写更加直接和清晰。

例如，以下示例展示了如何在 Swift Predicate 中处理一个可选字符串属性，根据其是否以特定前缀开始来进行过滤：

Swift 

```
let predicate = #Predicate<Note> {
  if let name = $0.name {
    return name.starts(with: "fat")
  } else {
    return false
  }
}
```

对于希望深入了解如何在 Swift Predicate 中高效处理可选值的开发者，推荐阅读[如何处理 SwiftData 谓词中的可选值](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/)。

### Swift Predicate 是线程安全的

Swift Predicate 的设计考虑到了并发编程的需求，确保了其线程安全性。通过遵循 `Sendable` 协议，Swift Predicate 支持在不同的执行上下文之间安全地传递。这一特性显著增强了 Swift Predicate 的实用性，使其能够适应现代 Swift 应用程序中对并发和异步编程的广泛需求。

### Swift Predicate 支持序列化和反序列化

通过实现 `Codable` 协议，Swift Predicate 可以被转换成 JSON 或其他格式，从而实现数据的序列化与反序列化。这一特性对于需要将谓词条件保存至数据库或配置文件，或者需要在客户端与服务器之间共享谓词逻辑的应用场景尤为重要。

以下示例展示了如何将一个 `Predicate` 实例序列化为 JSON 数据，进而可以存储或传输：

Swift 

```
struct A {
  var name:String
}

let predicate = #Predicate<A>{ $0.name == "fatbobman" }
var configuration = Predicate<A>.EncodingConfiguration.standardConfiguration
configuration.allowKeyPath(\A.name, identifier: "name")
let data = try JSONEncoder().encode(predicate, configuration: configuration)
```

### 在构建复杂谓词时，应注意其对编译时间的影响

类似于在 SwiftUI 中构建界面时遇到的情况，在构建复杂的 Swift Predicate 表达式时，Swift 编译器需要处理并转换成一个庞大且复杂的类型。这个过程中，一旦表达式的复杂度超过了某个阈值，编译器在进行类型推断的时间将显著增加。

当发现编译时长受到影响时，开发者可以考虑将复杂的谓词声明放置在独立的 Swift 文件中。这样做不仅有助于组织和管理代码，还可以在一定程度上减少因频繁修改其他部分代码而触发的重新编译。

### 尚不支持使用自定义谓词表达式构建谓词

目前，尽管开发者可以创建符合 `PredicateExpress` 协议的自定义表达式类型，但官方并不允许自定义表达式符合 `StandardPredicateExpression` 协议。因此，虽然可以创建自定义表达式类型，但在构建谓词时无法直接使用这些自定义表达式。

即使开发者将自定义表达式标注为遵循 `StandardPredicateExpression` 协议，但是 Predicate 宏目前仅支持使用 Foundation 中预置的 `StandardPredicateExpression` 实现。这一限制使得开发者无法在 Predicate 宏中使用自定义表达式，从而导致无法利用自定义表达式构建谓词。

### 尚不支持将多个谓词组合成更加复杂的谓词

在构建 `NSPredicate` 时，开发者可以通过 `NSCompoundPredicate` 将多个简单逻辑的 `NSPredicate` 灵活组合成更复杂的谓词。然而，Swift Predicate 目前尚未提供类似的能力，这在一定程度上限制了开发者构建复杂谓词的灵活性。

在之后的文章中，我将介绍如何在当前阶段通过 `PredicateExpress` 动态构建复杂的谓词，以满足特定的需求。这样的方法可能会在某些情况下提供一种替代方案，以应对当前不支持将多个谓词合并的局限性。

## 在 SwiftData 中应用 Swift Predicate

SwiftData 和 Core Data 中使用 Predicate 作为数据检索条件是许多开发者的常见场景。理解 SwiftData 对 Swift Predicate 的处理方式对于最大化其效用至关重要。

### SwiftData 与 Swift Predicate 的交互机制

当在 SwiftData 中设置 `FetchDescriptor` 的 Predicate 时，SwiftData 并不直接采用 Swift Predicate 的评估机制。相反，它通过解析 Predicate 的 `express` 属性所定义的表达式树，并将这些表达式转换成 SQL 语句，以便从 SQLite 数据库检索数据。这意味着，在 SwiftData 环境中，评估操作实际上是通过 SQL 指令从 SQLite 数据库获取数据的过程，是在数据库端进行的。

### SwiftData 对谓词参数的限制

SwiftData 要求每个 FetchDescriptor 必须对应一个具体的数据实体。因此，构建谓词时，相应的实体类型成为谓词的唯一参数，这一点对于有效利用 SwiftData 构建谓词至关重要。

### SwiftData 谓词构建的表达能力限制

虽然 Swift Predicate 提供了一个强大的框架用于数据筛选，但其在 SwiftData 环境中的表达能力相比于结合使用 NSPredicate 的 Core Data 有所限制。面对特定筛选需求时，开发者可能需要采用间接方法，例如执行多次筛选或在实体中预先添加适配当前谓词能力的特定属性。例如，由于内置的 `starts` 方法对大小写敏感，若需实现忽略大小写的匹配，推荐为筛选属性创建一个预处理版本（如全部转为小写），以支持更灵活的数据检索。

### 谓词出现运行时错误

即使 Swift Predicate 在编译时没有错误，使用 SwiftData 进行数据检索时也可能遇到无法成功转换为 SQL 语句的情况，从而导致出现运行时错误。考虑以下示例：

Swift 

```
let predicate = #Predicate<Note> { $0.id == noteID }
// Runtime error：Couldn't find \Note.id on Note with fields
```

虽然 `Note` 类型遵循 PersistentModel 协议，并且其 `id` 属性的类型也为 PersistentIdentifier，但 SwiftData 在讲谓词转换为 SQL 指令时却无法识别 `id` 属性。在这种情况下，开发者应使用 `persistentModelID` 属性进行比较（ 在进行谓词转换时，除了底层数据模型对应的属性外，`persistentModelID` 是为数不多的特别支持的属性 ）：

Swift 

```
let predicate = #Predicate<Note> { $0.persistentModelID == noteID }
```

此外，尝试在 PersistentModel 的属性上应用内置方法集时也可能遇到问题：

Swift 

```
let predicate = #Predicate<Note> {
  $0.name.localizedLowercase.starts(with: "abc".localizedLowercase)
}
// Runtime error: Couldn't find \Note.name.localizedLowercase on Note with fields
```

当 SwiftData 转换这些表达式时，很多内置方法同样也不适用于 PersistentModel 的属性，SwiftData 会错误地将其视为一个 `KeyPath`。因此，在当前阶段，开发者可能需要创建额外的属性（例如，属性的小写版本）来适应这种场景。

### 获取不到预期结果的情况

在某些情况下， Swift Predicate 能够顺利编译并在 SwiftData 环境下运行而不报错，却可能因为 SwiftData 转换了错误的 SQL 指令，导致无法检索到预期的结果。以下示例说明了这一点：

Swift 

```
let predicate = #Predicate<Item> {
  $0.note?.parent?.persistentModelID == rootNoteID
}
```

此谓词在编译和运行时都不会出现问题，但最终无法正确检索数据。为了解决这个问题，我们需要用其他的方式构建相同逻辑的谓词，确保它能够正确处理可选值，详情请见 [如何处理 SwiftData 谓词中的可选值](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/) 一文：

Swift 

```
let predicate = #Predicate<Item> {
  if let note = $0.note {
    return note.parent?.persistentModelID == rootNoteID
  } else {
    return false
  }
}
```

> 正因如此，进行全面而及时的单元测试在构建 SwiftData 谓词时显得尤为重要。通过测试，开发者可以验证谓词的行为与预期是否一致，确保数据检索的准确性和应用的稳定性。

[ 马上加入 ![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)的开发者社区!  加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长! ](https://t.ly/gzxeh) 

 🚀 马上加入，享受交流的快乐 →

## 总结

Swift Predicate 为 Swift 开发者带来了一种强大且灵活的工具，使得数据筛选和逻辑判断变得更加直观和高效。通过本文的探讨，我希望开发者不仅能够充分掌握 Swift Predicate 的强大功能和使用方法，而且能够在面对挑战和限制时，找到创造性的解决方案。

> "加入我们的 [Discord 社区](https://discord.com/invite/7FedN5E2QQ)，与超过 2000 名苹果生态的中文开发者一起交流！"

[SwiftData 中的关系：变化与注意事项](https://fatbobman.com/zh/posts/relationships-in-swiftdata-changes-and-considerations/)[CoreData 探秘 - 从数据模型构建到托管对象实例](https://fatbobman.com/zh/posts/from-data-model-construction-to-managed-object-instances-in-core-data/)[如何处理 SwiftData 谓词中的可选值](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/)[SwiftDataKit：让你在 SwiftData 中使用 Core Data 的高级功能](https://fatbobman.com/zh/posts/use-core-data-features-in-swiftdata-by-swiftdatakit/)[WWDC 2023 Core Data 有哪些新变化](https://fatbobman.com/zh/posts/what-s-new-in-core-data-in-wwdc23/)[如何为 SwiftData 动态的构建复杂的谓词](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/)

每周精选 Swift 与 SwiftUI 精华！

链接已复制

* [ 什么是谓词 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#什么是谓词)
* [ Swift Predicate 的引入与改进 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#swift-predicate-的引入与改进)
* [ NSPredicate 与 Swift Predicate 的比较 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#nspredicate-与-swift-predicate-的比较)
* [ Swift Predicate 的主要构成 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#swift-predicate-的主要构成)
* [ PredicateExpression 协议 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#predicateexpression-协议)
* [ #Expression 宏 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#expression-宏)
* [ Predicate 结构体 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#predicate-结构体)
* [ Predicate 宏 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#predicate-宏)
* [ Swift Predicate 构建的技巧与注意事项 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#swift-predicate-构建的技巧与注意事项)
* [ 全局函数的限制 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#全局函数的限制)
* [ 实例方法的限制 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#实例方法的限制)
* [ 支持创建多种泛型参数的谓词 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#支持创建多种泛型参数的谓词)
* [ 通过嵌套机制创建复杂的判断逻辑 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#通过嵌套机制创建复杂的判断逻辑)
* [ 支持构建包含可选值的谓词 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#支持构建包含可选值的谓词)
* [ Swift Predicate 是线程安全的 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#swift-predicate-是线程安全的)
* [ Swift Predicate 支持序列化和反序列化 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#swift-predicate-支持序列化和反序列化)
* [ 在构建复杂谓词时，应注意其对编译时间的影响 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#在构建复杂谓词时应注意其对编译时间的影响)
* [ 尚不支持使用自定义谓词表达式构建谓词 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#尚不支持使用自定义谓词表达式构建谓词)
* [ 尚不支持将多个谓词组合成更加复杂的谓词 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#尚不支持将多个谓词组合成更加复杂的谓词)
* [ 在 SwiftData 中应用 Swift Predicate ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#在-swiftdata-中应用-swift-predicate)
* [ SwiftData 与 Swift Predicate 的交互机制 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#swiftdata-与-swift-predicate-的交互机制)
* [ SwiftData 对谓词参数的限制 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#swiftdata-对谓词参数的限制)
* [ SwiftData 谓词构建的表达能力限制 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#swiftdata-谓词构建的表达能力限制)
* [ 谓词出现运行时错误 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#谓词出现运行时错误)
* [ 获取不到预期结果的情况 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#获取不到预期结果的情况)
* [ 总结 ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#总结)

如何处理 SwiftData 谓词中的可选值

⌥ + ←

如何为 SwiftData 动态的构建复杂的谓词

⌥ + →

[ ](https://x.com/intent/tweet?text=Swift%20Predicate%3A%20%E7%94%A8%E6%B3%95%E3%80%81%E6%9E%84%E6%88%90%E5%8F%8A%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9&url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fswift-predicate-usage-composition-and-considerations%2F&hashtags=ios%2Cswiftlang&via=fatbobman) 

Share on X

Share on BlueSky

[ ](https://www.facebook.com/sharer/sharer.php?u=https%253A%252F%252Ffatbobman.com%252Fzh%252Fposts%252Fswift-predicate-usage-composition-and-considerations%252F) 

Share on Facebook

[ ](https://service.weibo.com/share/share.php?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fswift-predicate-usage-composition-and-considerations%2F&title=Swift%20Predicate%3A%20%E7%94%A8%E6%B3%95%E3%80%81%E6%9E%84%E6%88%90%E5%8F%8A%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9%0A%20%23ios%20%23swift%0A%20by%20%40fatbobman%0A) 

Share on Weibo

[ ](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fswift-predicate-usage-composition-and-considerations%2F) 

Share on Linkedin

[ ](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/mailto:?subject=%E6%96%87%E7%AB%A0%3A%E3%80%90Swift%20Predicate%3A%20%E7%94%A8%E6%B3%95%E3%80%81%E6%9E%84%E6%88%90%E5%8F%8A%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9%E3%80%91&body=%E5%88%86%E4%BA%AB%E4%B8%80%E7%AF%87%E6%96%87%E7%AB%A0%E7%BB%99%E4%BD%A0%EF%BC%9A%0A%0ASwift%20Predicate%3A%20%E7%94%A8%E6%B3%95%E3%80%81%E6%9E%84%E6%88%90%E5%8F%8A%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9%0A%0A%E9%93%BE%E6%8E%A5%3A%20https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fswift-predicate-usage-composition-and-considerations%2F) 

Share via Email

Share Your Thoughts

Back to Top