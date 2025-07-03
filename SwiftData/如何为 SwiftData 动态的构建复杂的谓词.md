[ #SwiftData ](https://fatbobman.com/zh/tags/SwiftData/)[ #Swift ](https://fatbobman.com/zh/tags/Swift/) [ #English ](https://fatbobman.com/en/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/) 

# 如何为 SwiftData 动态的构建复杂的谓词

[东坡肘子](https://x.com/intent/follow?screen%5Fname=fatbobman) 

 发表于 2024 年 3 月 7 日 

`NSCompoundPredicate` 让开发者能够将多个 `NSPredicate` 对象组合成一个复合谓词。这一机制特别适用于那些需要基于多重判断标准进行数据过滤的场景。然而，在 Swift 重构的新 Foundation 框架中，缺失了与 `NSCompoundPredicate` 相对应的直接功能，这一变化对希望利用 SwiftData 构建应用的开发者造成了不小的挑战。本文旨在探索如何在当前的技术条件下，利用 `PredicateExpression`，动态地构建出符合 SwiftData 需求的复杂谓词。

> 2024 年 3 月 11 日更新：在本文发布不到一周的时间里，Noah Kamara 已经开发出了一个与 SwiftData 兼容的谓词合并方案 —— [CompoundPredicate](https://github.com/NoahKamara/CompoundPredicate?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web)。我在文章末尾提供了对此方案的介绍和说明。
> 
> 2024 年 6 月更新，新增适用于 iOS 17.4+ / macOS 14.4+ 系统的官方谓词合成方法。

[ 马上加入 ![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)的开发者社区!  加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长! ](https://t.ly/gzxeh) 

 🚀 马上加入，享受交流的快乐 →

## 挑战：实现灵活的数据筛选功能

在开发新版的 [健康笔记](https://fatbobman.com/healthnotes/) 应用时，我决定使用 SwiftData 替换传统的 Core Data，以便利用 Swift 语言的现代特性。这个以数据管理为中心的应用的核心之一是提供给用户一个灵活且强大的数据筛选功能。在这个过程中，我面临着一个关键的挑战：如何为用户检索数据构建多样化的筛选方案。这里是一些用于检索 Memo 实例的谓词：

Swift 

```
extension Memo {
  public static func predicateFor(_ filter: MemoPredicate) -> Predicate<Memo>? {
    var result: Predicate<Memo>?
    switch filter {
    case .filterAllMemo:
      // nil
      break
    case .filterAllGlobalMemo:
      result = #Predicate<Memo> { $0.itemData == nil }
    case let .filterAllMemoOfRootNote(noteID):
      result = #Predicate<Memo> {
        if let itemData = $0.itemData, let item = itemData.item, let note = item.note {
          return note.persistentModelID == noteID || note.parent?.persistentModelID == noteID
        } else {
          return false
        }
      }
    case .filterMemoWithImage:
      result = #Predicate<Memo> { $0.hasImages }
    case .filterMemoWithStar:
      result = #Predicate<Memo> { $0.star }
    case let .filterMemoContainsKeyword(keyword):
      result = #Predicate<Memo> {
        if let content = $0.content {
          return content.localizedStandardContains(keyword)
        } else {
          return false
        }
      }
    }
    return result
  }
}
```

在应用的早期版本中，用户能够灵活组合筛选条件，比如结合是否包含星标与图片，或是按特定笔记和关键词进行筛选。过去，这种动态组合的需求可以通过 `NSCompoundPredicate` 轻松实现，它允许开发者动态地组合多个谓词，并将结果用作 Core Data 的检索条件。然而，使用 SwiftData 后，我发现缺少了相应的功能来动态地组合 Swift Predicate，这对于应用的核心功能是一个重大的制约。解决这一问题对于保持应用的功能性和用户满意度至关重要。

## 组合 NSPredicate 的方法

`NSCompoundPredicate` 提供了一种强大的方式，使开发者能够动态地将多个 `NSPredicate` 实例组合成一个复合谓词。以下是一个示例，演示了如何使用 `AND` 逻辑操作符将两个独立的谓词 `a` 和 `b` 组合成一个新的谓词：

Swift 

```
let a = NSPredicate(format: "name = %@", "fat")
let b = NSPredicate(format: "age < %d", 100)
let result = NSCompoundPredicate(type: .and, subpredicates: [a,b])
```

此外，由于 `NSPredicate` 允许通过字符串来构建谓词，开发者可以利用这一特性通过组合 `predicateFormat` 属性来手动构建新的谓词表达式。这种方法提供了额外的灵活性，允许开发者直接操作和组合已存在谓词的字符串表示形式：

Swift 

```
let a = NSPredicate(format: "name = %@", "fat")
let b = NSPredicate(format: "age < %d", 100)
let andFormatString = a.predicateFormat + " AND " + b.predicateFormat // name == "fat" AND age < 100
let result = NSPredicate(format: andFormatString)
```

不幸的是，尽管这些方法在使用 `NSPredicate` 时非常有效和灵活，但它们不适用于 Swift Predicate。这意味着在转向使用 SwiftData 时，我们需要探索新的方法来实现类似的动态谓词组合功能。

## 组合 Swift Predicate 的挑战

在之前的文章“[Swift Predicate: 用法、构成及注意事项](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/)”中，我们详细探讨了 Swift Predicate 的结构和构成。简而言之，开发者通过声明满足 `PredicateExpression` 协议的类型，进而构建 `Predicate` 结构体。由于这个过程可能相当复杂，Foundation 为此提供了 `#Predicate` 宏，以简化这一操作。

当我们构建 Swift Predicate 时，`#Predicate` 宏会自动转换这些运算符为相应的谓词表达式：

Swift 

```
let predicate = #Predicate<People> { $0.name == "fat" && $0.age < 100 }
```

宏展开后，我们可以看到谓词表达式的详细构成：

Swift 

```
Foundation.Predicate<People>({
    PredicateExpressions.build_Conjunction(
        lhs: PredicateExpressions.build_Equal(
            lhs: PredicateExpressions.build_KeyPath(
                root: PredicateExpressions.build_Arg($0),
                keyPath: \.name
            ),
            rhs: PredicateExpressions.build_Arg("fat")
        ),
        rhs: PredicateExpressions.build_Comparison(
            lhs: PredicateExpressions.build_KeyPath(
                root: PredicateExpressions.build_Arg($0),
                keyPath: \.age
            ),
            rhs: PredicateExpressions.build_Arg(100),
            op: .lessThan
        )
    )
})
```

在这里，`PredicateExpressions.build_Conjunction` 创建一个对应 `&&` 操作符的 `PredicateExpressions.Conjunction` 表达式。它将两个布尔返回值的表达式连接，形成一个完整的表达式。如果我们能够单独提取并组合 Swift Predicate 中的表达式，理论上就可以动态组合基于 `AND` 逻辑的谓词。

> `||` 和 `!` 对应的表达式类型分别是 `PredicateExpressions.Disjunction` 和 `PredicateExpressions.Negation`。

考虑到 Swift Predicate 提供了一个 `expression` 属性，自然地，我们会考虑利用该属性来实现这种动态组合：

Swift 

```
let a = #Predicate<People1> { $0.name == "fat"}
let b = #Predicate<People1> { $0.age < 10 }
let combineExpression = PredicateExpressions.build_Conjunction(lhs: a.expression, rhs: b.expression)
```

然而，尝试上述代码会遇到编译错误：

Swift 

```
Type 'any StandardPredicateExpression<Bool>' cannot conform to 'PredicateExpression'
```

深入探究 `Predicate` 结构体和 `PredicateExpressions.Conjunction` 的实现，我们发现了其中的制约因素：

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

extension PredicateExpressions {
    public struct Conjunction<
        LHS : PredicateExpression,
        RHS : PredicateExpression
    > : PredicateExpression
    where
        LHS.Output == Bool,
        RHS.Output == Bool
    {
        public typealias Output = Bool
        
        public let lhs: LHS
        public let rhs: RHS
        
        public init(lhs: LHS, rhs: RHS) {
            self.lhs = lhs
            self.rhs = rhs
        }
        
        public func evaluate(_ bindings: PredicateBindings) throws -> Bool {
            try lhs.evaluate(bindings) && rhs.evaluate(bindings)
        }
    }
    
    public static func build_Conjunction<LHS, RHS>(lhs: LHS, rhs: RHS) -> Conjunction<LHS, RHS> {
        Conjunction(lhs: lhs, rhs: rhs)
    }
}
```

问题在于 `expression` 属性类型为 `any StandardPredicateExpression<Bool>`，它不包含足够的信息来标识具体的 `PredicateExpression` 实现类型。由于 `Conjunction` 需要具体的左右子表达式类型来初始化，因此我们就无法直接使用 `expression` 属性来动态构建新的组合表达式。

## 动态构建 Predicate 的策略

虽然无法直接利用 Swift Predicate 的 `expression` 属性，我们仍有其他途径来实现动态构建谓词的目标。关键在于理解如何从现有的谓词中提取或独立创建表达式，并利用如 `build_Conjunction` 或 `build_Disjunction` 这样的表达式构建器来生成新的谓词表达式。

### 利用 #Predicate 宏构建表达式

直接基于表达式类型构建谓词可能非常繁琐。一个更实用的方法是利用 `#Predicate` 宏，这样开发者可以间接地构建和提取谓词表达式。这种方法的灵感来源于社区成员 [nOk](https://stackoverflow.com/users/11450810/nok?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 在 [stackoverflow](https://stackoverflow.com/a/76861465/12260342?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 的贡献。

例如，我们有以下使用 `#Predicate` 宏构建的谓词：

Swift 

```
let filterByName = #Predicate<People> { $0.name == "fat" }
```

通过查看宏展开后的代码，我们可以提取出构成谓词的表达式部分：

![](https://cdn.fatbobman.com/image-20240304161732812-zipic.png) 

由于在构建表达式（PredicateExpression）实例时，还要根据表达式的类型为其提供不同的参数，因此，使用下面的方式无法生成正确的表达式：

Swift 

```
let expression = PredicateExpressions.build_Equal(
  lhs: PredicateExpressions.build_KeyPath(
      root: PredicateExpressions.build_Arg($0), // error: Anonymous closure argument not contained in a closure
      keyPath: \.name
  ),
  rhs: PredicateExpressions.build_Arg("fat")
)
```

尽管我们不能直接复制表达式来创建新的 `PredicateExpression` 实例，但我们可以通过定义一个闭包来重新创建相同的表达式：

Swift 

```
let expression = { (people: PredicateExpressions.Variable<People>) in
  PredicateExpressions.build_Equal(
      lhs: PredicateExpressions.build_KeyPath(
          root: PredicateExpressions.build_Arg(people),
          keyPath: \.name
      ),
      rhs: PredicateExpressions.build_Arg("fat")
  )
}
```

### 创建参数化的表达式闭包

由于表达式的右侧值（如 `"fat"`）可能需要动态赋值，我们可以设计一个返回表达式闭包的闭包，这样可以在运行时确定要比较的名字：

Swift 

```
let filterByNameExpression = { (name: String) in
  { (people: PredicateExpressions.Variable<People>) in
    PredicateExpressions.build_Equal(
      lhs: PredicateExpressions.build_KeyPath(
        root: PredicateExpressions.build_Arg(people),
        keyPath: \.name
      ),
      rhs: PredicateExpressions.build_Arg(name)
    )
  }
}
```

使用这个返回表达式的闭包，我们可以动态构建谓词：

Swift 

```
let name = "fat"
let predicate = Predicate<People>(filterByNameExpression(name))
```

### 组合表达式以构建新的谓词

一旦我们声明了返回表达式的闭包，就可以结合使用 `PredicateExpressions.build_Conjunction` 或其他逻辑构建器来创建包含复杂逻辑的新谓词：

Swift 

```
// #Predicate<People> { $0.age < 10 }
let filterByAgeExpression = { (age: Int) in
  { (people: PredicateExpressions.Variable<People>) in
    PredicateExpressions.build_Comparison(
        lhs: PredicateExpressions.build_KeyPath(
            root: PredicateExpressions.build_Arg(people),
            keyPath: \.age
        ),
        rhs: PredicateExpressions.build_Arg(age),
        op: .lessThan
    )
  }
}

// Combine new Predicate
let predicate = Predicate<People> {
  PredicateExpressions.Conjunction(
    lhs: filterByNameExpression(name)($0),
    rhs: filterByAgeExpression(age)($0)
  )
}
```

完整流程如下：

1. 使用 `#Predicate` 宏构建初始谓词。
2. 从宏展开的代码中提取表达式，创建生成表达式的闭包。
3. 结合布尔逻辑表达式（例如 `Conjunction`）将两个表达式组合成一个新的表达式，进而构建新的谓词实例。
4. 如需组合多个表达式，重复以上步骤。

这种方法虽然需要一些额外的步骤来手动创建和组合表达式，但它为动态构建复杂的 Swift Predicate 提供了可能。

### 动态组合表达式

掌握了从谓词到表达式，再从表达式回到谓词的完整流程后，我现在需要创建一个方法，该方法能够根据需求动态组合表达式并生成谓词，以满足我当前项目的具体需求。

参考了 Jeremy Schonfeld 在 [Swift Forums](https://forums.swift.org/t/pitch-swift-predicates/62000/50?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 提供的示例，我们可以构建一个动态合成用于检索 Memo 数据的谓词的方法，如下所示：

Swift 

```
extension Memo {
  static func combinePredicate(_ filters: [MemoPredicate]) -> Predicate<Memo> {
    func buildConjunction(lhs: some StandardPredicateExpression<Bool>, rhs: some StandardPredicateExpression<Bool>) -> any StandardPredicateExpression<Bool> {
      PredicateExpressions.Conjunction(lhs: lhs, rhs: rhs)
    }

    return Predicate<Memo>({ memo in
      var conditions: [any StandardPredicateExpression<Bool>] = []
      for filter in filters {
        switch filter {
        case .filterAllMemo:
          conditions.append(Self.Expressions.allMemos(memo))
        case .filterAllGlobalMemo:
          conditions.append(Self.Expressions.allGlobalMemos(memo))
        case let .filterAllMemoOfRootNote(noteID):
          conditions.append(Self.Expressions.memosOfRootNote(noteID)(memo))
        case .filterMemoWithImage:
          conditions.append(Self.Expressions.memoWithImage(memo))
        case .filterMemoWithStar:
          conditions.append(Self.Expressions.memosWithStar(memo))
        case let .filterMemoContainsKeyword(keyword):
          conditions.append(Self.Expressions.memosContainersKeyword(keyword)(memo))
        }
      }
      guard let first = conditions.first else {
        return PredicateExpressions.Value(true)
      }

      let closure: (any StandardPredicateExpression<Bool>, any StandardPredicateExpression<Bool>) -> any StandardPredicateExpression<Bool> = {
        buildConjunction(lhs: $0, rhs: $1)
      }

      return conditions.dropFirst().reduce(first, closure)
    })
  }
}
```

使用示例：

Swift 

```
let predicate = Memo.combinePredicate([.filterMemoWithImage,.filterMemoContainsKeyword(keyword: "fat")])
```

在当前的实现中，由于 Swift 的强类型系统（每种筛选逻辑对应着特定的谓词表达式类型），构建一个与 `NSCompoundPredicate` 相似的灵活且通用的组合机制显得相对复杂。我们面临的挑战在于如何在保持类型安全的同时，实现足够灵活的表达式组合策略。

对于我的应用场景，主要需求是处理 `Conjunction`（逻辑与）类型的组合，这种情况相对简单直接。如果未来的需求扩展到包括 `Disjunction`（逻辑或）的情况，我们就必须在组合过程中引入额外的逻辑判断和标识，以确保可以灵活地应对不同的逻辑组合需求，同时保持代码的可读性和可维护性。这可能需要更细致的设计，以适应多变的组合逻辑，同时确保不牺牲 Swift 的类型安全特性。

> [在此](https://gist.github.com/fatbobman/8e10c7a811c6cd384b943d1a25206b7b?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web)可查看完整代码。

## 一个无法应用于 SwiftData 的组合实现

> Noah Kamara 提供的更新后的解决方案，请参阅下一节内容。

Noah Kamara 在其 [Gist](https://gist.github.com/NoahKamara/d8660881b2ef8d6be18b8e26ed349bb7?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 中展示了一段代码，提供了类似于 `NSCompoundPredicate` 能力，使得组合 Swift Predicate 变得简便快捷。这种方法看起来是一个直观且强大的解决方案：

Swift 

```
let people = People(name: "fat", age: 50)
let filterByName = #Predicate<People> { $0.name == "fat" }
let filterByAge = #Predicate<People> { $0.age < 10 }
let combinedPredicate = [filterByName, filterByAge].conjunction()
try XCTAssertFalse(combinedPredicate.evaluate(people)) // return false
```

尽管这种方法极具吸引力，我们却不能在 SwiftData 中采用。为什么这种表面上完美的解决方案会在 SwiftData 中遇到障碍呢？

Noah Kamara 在代码中引入了一个名为 `VariableWrappingExpression` 的自定义类型，这个类型实现了 `StandardPredicateExpression` 协议，用以封装从 Predicate 的 `expression` 属性中提取的表达式。这种封装方法并不涉及到表达式的具体类型，它在评估谓词时仅调用被封装表达式的评估方法。

Swift 

```
struct VariableWrappingExpression<T>: StandardPredicateExpression {
    let predicate: Predicate<T>
    let variable: PredicateExpressions.Variable<T>
    
    func evaluate(_ bindings: PredicateBindings) throws -> Bool {
        // resolve the variable
        let value = try variable.evaluate(bindings)
        
        // create bindings for the expression of the predicate
        let innerBindings = bindings.binding(predicate.variable, to: value)
        
        return try predicate.expression.evaluate(innerBindings)
    }
}
```

在非 SwiftData 环境下，这种动态组合的谓词可以正常工作，因为它直接依赖于 Swift Predicate 的评估逻辑。然而，SwiftData 的工作机制有所不同。在使用 SwiftData 筛选数据时，并不会直接调用 Swift Predicate 的评估方法。相反，SwiftData 会解析 Predicate 的 `expression` 属性中表达式树，并将这些表达式转换为 SQL 语句，以便在 SQLite 数据库中执行数据检索。这意味着评估过程实际上是通过生成并执行 SQL 指令来完成的，完全在数据库层面上操作。

因此，当 SwiftData 尝试将这种动态组合的谓词转换为 SQL 指令时，由于无法识别自定义的 `VariableWrappingExpression` 类型，就会导致出现 `unSupport Predicate` 的运行时错误。

如果你的场景不涉及在 SwiftData 中使用谓词，那么 Noah Kamara 的方案可能是一个很好的选择。如果你的需求是在 SwiftData 环境中构建动态组合的谓词，可能仍需要依赖于本文介绍的策略。

## CompoundPredicate：可用于 SwiftData 的谓词合并方案

在之前的讨论中，我们揭示了一个主要障碍，即开发者无法直接利用 Predicate 的 `expression` 属性来合成复杂谓词，因为缺少了具体的谓词表达式类型。本文发布几天后，Noah Kamara 推出了一个创新的方案—— [CompoundPredicate](https://github.com/NoahKamara/CompoundPredicate/?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web)，这一新策略放弃了之前依赖自定义 `StandardPredicateExpression` 的实现方法，转而采用类型转换的策略，有效地将 `expression` 的信息具体化。这一改进意味着开发者可以避免手动提取和组合表达式的繁琐过程。

Noah Kamara 设计了 `VariableReplacing` 协议，并为 PredicateExpression 的每种类型实现了这一协议，如下所示：

Swift 

```
public protocol VariableReplacing<Output>: PredicateExpression {
    associatedtype Output
    typealias Variable<T> = PredicateExpressions.Variable<T>
    func replacing<T>(_ variable: Variable<T>, with replacement: Variable<T>) -> Self
}

extension PredicateExpressions.Variable: VariableReplacing {
    public func replacing<T>(_ variable: Variable<T>, with replacement: Variable<T>) -> Self {
        if let replacement = replacement as? Self, variable.key == key {
            return replacement
        } else {
            return self
        }
    }
}
```

这种方法使得在谓词组合过程中能自动获得 Predicate 内部表达式的确切类型，实现了谓词组合的自动化和高效化。

Swift 

```
let notTooShort = #Predicate<Book> {
    $0.pages > 50
}

let notTooLong = #Predicate<Book> {
    $0.pages <= 350
}

let lengthFilter = [notTooShort, notTooLong].conjunction()

// Match Books that are just the right length
let titleFilter = #Predicate<Book> {
    $0.title.contains("Swift")
}

// Match Books that contain "Swift" in the title or
// are just the right length
let filter = [lengthFilter, titleFilter].disjunction()
```

重要的是，这种方案不会引入 SwiftData 不能识别的表达式，因此合并后的谓词能够直接被 SwiftData 使用。这一方案可谓是在官方发布更加完善的解决方案前，为开发者提供的一种最理想的临时策略。

## 优化 Swift Predicate 表达式的编译效率

构建复杂的 Swift Predicate 表达式可能会对编译效率产生显著影响。Swift 编译器在处理这些表达式时，需要解析并生成复杂的类型信息。当表达式过于复杂时，编译器所需的类型推断时间可能会急剧增加，导致编译过程变慢。

考虑以下谓词示例：

Swift 

```
let result = #Predicate<Memo> {
  if let itemData = $0.itemData, let item = itemData.item, let note = item.note {
    return note.persistentModelID == noteID || note.parent?.persistentModelID == noteID
  } else {
    return false
  }
}
```

在这个例子中，即使是微小的代码更改也可能导致该文件的编译时间超过 10 秒。这种延迟在使用闭包生成表达式时也会出现。为了缓解这一问题，我们可以利用 Xcode 的辅助功能来明确表达式的类型。在闭包上使用 Option + 点击，可以揭示表达式的确切类型，进而允许我们为闭包的返回值提供明确的类型注解。

![](https://cdn.fatbobman.com/image-20240304183427052-zipic.png) 

Swift 

```
let memosWithStar = { (memo: PredicateExpressions.Variable<Memo>) -> PredicateExpressions.KeyPath<PredicateExpressions.Variable<Memo>, Bool> in
  PredicateExpressions.build_KeyPath(
    root: PredicateExpressions.build_Arg(memo),
    keyPath: \.star
  )
}
```

上述复杂谓词中的表达式具体类型如下所示：

![](https://cdn.fatbobman.com/image-20240304202411294-zipic.png) 

明确表达式的类型可以帮助编译器更快地处理代码，能显著提高整体的编译效率，因为它避免了编译器在类型推断上所花费的时间。

> 本策略不仅适用于需要组合谓词的情况，即使是不涉及谓词组合但逻辑复杂的谓词也会面临同样的编译挑战。通过提取表达式、明确标注类型，开发者可以显著改善复杂谓词的编译时间，确保更高效的开发体验。

## iOS 17.4+ / macOS 14.4

从 iOS 17.4 开始，开发者可以用如下方法来动态 [合成](https://github.com/apple/swift-foundation/pull/343?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 谓词：

Swift 

```
@Model
final class People {
  var age: Int = 9

  init(age: Int) {
    self.age = age
  }
}

let minAge = 10
let notTooYong = #Predicate<People> { item in
  item.age >= minAge
}

let maxAge = 50
let notTooOld = #Predicate<People> { item in
  item.age <= maxAge
}

let enableMin = true // enable minAge filter
let enableMax = false // enable maxAge filter

let filterByAge = #Predicate<People> { item in
  (enableMin ? notTooYong.evaluate(item) : true) // >= minAge
    && (enableMax ? notTooOld.evaluate(item) : true) // <= maxAge
}
```

该方法需要事先将所有的条件都列出来。在构成复杂谓词时，没有 [CompoundPredicate](https://github.com/NoahKamara/CompoundPredicate/?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 灵活。

[ 马上加入 ![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)的开发者社区!  加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长! ](https://t.ly/gzxeh) 

 🚀 马上加入，享受交流的快乐 →

## 总结

本文探讨了在 SwiftData 环境中动态构建复杂谓词的方法。虽然当前的解决方案可能不如我们期望的那样优雅直接，但它们确实提供了一种可行的方式，使得依赖 SwiftData 的应用能够实现灵活的数据查询功能，而不会因缺少某些特性而受限。

尽管我们已经找到了在当前技术限制下工作的方法，但仍然希望未来的 Foundation 和 SwiftData 版本能够提供内置的支持，使得构建动态复杂谓词变得更加简单和直观。完善这些功能将进一步增强 Swift Predicate 和 SwiftData 的实用性，让开发者能够更高效地实现复杂的数据处理逻辑。

> "加入我们的 [Discord 社区](https://discord.com/invite/7FedN5E2QQ)，与超过 2000 名苹果生态的中文开发者一起交流！"

[如何通过 Persistent History Tracking 观察 SwiftData 的数据变化](https://fatbobman.com/zh/posts/persistent-history-tracking-in-swiftdata/)[CoreData 探秘 - 从数据模型构建到托管对象实例](https://fatbobman.com/zh/posts/from-data-model-construction-to-managed-object-instances-in-core-data/)[SwiftDataKit：让你在 SwiftData 中使用 Core Data 的高级功能](https://fatbobman.com/zh/posts/use-core-data-features-in-swiftdata-by-swiftdatakit/)[如何处理 SwiftData 谓词中的可选值](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/)[SwiftData 中的并发编程](https://fatbobman.com/zh/posts/concurret-programming-in-swiftdata/)[Swift Predicate: 用法、构成及注意事项](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/)

每周精选 Swift 与 SwiftUI 精华！

链接已复制

* [ 挑战：实现灵活的数据筛选功能 ](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/#挑战实现灵活的数据筛选功能)
* [ 组合 NSPredicate 的方法 ](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/#组合-nspredicate-的方法)
* [ 组合 Swift Predicate 的挑战 ](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/#组合-swift-predicate-的挑战)
* [ 动态构建 Predicate 的策略 ](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/#动态构建-predicate-的策略)
* [ 利用 #Predicate 宏构建表达式 ](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/#利用-predicate-宏构建表达式)
* [ 创建参数化的表达式闭包 ](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/#创建参数化的表达式闭包)
* [ 组合表达式以构建新的谓词 ](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/#组合表达式以构建新的谓词)
* [ 动态组合表达式 ](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/#动态组合表达式)
* [ 一个无法应用于 SwiftData 的组合实现 ](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/#一个无法应用于-swiftdata-的组合实现)
* [ CompoundPredicate：可用于 SwiftData 的谓词合并方案 ](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/#compoundpredicate可用于-swiftdata-的谓词合并方案)
* [ 优化 Swift Predicate 表达式的编译效率 ](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/#优化-swift-predicate-表达式的编译效率)
* [ iOS 17.4+ / macOS 14.4 ](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/#ios-174--macos-144)
* [ 总结 ](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/#总结)

Swift Predicate: 用法、构成及注意事项

⌥ + ←

几个在 SwiftUI 中使用惰性容器的技巧和注意事项

⌥ + →

[ ](https://x.com/intent/tweet?text=%E5%A6%82%E4%BD%95%E4%B8%BA%20SwiftData%20%E5%8A%A8%E6%80%81%E7%9A%84%E6%9E%84%E5%BB%BA%E5%A4%8D%E6%9D%82%E7%9A%84%E8%B0%93%E8%AF%8D&url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fhow-to-dynamically-construct-complex-predicates-for-swiftdata%2F&hashtags=ios%2Cswiftlang&via=fatbobman) 

Share on X

Share on BlueSky

[ ](https://www.facebook.com/sharer/sharer.php?u=https%253A%252F%252Ffatbobman.com%252Fzh%252Fposts%252Fhow-to-dynamically-construct-complex-predicates-for-swiftdata%252F) 

Share on Facebook

[ ](https://service.weibo.com/share/share.php?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fhow-to-dynamically-construct-complex-predicates-for-swiftdata%2F&title=%E5%A6%82%E4%BD%95%E4%B8%BA%20SwiftData%20%E5%8A%A8%E6%80%81%E7%9A%84%E6%9E%84%E5%BB%BA%E5%A4%8D%E6%9D%82%E7%9A%84%E8%B0%93%E8%AF%8D%0A%20%23ios%20%23swift%0A%20by%20%40fatbobman%0A) 

Share on Weibo

[ ](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fhow-to-dynamically-construct-complex-predicates-for-swiftdata%2F) 

Share on Linkedin

[ ](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/mailto:?subject=%E6%96%87%E7%AB%A0%3A%E3%80%90%E5%A6%82%E4%BD%95%E4%B8%BA%20SwiftData%20%E5%8A%A8%E6%80%81%E7%9A%84%E6%9E%84%E5%BB%BA%E5%A4%8D%E6%9D%82%E7%9A%84%E8%B0%93%E8%AF%8D%E3%80%91&body=%E5%88%86%E4%BA%AB%E4%B8%80%E7%AF%87%E6%96%87%E7%AB%A0%E7%BB%99%E4%BD%A0%EF%BC%9A%0A%0A%E5%A6%82%E4%BD%95%E4%B8%BA%20SwiftData%20%E5%8A%A8%E6%80%81%E7%9A%84%E6%9E%84%E5%BB%BA%E5%A4%8D%E6%9D%82%E7%9A%84%E8%B0%93%E8%AF%8D%0A%0A%E9%93%BE%E6%8E%A5%3A%20https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fhow-to-dynamically-construct-complex-predicates-for-swiftdata%2F) 

Share via Email

Share Your Thoughts

Back to Top