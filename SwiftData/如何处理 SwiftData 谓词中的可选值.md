[ #SwiftData ](https://fatbobman.com/zh/tags/SwiftData/) [ #English ](https://fatbobman.com/en/posts/how-to-handle-optional-values-in-swiftdata-predicates/) 

# 如何处理 SwiftData 谓词中的可选值

[东坡肘子](https://x.com/intent/follow?screen%5Fname=fatbobman) 

 发表于 2024 年 2 月 22 日 

由于 SwiftData 更改了数据模型的创建机制，而且谓词创建也采用了基于模型代码的类型安全模式。因此，当开发者在为 SwiftData 构建谓词时会遇到大量的处理可选值的操作。本文将探讨在构建谓词时，处理可选值的一些技巧和注意事项。

[ 马上加入 ![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)的开发者社区!  加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长! ](https://t.ly/gzxeh) 

 🚀 马上加入，享受交流的快乐 →

## 从 “由里至表” 到 “由表及里” 的变革

在 SwiftData 的众多创新中，最引人注目的莫过于允许开发者通过纯代码直接声明数据模型。在 Core Data 中，开发者需要先在 Xcode 的数据模型编辑器里创建数据模型（对应 NSManagedObjectModel），然后才是编写或自动生成 NSManagedObject 子类代码。

这一过程本质上是从模型（“里”）到类型代码（“表”）的转换。开发者能在一定程度上调整类型代码，例如将 `Optional` 改为 `Non-Optional` 或将 `NSSet` 改为 `Set`，从而优化开发体验，只要这些修改不影响 Core Data 在代码与模型之间的映射。

SwiftData 的纯代码声明方式彻底改变了这一流程。在 SwiftData 中，类型代码和数据模型的声明是同步进行的，或者更准确地说，SwiftData 会根据开发者声明的类型代码自动生成相应的数据模型。声明方式由传统的“由里至表”转变为了“由表及里”。

## 可选值与 Predicate

在为 Core Data 创建谓词的操作中，谓词表达式与类型代码并无直接联系。谓词表达式中对应的是模型编辑器内定义的属性（ 数据模型 ），而这些属性的“可选值”特性并不对应 Swift 语言中的可选类型概念，而是指 SQLite 字段是否可以为 `NULL`。这意味着，当谓词表达式中出现了可为 NULL 的属性和一个非 NULL 值时，通常不需要考虑其可选性。

Swift 

```
public class Note: NSManagedObject {
    @NSManaged public var name: String?
}

let predicate = NSPredicate(format: "name BEGINSWITH %@", "fat")
```

然而，SwiftData 的出现改变了这一情况。由于 SwiftData 谓词的构建基于模型代码，其中的可选类型真正体现了 Swift 的可选值概念，这就要求我们在构建谓词时必须特别注意可选值的处理。

考虑以下 SwiftData 的代码实例，如果不适当处理可选值，将会导致编译错误：

Swift 

```
@Model
final class Note {
  var name: String?
  init(name: String?) {
    self.name = name
  }
}

let predicate1 = #Predicate<Note> { note in
  note.name.starts(with: "fat")  // error 
}
// Value of optional type 'String?' must be unwrapped to refer to member 'starts' of wrapped base type 'String'

let predicate2 = #Predicate<Note> { note in
  note.name?.starts(with: "fat")  // error 
}
// Cannot convert value of type 'Bool?' to closure result type 'Bool'
```

因此，在为 SwiftData 构建谓词时，正确处理可选值成为一个重要的考虑因素。

## 正确处理 SwiftData 中的可选值

SwiftData 的谓词构建虽与编写返回布尔值的闭包相似，但开发者仅可以在其中使用[官方文档](https://developer.apple.com/documentation/foundation/predicate?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web)中列出的操作符和方法，这些操作都将通过宏转换成相应的 `PredicateExpressions`。对于上文中的可选类型的 `name` 属性，开发者可以采用以下方法进行处理：

### 方法 1：使用可选链和空合并运算符

通过结合可选链（`?.`）和空合并运算符（`??`），可以在属性为 `nil` 时提供一个默认的布尔值。

Swift 

```
let predicate1 = #Predicate<Note> { 
  $0.name?.starts(with: "fat") ?? false
}
```

### 方法 2：使用条件绑定

利用条件绑定（`if let`），可以在属性非空时执行特定逻辑，否则返回 `false`。

Swift 

```
let predicate2 = #Predicate<Note> {
  if let name = $0.name {
    return name.starts(with: "fat")
  } else {
    return false
  }
}
```

注意，在谓词的 body 中只能包含单一表达式。因此，尝试在 `if` 之外再返回一个值，将无法构建有效的谓词：

Swift 

```
let predicate2 = #Predicate<Note> {
  if let name = $0.name {
    return name.starts(with: "fat")
  }
  return false
}
```

这里的限制指的是，`if else` 结构和 `if` 结构，都各自被视为一个单一的表达式，它们都有直接对应的 `PredicateExpressions`，而在一个 `if` 结构之外的额外返回则对应两个不同的表达式。

尽管谓词闭包中只能包含一个表达式，通过嵌套使用仍可以构建复杂的查询逻辑。

### 方法 3：使用 flatMap 方法

`flatMap` 方法能够处理可选值，并在非 `nil` 时应用给定的闭包，其结果仍然可以使用空合并运算符提供默认值。

Swift 

```
let predicate3 = #Predicate<Note> {
  $0.name.flatMap { $0.starts(with: "fat") } ?? false
}
```

> 以上提供了几种安全且有效的策略，以确保在 SwiftData 谓词构建中正确处理可选值，从而避免编译时或运行时错误，确保数据查询的准确性和稳定性。

### 错误的做法：使用强制解包

尽管开发者可能确信某属性非空，使用 `!` 进行强制解包在 SwiftData 谓词中仍可能导致运行时错误。

Swift 

```
let predicate = #Predicate<Note> {
  $0.name!.starts(with: "fat") // error
}

// Runtime Error: SwiftData.SwiftDataError._Error.unsupportedPredicate
```

### 不可处理的可选值

当前（截止 Xcode 15C500b），当数据模型包含对多关系且为可选时，上述方法均不起作用。例如：

Swift 

```
let predicate = #Predicate<Memo>{
      $0.assets?.isEmpty == true
}

// or 

let predicate = #Predicate<Memo>{ $0.assets == nil }
```

SwiftData 在将谓词转换成 SQL 指令后，都会出现运行时错误：

Bash 

```
error: SQLCore dispatchRequest: exception handling request: <NSSQLCountRequestContext: 0x6000038dc620> , to-many key not allowed here with userInfo of (null)
```

## 特殊情形下的可选值处理

在 SwiftData 中构建谓词时，尽管常规情况下需要采用特定方法来处理可选值，但存在一些特殊情况，其中的处理规则略有不同。

### 直接等值比较

当仅需对可选值进行等值（`==`）比较操作时，SwiftData 允许直接比较，无需额外处理可选性。这意味着即便属性是可选类型，也可以直接进行比较，如下所示：

Swift 

```
let predicate = #Predicate<Note> {
  $0.name == "root"
}
```

同样，这一规则也适用于对象之间的可选关系属性比较。例如，在 `Item` 与 `Note` 构成的一对一可选关系中，可以直接进行比较（ 即使 `name` 也是可选类型 ）：

Swift 

```
let predicate = #Predicate<Item> {
  $0.note?.name == "root"
}
```

### 可选链的特例

虽然当可选链中只包含一个 `?` 时，在等值比较中无需特别处理，但当涉及到包含多个 `?` 的情况时，尽管代码在编译、运行期间不报错，但 SwiftData 无法通过该谓词从数据库中检索出正确的结果。

考虑下面的场景，其中 `Item` 和 `Note` 之间存在一对一的可选关系，`Note` 与 `Parent` 之间也是一对一的可选关系：

Swift 

```
let predicate = #Predicate<Item> {
  $0.note?.parent?.persistentModelID == rootNoteID
}
```

要解决这个问题，需要确保可选链中只包含一个 `?`。这可以通过将部分可选链解包来实现，例如：

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

或者：

Swift 

```
let predicate = #Predicate<Item> {
  if let note = $0.note, let parent = note.parent {
    return parent.persistentModelID == rootNoteID
  } else {
    return false
  }
}
```

> 该问题已在 iOS 17.5+ 中修复，不再存在。

[ 马上加入 ![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)的开发者社区!  加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长! ](https://t.ly/gzxeh) 

 🚀 马上加入，享受交流的快乐 →

## 总结

在本文中，我们探讨了如何在 SwiftData 中正确处理谓词构建过程中的可选值问题。通过介绍不同的方法，包括使用可选链和空合并运算符、条件绑定、以及 `flatMap` 方法，我们提供了有效处理可选性的策略。同时，我们也指出了直接等值比较可选值时的特殊情况，以及在可选链中包含多个 `?` 时需要采取的特别处理措施。这些技巧和注意事项旨在帮助开发者避免常见的陷阱，确保能够构建出准确且高效的数据查询谓词，从而充分利用 SwiftData 的强大功能。

> "加入我们的 [Discord 社区](https://discord.com/invite/7FedN5E2QQ)，与超过 2000 名苹果生态的中文开发者一起交流！"

[WWDC 2023 Core Data 有哪些新变化](https://fatbobman.com/zh/posts/what-s-new-in-core-data-in-wwdc23/)[如何为 SwiftData 动态的构建复杂的谓词](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/)[如何通过 Persistent History Tracking 观察 SwiftData 的数据变化](https://fatbobman.com/zh/posts/persistent-history-tracking-in-swiftdata/)[SwiftData 中的并发编程](https://fatbobman.com/zh/posts/concurret-programming-in-swiftdata/)[SwiftDataKit：让你在 SwiftData 中使用 Core Data 的高级功能](https://fatbobman.com/zh/posts/use-core-data-features-in-swiftdata-by-swiftdatakit/)[CoreData 探秘 - 从数据模型构建到托管对象实例](https://fatbobman.com/zh/posts/from-data-model-construction-to-managed-object-instances-in-core-data/)

每周精选 Swift 与 SwiftUI 精华！

链接已复制

* [ 从 “由里至表” 到 “由表及里” 的变革 ](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/#从-由里至表-到-由表及里-的变革)
* [ 可选值与 Predicate ](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/#可选值与-predicate)
* [ 正确处理 SwiftData 中的可选值 ](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/#正确处理-swiftdata-中的可选值)
* [ 方法 1：使用可选链和空合并运算符 ](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/#方法-1使用可选链和空合并运算符)
* [ 方法 2：使用条件绑定 ](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/#方法-2使用条件绑定)
* [ 方法 3：使用 flatMap 方法 ](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/#方法-3使用-flatmap-方法)
* [ 错误的做法：使用强制解包 ](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/#错误的做法使用强制解包)
* [ 不可处理的可选值 ](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/#不可处理的可选值)
* [ 特殊情形下的可选值处理 ](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/#特殊情形下的可选值处理)
* [ 直接等值比较 ](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/#直接等值比较)
* [ 可选链的特例 ](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/#可选链的特例)
* [ 总结 ](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/#总结)

探讨 SwiftUI 中的属性包装器：@UIApplicationDelegateAdaptor、@AccessibilityFocusState、@FocusedObject、@FocusedValue 和 @FocusedBinding

⌥ + ←

Swift Predicate: 用法、构成及注意事项

⌥ + →

[ ](https://x.com/intent/tweet?text=%E5%A6%82%E4%BD%95%E5%A4%84%E7%90%86%20SwiftData%20%E8%B0%93%E8%AF%8D%E4%B8%AD%E7%9A%84%E5%8F%AF%E9%80%89%E5%80%BC&url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fhow-to-handle-optional-values-in-swiftdata-predicates%2F&hashtags=ios%2Cswiftlang&via=fatbobman) 

Share on X

Share on BlueSky

[ ](https://www.facebook.com/sharer/sharer.php?u=https%253A%252F%252Ffatbobman.com%252Fzh%252Fposts%252Fhow-to-handle-optional-values-in-swiftdata-predicates%252F) 

Share on Facebook

[ ](https://service.weibo.com/share/share.php?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fhow-to-handle-optional-values-in-swiftdata-predicates%2F&title=%E5%A6%82%E4%BD%95%E5%A4%84%E7%90%86%20SwiftData%20%E8%B0%93%E8%AF%8D%E4%B8%AD%E7%9A%84%E5%8F%AF%E9%80%89%E5%80%BC%0A%20%23ios%20%23swift%0A%20by%20%40fatbobman%0A) 

Share on Weibo

[ ](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fhow-to-handle-optional-values-in-swiftdata-predicates%2F) 

Share on Linkedin

[ ](https://fatbobman.com/zh/posts/how-to-handle-optional-values-in-swiftdata-predicates/mailto:?subject=%E6%96%87%E7%AB%A0%3A%E3%80%90%E5%A6%82%E4%BD%95%E5%A4%84%E7%90%86%20SwiftData%20%E8%B0%93%E8%AF%8D%E4%B8%AD%E7%9A%84%E5%8F%AF%E9%80%89%E5%80%BC%E3%80%91&body=%E5%88%86%E4%BA%AB%E4%B8%80%E7%AF%87%E6%96%87%E7%AB%A0%E7%BB%99%E4%BD%A0%EF%BC%9A%0A%0A%E5%A6%82%E4%BD%95%E5%A4%84%E7%90%86%20SwiftData%20%E8%B0%93%E8%AF%8D%E4%B8%AD%E7%9A%84%E5%8F%AF%E9%80%89%E5%80%BC%0A%0A%E9%93%BE%E6%8E%A5%3A%20https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fhow-to-handle-optional-values-in-swiftdata-predicates%2F) 

Share via Email

Share Your Thoughts

Back to Top