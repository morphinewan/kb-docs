---
---

[#Swift](https://fatbobman.com/zh/tags/Swift/) [#English](https://fatbobman.com/en/posts/default-actor-isolation/) 

# Default Actor Isolation：好初衷带来的新问题

[东坡肘子](https://x.com/intent/follow?screen%5Fname=fatbobman)

发表于2025 年 7 月 30 日

尽管 Swift 严格并发检查的初衷是好的，但对于很多单线程场景来说，却明显增加了开发者的负担。开发者不得不在代码中添加一些并不必要的`Sendable`、`@MainActor`等声明，只为了满足编译器的要求。Swift 6.2 新增的 Default Actor Isolation 功能将极大地改善这种状况，减少不必要的样板代码。本文将对 Default Actor Isolation 功能进行介绍，并指出在使用该功能后需要注意的一些情况。

 马上加入

![](https://cdn.fatbobman.com/ads/apple-100.svg) 

代码相连，创意无限 - 加入我们的开发者社区!

 加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长!

 🚀 马上加入，享受交流的快乐 →

## 初衷

很多开发者在开启 Swift 严格并发检查后会发现，一些原本运行良好的代码会出现大量的警告甚至错误。尤其是那些单线程代码，明明就运行在`MainActor`上，却仍然无法编译通过。为此，开发者需要对这些代码进行不少调整才能满足编译器的要求。

之所以出现这种情况，是因为在 Swift 6.2 之前，编译器对并发的默认推断策略是：如果一个函数或类型没有显式声明或推断出隔离域，则将其视为非隔离（non-isolated），这意味着它可以被并发使用。即便你明知这个模块的绝大多数代码只运行在 MainActor 中，但仍缺乏统一向编译器表明这一事实的手段。

Default Actor Isolation 功能（[SE-0466](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web)）正是用来解决这个问题的。该功能为开发者提供了在 Target 范围内统一向编译器表明这些代码运行在`MainActor`之上的能力。设置后，编译器在编译该 Target 代码时，会对没有明确标注隔离域的代码隐式推断为隔离到`@MainActor`中，从而减少开发者的负担。

## 使用方法

在 Xcode 26 中，新创建的项目已经默认启用了这个选项，并将默认隔离域设置为`MainActor`。如果你想将项目的 Default Actor Isolation 恢复成之前的行为，可以在 Build Settings 中进行修改。

![](https://cdn.fatbobman.com/image-20250728145844766.webp)

对于 SPM，仍采用默认非隔离的设置。你可以用如下方式将一个 Target 的 Default Actor Isolation 设置为`MainActor`：

Swift

Copied!

```
.target(
 name: "CareLogUI",
 swiftSettings: [
 .defaultIsolation(MainActor.self), // set Default Actor Isolation 
 ]),
```

## 从隔离域中退出

或许一些开发者会注意到，如果你在 Xcode 26 的 SwiftData 模板项目中添加如下代码，会报错：

Swift

Copied!

```
@Model
final class Item {
 var timestamp: Date

 init(timestamp: Date) {
 self.timestamp = timestamp
 }
}

// 新增代码，使用 SwiftData ModeActor 宏
@ModelActor
actor DataHandler {
 func createItem(timeStamp: Date) throws {
 let item = Item(timestamp: timeStamp)
 modelContext.insert(item)
 try modelContext.save()
 }
}
```

![](https://cdn.fatbobman.com/image-20250728151112896.webp)

之所以出现这个问题，是因为 Xcode 26 的模板代码都将 Default Actor Isolation 设置为了`MainActor`。在上面的代码中，`DataHandler`由于是一个`Actor`，Swift 编译器会尊重它的隔离域（而不使用默认 MainActor），但`Item`的声明没有添加任何标注，编译器会将其隐式推断为只能运行在`MainActor`中（可以理解为编译器帮我们增加了一个`@MainActor`）。如此一来，我们在一个非`MainActor`的隔离域（`DataHandler`）中创建`Item`就违反了安全并发的原则。

解决这个问题很简单。在`Item`的声明前添加`nonisolated`，这样 Swift 编译器就不会对`Item`应用默认隔离推断了。`Item`类型也就可以运行在不同的隔离域当中了。

Swift

Copied!

```
@Model
nonisolated final class Item { // 增加 nonisolated
 var timestamp: Date

 init(timestamp: Date) {
 self.timestamp = timestamp
 }
}
```

当然，如果你只是想让某个属性或方法从默认隔离域中脱离出来，可以直接在其前面添加`nonisolated`即可。

Swift

Copied!

```
class RunInMainActor {
 var name: String = "example" // 运行在 MainActor

 nonisolated func processData() async {
 // 脱离 MainActor 隔离域（编译层面）
 }

 nonisolated var computedValue: String {
 // 非隔离的计算属性（编译层面）
 return "computed"
 }
}
```

**需要注意的是**，由于 Swift 6.2 对`nonisolated`的语义进行了重要调整（引入了`nonisolated(nonsending)`作为默认行为(启用`NonisolatedNonsendingByDefault`)），`nonisolated`异步方法现在会继承调用者的隔离域，而不是像之前那样强制切换到后台执行。如果你确实需要强制方法在后台线程执行，应该使用`@concurrent`注解：

Swift

Copied!

```
class RunInMainActor {
 @concurrent
 func guaranteedBackground() async {
 // 确保在后台线程执行
 }
}
```

如果你在一个 Default Actor Isolation 为`MainActor`的项目中需要大量添加`nonisolated`时，你就要考虑是否应该继续使用该模式了。一种解决方式是将项目的默认隔离改成`nonisolated`，另一种方式是将这部分代码剥离到另一个 Target 中，使用之前的`nonisolated`默认隔离推断方式。

从某种角度来说，Default Actor Isolation 在减轻开发者负担的同时，也会促进更多人采用模块化编程。

## Default Actor Isolation 为 MainActor 与 @MainActor 并不完全一样

尽管在大多数情况下，我们可以将 Default Actor Isolation 设置为`MainActor`视作自动为未标注的类型添加`@MainActor`，但在个别情况下，两者之间还是会有一些差异。比如下面的代码：

Swift

Copied!

```
@MainActor
class UserMainActorClass {
 private final class DefaultsObservation: @unchecked Sendable {
 private var notificationObserver: NSObjectProtocol?

 deinit {
 if let observer = notificationObserver {
 NotificationCenter.default.removeObserver(observer)
 }
 }
 }
}
```

这段代码在 Default Actor Isolation 为`nonisolated`时可以正确编译，但如果切换成`MainActor`后会出现编译错误：

![](https://cdn.fatbobman.com/image-20250728154754751.webp)

此时编译器会认为内部嵌套类型的`deinit`并非在同一个隔离域中。我们必须在`deinit`前添加`isolated`或`@MainActor`才能让编译器将这个`deinit`视作运行在`@MainActor`中。

Swift

Copied!

```
class UserMainActorClass {
 private final class DefaultsObservation: @unchecked Sendable {
 private var notificationObserver: NSObjectProtocol?

 isolated // 或 @MainActor
 deinit {
 if let observer = notificationObserver {
 NotificationCenter.default.removeObserver(observer)
 }
 }
 }
}
```

我并不确定这个差异是故意设计的还是目前的一个缺陷，但至少在包含嵌套类型的声明场景中，Default Actor Isolation 为`MainActor`与显式的`@MainActor`并不完全等同。

## 宏作者面临的新挑战

几天前，我创建的[ObservableDefaults](https://github.com/fatbobman/ObservableDefaults?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web)宏收到了一个[Issue](https://github.com/fatbobman/ObservableDefaults/issues/16?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web)，使用者表示在 Xcode 26 中，当 Default Actor Isolation 为`MainActor`时，会出现编译错误：

Swift

Copied!

```
 @Sendable // ERROR: Main actor-isolated synchronous instance method 'userDefaultsDidChange' cannot be marked as '@Sendable'
```

刚看到这个 Issue 时我有些疑惑，因为在 ObservableDefaults 宏的实现中，特别对`@MainActor`做了处理。当发现用户声明的类型标注了`@MainActor`后，会在生成的代码中有相应的应对。下面这段代码就是在宏中用来判断是否给类型添加了`@MainActor`标注的：

Swift

Copied!

```
let hasExplicitMainActor = classDecl.attributes.contains(where: { attribute in
 if case let .attribute(attr) = attribute,
 let identifierType = attr.attributeName.as(IdentifierTypeSyntax.self)
 {
 return identifierType.name.text == "MainActor"
 }
 return false
})
```

显然，在 Default Actor Isolation 为`MainActor`时，并没有走对应的分支。

我在 Xcode 26 中进行测试时也发现了同样的现象。在 Default Actor Isolation 为`MainActor`时，尽管编译器会使用默认隔离域推断，但宏无法得知这个情况。

由于我还没有找到在宏中获取 Default Actor Isolation 状态的手段（可能永远不会有），最终只能增加了一个宏参数，让使用者在 Default Actor Isolation 为`MainActor`时进行显式标注，从而在宏中根据这个参数状态来进行判断：

Swift

Copied!

```
@ObservableDefaults(defaultIsolationIsMainActor: true)
class Settings {
 var name: String = "Fatbobman"
 var age: Int = 20
}
```

这或许是 Default Actor Isolation 给宏开发者带来的新挑战吧。

 马上加入

![](https://cdn.fatbobman.com/ads/apple-100.svg) 

代码相连，创意无限 - 加入我们的开发者社区!

 加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长!

 🚀 马上加入，享受交流的快乐 →

## 总结

Xcode 26 将 Default Actor Isolation 默认设置为`MainActor`或许会在短时间内给部分开发者带来一定的困扰，但熟悉了该功能后，开发者会感谢 Swift 6.2 的这个新变化，它有利于 Swift 严格并发的进一步普及。

> "加入我们的[Discord 社区](https://discord.com/invite/7FedN5E2QQ)，与超过 2000 名苹果生态的中文开发者一起交流！"

[Swifter and Swifty：掌握 Swift Testing 框架](https://fatbobman.com/zh/posts/mastering-the-swift-testing-framework/) [聊聊 Combine 和 async/await 之间的合作](https://fatbobman.com/zh/posts/combineandasync/) [NotificationCenter.Message：Swift 6.2 并发安全通知的全新体验](https://fatbobman.com/zh/posts/notificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62/) [为自定义属性包装类型添加类 @Published 的能力](https://fatbobman.com/zh/posts/adding-published-ability-to-custom-property-wrapper-types/) [深度解读 Observation —— SwiftUI 性能提升的新途径](https://fatbobman.com/zh/posts/mastering-observation/) [Swift Predicate: 用法、构成及注意事项](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/) 

 每周精选 Swift 与 SwiftUI 精华！

 链接已复制

* [初衷](https://fatbobman.com/zh/posts/default-actor-isolation/#%E5%88%9D%E8%A1%B7)
* [使用方法](https://fatbobman.com/zh/posts/default-actor-isolation/#%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95)
* [从隔离域中退出](https://fatbobman.com/zh/posts/default-actor-isolation/#%E4%BB%8E%E9%9A%94%E7%A6%BB%E5%9F%9F%E4%B8%AD%E9%80%80%E5%87%BA)
* [Default Actor Isolation 为 MainActor 与 @MainActor 并不完全一样](https://fatbobman.com/zh/posts/default-actor-isolation/#default-actor-isolation-%E4%B8%BA-mainactor-%E4%B8%8E-mainactor-%E5%B9%B6%E4%B8%8D%E5%AE%8C%E5%85%A8%E4%B8%80%E6%A0%B7)
* [宏作者面临的新挑战](https://fatbobman.com/zh/posts/default-actor-isolation/#%E5%AE%8F%E4%BD%9C%E8%80%85%E9%9D%A2%E4%B8%B4%E7%9A%84%E6%96%B0%E6%8C%91%E6%88%98)
* [总结](https://fatbobman.com/zh/posts/default-actor-isolation/#%E6%80%BB%E7%BB%93)

 Core Data 迁移事故复盘：那些被忽视的隐藏陷阱

 ⌥ + ←

[ ](https://x.com/intent/tweet?text=Default%20Actor%20Isolation%EF%BC%9A%E5%A5%BD%E5%88%9D%E8%A1%B7%E5%B8%A6%E6%9D%A5%E7%9A%84%E6%96%B0%E9%97%AE%E9%A2%98&url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fdefault-actor-isolation%2F&hashtags=ios%2Cswiftlang&via=fatbobman) 

 Share on X

 Share on BlueSky

[ ](https://www.facebook.com/sharer/sharer.php?u=https%253A%252F%252Ffatbobman.com%252Fzh%252Fposts%252Fdefault-actor-isolation%252F) 

 Share on Facebook

[ ](https://service.weibo.com/share/share.php?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fdefault-actor-isolation%2F&title=Default%20Actor%20Isolation%EF%BC%9A%E5%A5%BD%E5%88%9D%E8%A1%B7%E5%B8%A6%E6%9D%A5%E7%9A%84%E6%96%B0%E9%97%AE%E9%A2%98%0A%20%23ios%20%23swift%0A%20by%20%40fatbobman%0A) 

 Share on Weibo

[ ](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fdefault-actor-isolation%2F) 

 Share on Linkedin

[ ](https://fatbobman.com/zh/posts/default-actor-isolation/mailto:?subject=%E6%96%87%E7%AB%A0%3A%E3%80%90Default%20Actor%20Isolation%EF%BC%9A%E5%A5%BD%E5%88%9D%E8%A1%B7%E5%B8%A6%E6%9D%A5%E7%9A%84%E6%96%B0%E9%97%AE%E9%A2%98%E3%80%91&body=%E5%88%86%E4%BA%AB%E4%B8%80%E7%AF%87%E6%96%87%E7%AB%A0%E7%BB%99%E4%BD%A0%EF%BC%9A%0A%0ADefault%20Actor%20Isolation%EF%BC%9A%E5%A5%BD%E5%88%9D%E8%A1%B7%E5%B8%A6%E6%9D%A5%E7%9A%84%E6%96%B0%E9%97%AE%E9%A2%98%0A%0A%E9%93%BE%E6%8E%A5%3A%20https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fdefault-actor-isolation%2F) 

 Share via Email

 Share Your Thoughts

 Back to Top