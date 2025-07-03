[ #SwiftData ](https://fatbobman.com/zh/tags/SwiftData/)[ #Core Data ](https://fatbobman.com/zh/tags/Core-Data/) [ #English ](https://fatbobman.com/en/posts/building-typesafe-highperformance-swiftdata-core-data-models/) 

# 构建类型安全、高效的 SwiftData/Core Data 模型

[东坡肘子](https://x.com/intent/follow?screen%5Fname=fatbobman) 

 发表于 2025 年 4 月 23 日 

Swift 强大的类型系统使我们能够创建语义明确且安全的数据模型。然而，当面对 SwiftData 或 Core Data 时，我们常因底层存储机制的限制，而不得不在类型表达上做出妥协。这种妥协不仅模糊了领域模型的本意，也为应用的稳定性埋下隐患。

本文将探索如何在数据持久化的约束下，通过巧妙的类型封装和转换，构建兼具类型安全、语义明确与高效性能的数据模型。

[ 马上加入 ![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)的开发者社区!  加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长! ](https://t.ly/gzxeh) 

 🚀 马上加入，享受交流的快乐 →

## Double 与 Double?：在类型安全与存储机制间架设桥梁

可选类型（Optional）是 Swift 语言中的一颗明珠，它将「不存在」转化为了一种明确的类型表达。这一特性在数据建模中尤为重要，因为它允许我们准确地区分「值为零」和「没有值」这两种截然不同的语义状态。

在 SwiftData 中，我们可以轻松地声明可选数值属性，使模型能够自然地表达业务语义：

Swift 

```
@Model
class People {
  var name: String = ""
  var weight: Double? // 完美表达"可能没有体重记录"的语义
}
```

然而，当我们转向 Core Data 时，这种简洁的声明方式却遇到了阻碍。即使在模型编辑器中将 `weight` 标记为可选（Optional），对应的模型属性依然无法直接声明为 `Double?`——这一限制甚至在手动编写模型代码时仍然存在。

![](https://cdn.fatbobman.com/image-20250421075441668.webp) 

![](https://cdn.fatbobman.com/image-20250421075512832.webp) 

这一现象源于 Core Data 的底层实现机制：所有存储在 SQLite 中的数值类型（Float、Int 等）都会被统一转换为 `NSNumber` 类型。由于 Swift 的基础类型（Double、Float、Int）与 `NSNumber` 并非一一对应的关系，在 Core Data 的上下文中，只有 `NSNumber` 类型才能被声明为可选：

Swift 

```
extension People {
    @NSManaged public var weight: NSNumber? // 符合 Core Data 要求但不符合 Swift 风格
}
```

为了兼顾 Core Data 的存储机制与 Swift 的类型安全特性，我们可以采用属性包装的策略。首先，将实体中的原始属性命名为 `weightRaw`，然后通过计算属性和访问权限控制，为调用者提供符合 Swift 惯用法的 API：

![](https://cdn.fatbobman.com/image-20250421081620787.webp) 

Swift 

```
extension People {
    @NSManaged public var name: String
  
    // 对外暴露符合 Swift 风格的类型安全 API
    public var weight: Double? {
        get { weightRaw?.doubleValue }
        set {
            weightRaw = newValue as NSNumber?
        }
    }
}

extension People {
    // 内部属性，对调用者隐藏实现细节
    @NSManaged var weightRaw: NSNumber?
}
```

细心的读者可能已经注意到，如果直接使用模型编辑器生成的代码，即使我们将 `name` 设置为非可选，其类型依然会被声明为 `String?`。这就是为什么在手动编写模型代码时，我们需要明确地将其声明为 `String` 类型。

> SwiftData 之所以能够直接支持 `Double?` 这样的声明，是因为它在模型定义中巧妙地实现了外部属性与底层存储（backingData）之间的类型转换——这与我们上面手动实现的方案如出一辙，也是 SwiftData 相较于 Core Data 的一项显著改进。

通过这种方式，我们既满足了底层存储的技术要求，又为上层代码提供了类型安全、语义明确的 API，真正实现了“鱼和熊掌兼得”。

## String 与 NonEmpty<String>：让类型系统为数据合法性背书

在 `People` 模型中，虽然我们已经将 `name` 调整为非可选类型，但这种简单的 `String` 类型并不能充分表达名称的业务约束和有效性规则。空字符串(`""`)或仅包含空白字符的字符串(`" "`)虽然满足”非可选”的技术要求，但显然违背了“有效名称”的业务语义。

此时，我们需要一种能够在类型层面确保字符串非空的方案。[Point-Free](https://www.pointfree.co?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 的 [NonEmpty](https://github.com/pointfreeco/swift-nonempty?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 库提供了这样一种优雅解决方案：

![](https://cdn.fatbobman.com/image-20250421083939936.webp) 

Swift 

```
import NonEmpty

extension People {
    public var name: NonEmptyString {
        get { NonEmpty(stringLiteral: nameRaw) }
        set { nameRaw = newValue.rawValue }
    }
}

extension People {
    @NSManaged var nameRaw: String
}
```

通过这种方式，我们不仅在语义上表达了“名称必须非空”的业务约束，更重要的是，这种约束被编码进了类型系统，使得编译器能够在编译时捕获潜在的错误。

然而，在实际业务中，对名称的要求往往不止于“非空”。可能还包括长度限制、字符集限制等多种约束。这时，我们可以通过自定义类型来封装更复杂的验证逻辑：

Swift 

```
/// 验证后的名称结构体，确保名称符合长度要求
public struct ValidatedName {
    /// 笔记名称最小长度
    public static let lengthLowerBound = 3
    /// 笔记名称最大长度
    public static let lengthUpperBound = 20
    /// 笔记名称的原始字符串值
    public let rawValue: String

    /// 创建并验证名称
    public init(_ name: String) throws(NameError) {
        let cleanName = try Self.validate(name)
        rawValue = cleanName
    }

    /// 验证名称是否符合规则
    static func validate(_ name: String) throws(NameError) -> String {
        let name = name.toSingleLineCleanedString() // 去除非法字符
        guard let name else {
            throw NameError.nameIsEmpty
        }
        guard name.count >= lengthLowerBound, name.count < lengthUpperBound else {
            throw NameError.nameLengthIsInvalid(
                lowerBound: lengthLowerBound, upperBound: lengthUpperBound)
        }
        return name
    }

    /// 名称验证错误类型
    public enum NameError: Equatable, Sendable, Error {
        case nameIsEmpty
        case nameLengthIsInvalid(lowerBound: Int, upperBound: Int)
    }
}

/// 扩展 ValidatedName 以支持信任初始化方法
extension ValidatedName {
    /// 从已验证的数据源创建实例
    /// - Parameter validatedValue: 已经过验证的笔记名称
    /// - Note: 此初始化方法仅应在确保数据已经过验证的情况下使用（如从数据库读取）。
    ///         对于新输入的数据，请使用 `init(_:) throws` 方法进行验证。
    public init(trust validatedValue: String) {
        rawValue = validatedValue
    }
}

extension String {
    // 字符串处理，根据业务需求实现过滤逻辑
    func toSingleLineCleanedString() -> String? {
        let result = self.trimmingCharacters(in: .whitespacesAndNewlines)
        return result.isEmpty ? nil : result
    }
}
```

有了这个自定义类型，我们就可以在模型中使用它来替代原始的 `String` 类型：

Swift 

```
extension People {
    public var name: ValidatedName {
        get { ValidatedName(trust: nameRaw) }
        set { nameRaw = newValue.rawValue }
    }
}

// 使用示例
let people = People(content: viewContext)
people.name = try ValidatedName("    ") // error，不满足条件
```

正如 [Alex Ozun](https://x.com/AlexOzun?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 在其文章 [Making illegal states unrepresentable](https://swiftology.io/articles/making-illegal-states-unrepresentable/?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 中所强调的，非法状态是系统复杂性和意外错误的常见来源。通过精心设计的类型系统，我们可以在编译时就消除这些潜在的问题，而不必等到运行时才发现错误。

> 这种类型安全的建模方法不仅适用于 Core Data，在 SwiftData 和其他模型创建场景中同样适用。

使用这种“类型优先”的设计思路，我们不仅可以在模型层面建立公开属性与内部属性的清晰映射，还可以在更复杂的场景中将分散的原始属性整合成语义丰富的复合类型。例如，在 [Core Data 的模型继承](https://fatbobman.com/zh/posts/model-inheritance-in-core-data/) 一文中，我们介绍了如何通过这种方法避免将底层实现细节暴露给 API 使用者：

Swift 

```
// 通过枚举定义 ItemData 实体中不同数据类型的特化内容
public enum ItemDataContent: Equatable, Sendable {
    case singleValue(eventDate: Date, value: Double)
    case singleOption(eventDate: Date, optionID: UUID)
    case valueWithOption(eventDate: Date, value: Double, optionID: UUID)
    case dualValues(eventDate: Date, pairValue: ValidatedPairDouble)
    case valueWithInterval(pairDate: ValidatedPairDate, value: Double)
    case optionWithInterval(pairDate: ValidatedPairDate, optionID: UUID)
    case interval(pairDate: ValidatedPairDate)
}

// MARK: - 公开属性
extension ItemData {
    @NSManaged public var createTimestamp: Date
    @NSManaged public var uid: UUID
    @NSManaged public var item: Item?
    @NSManaged public var memo: Memo?

    // 将多个原始属性整合为一个语义丰富的复合类型
    public var dataContent: ItemDataContent? {
        get { dataContentGetter(type: type) }
        set { dateContentSetter(content: newValue) }
    }
}

// 隐藏的内部属性实现
extension ItemData {
  @NSManaged var startDate: Date?
  @NSManaged var endDate: Date?
  @NSManaged var value1: NSNumber?
  // 其他内部属性...
}
```

通过这种方式，我们成功地将底层的数据存储细节与上层的业务语义分离，为调用者提供了一个类型安全、语义明确且易于使用的 API。

## 在便捷使用与存储检索效率之间寻求平衡

尽管 SwiftData 和 Core Data 的模型声明越来越接近 Swift 的原生类型系统，但我们仍然无法完全摆脱底层 SQLite 存储机制的固有限制。在某些特定场景下，开发者常常需要在 API 便捷性和存储检索效率之间做出精妙的权衡。

以多选选项的存储为例。假设我们的应用允许用户从一组选项中选择多个项目（项目 ID 为 `Int8` ），如何既提供简洁易用的 API，又能实现高效的数据检索呢？

最直观的方案是将多选结果表示为整数数组：

Swift 

```
extension DataContent {
    public var optionSelections: [Int8] {
        get { /* 内部转换逻辑*/ } // 比如保存成数组 
        set { /* 内部转换逻辑 */ }
    }
}

// 使用示例
dataContent.optionSelections = [2, 3, 6, 1]  // 用户选择了 ID 为 2, 3, 6, 1 的选项
```

然而，这种实现方式（ 底层保存成数组 ）存在明显的检索瓶颈。例如，当我们需要查找所有包含特定选项组合（如 `[1, 6]`）的记录时，无法直接在数据库层面执行此类查询。而将这些数值转换为特定格式的字符串进行存储，虽然可行，但在数据量较大时同样会面临性能问题。

考虑到实际使用场景中每个多选组合的选项数量有限（假设不超过 10 个选项），我们可以采用位操作（bitwise operations）将多选结果编码为单个 `Int64` 值。这种方法不仅提供了简洁的 API，还能显著减少存储空间：

Swift 

```
extension DataContent {
    // 对外提供直观的数组接口
    public var optionSelections: [Int] {
        get { optionIDsNumber.toArray() }
        set { optionIDsNumber = optionIDsToInt64(optionIDs: newValue) }
    }
    
    // 内部存储为位掩码
    private var optionIDsNumber: Int64
}

func optionIDsToInt64(optionIDs: Int[Int]) -> Int64 {
    var result: Int64 = 0
    for id in optionIDs {
        if id >= 0, id <= 63 {
            // 每个ID设置对应的位
            result |= (1 << id)
        }
    }
    return result
}
NSPredicate(format: "(optionIDsNumber & %@) == %@", NSNumber(value: bitmask),NSNumber(value: bitmask))
```

由于 SQLite 天然支持高效的位运算操作，非常适合高性能检索需求场景。我们可以通过下面的谓词语句高效地查找包含特定选项组合的记录：

Swift 

```
// 构建查询谓词：查找同时包含 ID 为 2、3 和 10 的所有记录
let optionIDs = [2, 3, 10]
let bitmask = optionIDsToInt64(optionIDs: optionIDs)  // 将选项 ID 数组转换为位掩码

// 使用位与(&)运算符检查是否包含所有指定选项
let predicate = NSPredicate(
    format: "(optionIDsNumber & %@) == %@", 
    NSNumber(value: bitmask), 
    NSNumber(value: bitmask)
)
```

值得注意的是，实现这类优化需要开发者对 SwiftData/Core Data 的底层存储机制有深入理解。考虑到数据模型的变更通常伴随着高昂的迁移成本，在设计阶段就应当充分考虑未来可能的查询需求和数据量增长，避免后期重构带来的技术债务。

## 用构造方法筑起模型创建的安全防线

SwiftData 相较于 Core Data 的另一项重要改进是引入了强制性的构造方法声明机制。这一设计要求开发者必须明确定义模型的初始化方式，从而在编译时就能防止潜在的属性赋值遗漏，并为属性赋值建立起严格的时序控制。

### 复合类型的便捷构造

回顾我们前面讨论的“化零为整”策略，即使用 `ItemDataContent` 这样的复合类型来封装多个散落的原始属性。一个精心设计的构造方法可以显著降低API使用者的心智负担：

Swift 

```
extension ItemData {
    /// 创建新的数据项，但不自动插入上下文
    /// - Parameters:
    ///   - createTimestamp: 创建时间戳
    ///   - uid: 唯一标识符
    ///   - dataContent: 数据内容（复合类型）
    public convenience init(
        createTimestamp: Date,
        uid: UUID,
        dataContent: ItemDataContent // 只需提供一个复合类型，而非多个分散属性
    ) {
        self.init(entity: Self.entity(), insertInto: nil) // 借鉴SwiftData的设计，不主动插入上下文
        self.createTimestamp = createTimestamp
        self.uid = uid
        self.dataContent = dataContent
    }
}
```

这种构造方法的优势在于，它将复杂的内部属性赋值逻辑封装在模型内部，开发者只需关注业务层面的参数传递。这不仅简化了API调用，还确保了数据完整性，因为复合类型本身已经包含了必要的验证逻辑。

### 控制关系建立的时序

在构造方法中，我们采用了与 SwiftData 相似的策略——构造方法不会自动将新创建的实例插入 `NSManagedObjectContext`（`insertInto: nil`）。这一设计为关系建立提供了更加明确的控制流程：

1. 通过自定义构造方法创建 A 类型的实例 a
2. 将实例 a 显式插入管理上下文
3. 通过自定义构造方法创建 B 类型的实例 b
4. 将实例 b 显式插入管理上下文
5. 最后在 a 和 b 之间建立关系

Swift 

```
let item = Item(
    name: try ValidatedName("New Item"),
    createTimestamp: Date()
)
viewContext.insert(item) // 显式插入上下文

let itemData = ItemData(
    createTimestamp: Date(),
    uid: UUID(),
    dataContent: .singleValue(eventDate: Date(), value: 98.6)
)
viewContext.insert(itemData) // 显式插入上下文

// 建立关系
itemData.item = item
```

这种显式的关系建立流程不仅提高了代码的可读性，还避免了常见的上下文管理错误，如试图在未插入上下文的对象之间建立关系等。

### 多样化的构造接口

一个设计良好的数据模型应该提供多种构造方法，以适应不同的使用场景。通过提供多个语义清晰、参数明确的构造方法，不仅能简化 API 调用，还能在编译时为数据完整性和一致性提供更多的保障。

在模型设计中，精心设计的构造方法是确保数据一致性和安全性的第一道防线。它不仅能减少使用错误，还能为模型的演化提供更大的灵活性。当模型内部实现发生变化时，只需调整构造方法的实现，而不必修改所有调用点的代码。

[ 马上加入 ![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)的开发者社区!  加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长! ](https://t.ly/gzxeh) 

 🚀 马上加入，享受交流的快乐 →

## 不要怕麻烦

或许，很多开发者对本文中提到的技巧或多或少都有所了解，却因为担心额外工作量而止步于更细致的实现。但仔细思考一下，数据模型作为整个应用的基石，直接决定了代码的安全性、可维护性，以及未来演进的灵活程度。

我们投入在模型设计上的精力与时间，绝非“负担”，而是对技术债务的有效预防。好的模型设计能够在长期维护中持续地节省时间、减少错误，并显著提高整个应用的开发效率。

既然我们选择使用 Swift 这样一门强大且富有表达力的语言，何不彻底发挥它的优势，让类型系统成为守护业务规则的坚实防线呢？

> "加入我们的 [Discord 社区](https://discord.com/invite/7FedN5E2QQ)，与超过 2000 名苹果生态的中文开发者一起交流！"

[Core Data 改革：实现 SwiftData 般的优雅并发操作](https://fatbobman.com/zh/posts/core-data-reform-achieving-elegant-concurrency-operations-like-swiftdata/)[CoreData 探秘 - 从数据模型构建到托管对象实例](https://fatbobman.com/zh/posts/from-data-model-construction-to-managed-object-instances-in-core-data/)[SwiftData 使用前必须了解的关键问题](https://fatbobman.com/zh/posts/key-considerations-before-using-swiftdata/)[揭秘 SwiftData 的数据建模原理](https://fatbobman.com/zh/posts/unveiling-the-data-modeling-principles-of-swiftdata/)[关于 Core Data 并发编程的几点提示](https://fatbobman.com/zh/posts/concurrencyofcoredata/)[SwiftData 中的并发编程](https://fatbobman.com/zh/posts/concurret-programming-in-swiftdata/)

每周精选 Swift 与 SwiftUI 精华！

链接已复制

* [ Double 与 Double?：在类型安全与存储机制间架设桥梁 ](https://fatbobman.com/zh/posts/building-typesafe-highperformance-swiftdata-core-data-models/#double-与-double在类型安全与存储机制间架设桥梁)
* [ String 与 NonEmpty<String>：让类型系统为数据合法性背书 ](https://fatbobman.com/zh/posts/building-typesafe-highperformance-swiftdata-core-data-models/#string-与-nonempty<string>让类型系统为数据合法性背书)
* [ 在便捷使用与存储检索效率之间寻求平衡 ](https://fatbobman.com/zh/posts/building-typesafe-highperformance-swiftdata-core-data-models/#在便捷使用与存储检索效率之间寻求平衡)
* [ 用构造方法筑起模型创建的安全防线 ](https://fatbobman.com/zh/posts/building-typesafe-highperformance-swiftdata-core-data-models/#用构造方法筑起模型创建的安全防线)
* [ 复合类型的便捷构造 ](https://fatbobman.com/zh/posts/building-typesafe-highperformance-swiftdata-core-data-models/#复合类型的便捷构造)
* [ 控制关系建立的时序 ](https://fatbobman.com/zh/posts/building-typesafe-highperformance-swiftdata-core-data-models/#控制关系建立的时序)
* [ 多样化的构造接口 ](https://fatbobman.com/zh/posts/building-typesafe-highperformance-swiftdata-core-data-models/#多样化的构造接口)
* [ 不要怕麻烦 ](https://fatbobman.com/zh/posts/building-typesafe-highperformance-swiftdata-core-data-models/#不要怕麻烦)

我所希望的 Xcode

⌥ + ←

使用 equatable() 避免 NavigationLink 预构建陷阱

⌥ + →

[ ](https://x.com/intent/tweet?text=%E6%9E%84%E5%BB%BA%E7%B1%BB%E5%9E%8B%E5%AE%89%E5%85%A8%E3%80%81%E9%AB%98%E6%95%88%E7%9A%84%20SwiftData%2FCore%20Data%20%E6%A8%A1%E5%9E%8B&url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fbuilding-typesafe-highperformance-swiftdata-core-data-models%2F&hashtags=ios%2Cswiftlang&via=fatbobman) 

Share on X

Share on BlueSky

[ ](https://www.facebook.com/sharer/sharer.php?u=https%253A%252F%252Ffatbobman.com%252Fzh%252Fposts%252Fbuilding-typesafe-highperformance-swiftdata-core-data-models%252F) 

Share on Facebook

[ ](https://service.weibo.com/share/share.php?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fbuilding-typesafe-highperformance-swiftdata-core-data-models%2F&title=%E6%9E%84%E5%BB%BA%E7%B1%BB%E5%9E%8B%E5%AE%89%E5%85%A8%E3%80%81%E9%AB%98%E6%95%88%E7%9A%84%20SwiftData%2FCore%20Data%20%E6%A8%A1%E5%9E%8B%0A%20%23ios%20%23swift%0A%20by%20%40fatbobman%0A) 

Share on Weibo

[ ](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fbuilding-typesafe-highperformance-swiftdata-core-data-models%2F) 

Share on Linkedin

[ ](https://fatbobman.com/zh/posts/building-typesafe-highperformance-swiftdata-core-data-models/mailto:?subject=%E6%96%87%E7%AB%A0%3A%E3%80%90%E6%9E%84%E5%BB%BA%E7%B1%BB%E5%9E%8B%E5%AE%89%E5%85%A8%E3%80%81%E9%AB%98%E6%95%88%E7%9A%84%20SwiftData%2FCore%20Data%20%E6%A8%A1%E5%9E%8B%E3%80%91&body=%E5%88%86%E4%BA%AB%E4%B8%80%E7%AF%87%E6%96%87%E7%AB%A0%E7%BB%99%E4%BD%A0%EF%BC%9A%0A%0A%E6%9E%84%E5%BB%BA%E7%B1%BB%E5%9E%8B%E5%AE%89%E5%85%A8%E3%80%81%E9%AB%98%E6%95%88%E7%9A%84%20SwiftData%2FCore%20Data%20%E6%A8%A1%E5%9E%8B%0A%0A%E9%93%BE%E6%8E%A5%3A%20https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fbuilding-typesafe-highperformance-swiftdata-core-data-models%2F) 

Share via Email

Share Your Thoughts

Back to Top