[ #Swift ](https://fatbobman.com/zh/tags/Swift/) [ #English ](https://fatbobman.com/en/posts/notificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62/) 

# NotificationCenter.Message：Swift 6.2 并发安全通知的全新体验

[东坡肘子](https://x.com/intent/follow?screen%5Fname=fatbobman) 

 发表于 2025 年 6 月 25 日 

`NotificationCenter` 作为 iOS 开发中的经典组件，为开发者提供了灵活的广播——订阅机制。然而，随着 Swift 并发模型的不断演进，传统基于字符串标识和 `userInfo` 字典的通知方式暴露出了诸多问题：线程安全隐患、拼写错误风险、类型转换不安全等，这些问题往往只有在运行时才会被发现。

为了彻底解决这些痛点，Swift 6.2 在 Foundation 中引入了全新的并发安全通知协议：`NotificationCenter.MainActorMessage` 和 `NotificationCenter.AsyncMessage`。它们充分利用 Swift 的类型系统和并发隔离特性，让消息的发布与订阅在编译期就能得到验证，从根本上杜绝了“线程冲突”和“数据类型错误”等常见问题。

[ 马上加入 ![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)的开发者社区!  加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长! ](https://t.ly/gzxeh) 

 🚀 马上加入，享受交流的快乐 →

## 传统 Notification 的局限性

让我们先回顾一下传统 `Notification` 的使用方式：

Swift 

```
// 发布通知
NotificationCenter.default.post(
    name: .userDidLogin,
    object: authManager,
    userInfo: ["userID": user.id]
)

// 订阅通知
NotificationCenter.default.addObserver(
    forName: .userDidLogin,
    object: authManager,
    queue: .main
) { notification in
    guard let id = notification.userInfo?["userID"] as? String else { 
        return 
    }
    print("User logged in:", id)
}
```

虽然使用简单，但这种方式存在明显的安全隐患：

* **字符串标识符易错**：拼写错误只能在运行时发现，增加了调试成本
* **类型安全缺失**：`userInfo` 需要手动类型转换，容易遗漏字段或转换失败
* **并发行为不明确**：回调执行的线程取决于发送方，容易引发线程安全问题
* **编译期无法验证**：错误的使用方式只有在运行时才会暴露

特别是在使用 `@ModelActor` 等新并发特性时，传统通知机制甚至可能直接导致应用崩溃，因为我们无法在订阅时精确控制执行线程。

## NotificationCenter.Message：类型安全的解决方案

为了解决这些问题，Swift 社区在 2024 年底提出了 [“Concurrency-Safe Notification” 提案](https://github.com/swiftlang/swift-foundation/blob/main/Proposals/0011-concurrency-safe-notifications.md?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web)，并在 Swift 6.2 中正式推出。

在最终的实现中，`NotificationCenter` 提供了两个明确的消息协议：

Swift 

```
@available(macOS 26.0, iOS 26.0, tvOS 26.0, watchOS 26.0, *)
extension NotificationCenter {
    public protocol MainActorMessage: SendableMetatype {
        associatedtype Subject
        static var name: Notification.Name { get }
        static func makeMessage(_ notification: Notification) -> Self?
        static func makeNotification(_ message: Self, object: Self.Subject?) -> Notification
    }
    
    public protocol AsyncMessage: Sendable {
        associatedtype Subject
        static var name: Notification.Name { get }
        static func makeMessage(_ notification: Notification) -> Self?
        static func makeNotification(_ message: Self, object: Self.Subject?) -> Notification
    }
}
```

该体系包含两个核心协议：

* **`MainActorMessage`**：主线程消息，保证同步执行且绑定到 `@MainActor`
* **`AsyncMessage`**：异步消息，支持 `Sendable` 并可跨线程安全传递

## 实践指南

### 声明消息类型

首先，我们需要定义一个符合 `MainActorMessage` 的消息类型：

Swift 

```
public class DownloadManager {
    static let shared = DownloadManager()
}

public struct DownloadDidFinish: MainActorMessage {
    public typealias Subject = DownloadManager // 对应 NotificationCenter.default.post 中的 object 类型

    public static var name: Notification.Name {
        .init("DownloadManager.DownloadDidFinish")
    }

  	// 强类型的消息内容，替代 userInfo 字典
    public let fileURL: URL // 相当于旧版本中 userInfo 的 ["fileURL": URL]
    public let success: Bool // 相当于旧版本中 userInfo 的 ["success": Bool]
}
```

在这个示例中：

* `Subject` 定义了消息发送者的类型
* `fileURL` 和 `success` 是强类型的消息内容
* `name` 为兼容性保留（如果仅使用新 API，可以省略）

### 定义消息标识符（可选）

为了提供更好的 API 体验，我们可以定义消息标识符：

Swift 

```
extension NotificationCenter.MessageIdentifier
where Self == NotificationCenter.BaseMessageIdentifier<DownloadDidFinish> {
    static var downloadDidFinish: Self { .init() }
}
```

这样可以在订阅时使用 `.downloadDidFinish` 这样的简洁语法。

### 发送消息

新 API 提供了两种发送方式：

Swift 

```
// 关联特定实例
NotificationCenter.default.post(
    DownloadDidFinish(fileURL: url, success: true),
    subject: DownloadManager.shared
)

// 全局广播
NotificationCenter.default.post(
    DownloadDidFinish(fileURL: url, success: false)
)
```

### 订阅消息

#### 同步回调方式

Swift 

```
// 仅响应特定实例的消息
let token = NotificationCenter.default.addObserver(
    of: DownloadManager.shared,
    for: .downloadDidFinish
) { message in
    print("下载完成，成功：\(message.success)")
}

// 响应所有 DownloadDidFinish 消息
let token = NotificationCenter.default.addObserver(
    for: DownloadDidFinish.self
) { message in
    print("文件：\(message.fileURL)，成功：\(message.success)")
}
```

#### 异步流式订阅

对于 `AsyncMessage`，还可以使用 Swift 的 `AsyncSequence` 特性：

Swift 

```
struct DownloadFinishAsyncMessage: NotificationCenter.AsyncMessage {
    typealias Subject = DownloadManager

    public let fileURL: URL
    public let success: Bool
}

// 使用 for await in 语法
Task {
    for await msg in NotificationCenter.default.messages( 
        of: DownloadManager.shared,
        for: DownloadFinishAsyncMessage.self
    ) {
        print("异步收到下载完成：", msg.fileURL)
    }
}
```

## 与传统 API 的无缝集成

如果您的项目需要逐步迁移，或者要与使用传统 API 的第三方库兼容，可以实现协议约定的转换方法：

Swift 

```
extension DownloadDidFinish {
    // 将传统 Notification 转换为 Message
    public static func makeMessage(_ notification: Notification) ->
    DownloadDidFinish? {
        guard
            let info = notification.userInfo,
            let url = info["fileURL"] as? URL,
            let ok = info["success"] as? Bool
        else {
            return nil
        }
        return Self(fileURL: url, success: ok)
    }

		// 将 Message 转换为传统 Notification
    public static func makeNotification(_ message: DownloadDidFinish, object: DownloadManager?) -> Notification {
        Notification(
            name: name,
            object: object,
            userInfo: [
                "fileURL": message.fileURL,
                "success": message.success
            ]
        )
    }
}
```

有了这些转换方法，新旧 API 可以完全互通：

Swift 

```
// 使用传统方式发送
NotificationCenter.default.post(
    name: .init(rawValue: "DownloadManager.DownloadDidFinish"),
    object: nil,
    userInfo: [
        "fileURL": URL(string: "https://www.baidu.com")!,
        "success": false,
    ])
```

新 API 的订阅者会收到自动转换的 `DownloadDidFinish` 消息。

Swift 

```
// 使用传统的方式订阅
.onReceive(
    NotificationCenter.default
        .publisher(for: DownloadDidFinish.name))
{ notification in
    guard let userInfo = notification.userInfo,
          let success = userInfo["success"] as? Bool
    else {
        return
    }
    print(success)
}
```

## SwiftUI 集成方案

虽然 SwiftUI 暂未提供内置的新消息订阅修饰器，但我们可以轻松自定义：

Swift 

```
struct MessageIdentifierObserverModifier<ID>: ViewModifier
    where ID: NotificationCenter.MessageIdentifier,
    ID.MessageType: NotificationCenter.MainActorMessage,
    ID.MessageType.Subject: AnyObject
{
    let messageID: ID
    let subject: ID.MessageType.Subject?
    let perform: (ID.MessageType) -> Void
    @State var token: NotificationCenter.ObservationToken?
    init(
        messageID: ID,
        subject: ID.MessageType.Subject?,
        perform: @escaping (ID.MessageType) -> Void)
    {
        self.messageID = messageID
        self.subject = subject
        self.perform = perform
    }

    func body(content: Content) -> some View {
        content
            .onAppear {
                guard token == nil else { return }
                if let subject {
                    token = NotificationCenter.default.addObserver(
                        of: subject,
                        for: messageID
                    ) { perform($0)
                    }
                } else {
                    token = NotificationCenter.default.addObserver(
                        of: subject,
                        for: ID.MessageType.self)
                    { perform($0)
                    }
                }
            }
            .onDisappear {
                if let t = token {
                    NotificationCenter.default.removeObserver(t)
                }
            }
    }
}

struct MessageTypeObserverModifier<Message>: ViewModifier
    where Message: NotificationCenter.MainActorMessage,
    Message.Subject: AnyObject
{
    let messageType: Message.Type
    let subject: Message.Subject?
    let perform: (Message) -> Void

    @State private var token: NotificationCenter.ObservationToken?

    init(
        messageType: Message.Type,
        subject: Message.Subject? = nil,
        perform: @escaping (Message) -> Void)
    {
        self.messageType = messageType
        self.subject = subject
        self.perform = perform
    }

    func body(content: Content) -> some View {
        content
            .onAppear {
                guard token == nil else { return }
                token = NotificationCenter.default.addObserver(
                    of: subject,
                    for: messageType,
                    using: perform)
            }
            .onDisappear {
                if let t = token {
                    NotificationCenter.default.removeObserver(t)
                    token = nil
                }
            }
    }
}

extension View {
    func onReceive<Message>(
        of messageType: Message.Type,
        subject: Message.Subject? = nil,
        perform: @escaping (Message) -> Void) -> some View
        where Message: NotificationCenter.MainActorMessage,
        Message.Subject: AnyObject
    {
        modifier(
            MessageTypeObserverModifier(
                messageType: messageType,
                subject: subject,
                perform: perform))
    }
}

extension View {
    func onReceive<ID>(
        for messageID: ID,
        subject: ID.MessageType.Subject? = nil,
        perform: @escaping (ID.MessageType) -> Void) -> some View
        where
        ID: NotificationCenter.MessageIdentifier,
        ID.MessageType: NotificationCenter.MainActorMessage,
        ID.MessageType.Subject: AnyObject
    {
        modifier(MessageIdentifierObserverModifier(
            messageID: messageID,
            subject: subject,
            perform: perform))
    }
}
```

现在便可以在 SwiftUI 视图中优雅地订阅 `MainActorMessage` 消息了：

Swift 

```
// 针对特定对象
.onReceive(for: .downloadDidFinish, subject: DownloadManager.shared) { message in
    print(message.success)
}
// 全局响应
.onReceive(of: DownloadDidFinish.self) {
    print(message.success)
}
```

## 迁移建议

对于现有项目，建议采用渐进式迁移策略：

1. **新功能优先**：新开发的功能直接使用 `NotificationCenter.Message`
2. **关键路径迁移**：对于并发敏感的通知，优先迁移为 `MainActorMessage` 或 `AsyncMessage`
3. **保持兼容性**：实现 `makeMessage` 和 `makeNotification` 方法，确保新旧代码能够互通
4. **逐步替换**：在合适的时机将传统通知调用替换为新 API

[ 马上加入 ![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)的开发者社区!  加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长! ](https://t.ly/gzxeh) 

 🚀 马上加入，享受交流的快乐 →

## 总结

`NotificationCenter.Message` 为 Swift 的通知系统带来了期待已久的类型安全和并发安全特性。通过编译期检查和明确的隔离语义，它不仅提升了代码的可靠性，还提供了更好的开发体验。

如果您正在使用 Swift 6.2 且项目最低版本为 iOS 26 / macOS 26，不妨在新项目中尝试这套并发安全的通知方式，相信它会让您的代码更加安全和优雅。

> "加入我们的 [Discord 社区](https://discord.com/invite/7FedN5E2QQ)，与超过 2000 名苹果生态的中文开发者一起交流！"

[SwiftUI 中的 UserDefaults 与 Observation：如何实现精准响应](https://fatbobman.com/zh/posts/userdefaults-and-observation/)[Swifter and Swifty：掌握 Swift Testing 框架](https://fatbobman.com/zh/posts/mastering-the-swift-testing-framework/)[从 180 cm 到 5′ 11″：Swift Measurement 全解析](https://fatbobman.com/zh/posts/a-complete-guide-to-swift-measurement/)[聊聊 Combine 和 async/await 之间的合作](https://fatbobman.com/zh/posts/combineandasync/)[Swift Predicate: 用法、构成及注意事项](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/)[为自定义属性包装类型添加类 @Published 的能力](https://fatbobman.com/zh/posts/adding-published-ability-to-custom-property-wrapper-types/)

每周精选 Swift 与 SwiftUI 精华！

链接已复制

* [ 传统 Notification 的局限性 ](https://fatbobman.com/zh/posts/notificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62/#传统-notification-的局限性)
* [ NotificationCenter.Message：类型安全的解决方案 ](https://fatbobman.com/zh/posts/notificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62/#notificationcentermessage类型安全的解决方案)
* [ 实践指南 ](https://fatbobman.com/zh/posts/notificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62/#实践指南)
* [ 声明消息类型 ](https://fatbobman.com/zh/posts/notificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62/#声明消息类型)
* [ 定义消息标识符（可选） ](https://fatbobman.com/zh/posts/notificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62/#定义消息标识符可选)
* [ 发送消息 ](https://fatbobman.com/zh/posts/notificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62/#发送消息)
* [ 订阅消息 ](https://fatbobman.com/zh/posts/notificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62/#订阅消息)
* [ 与传统 API 的无缝集成 ](https://fatbobman.com/zh/posts/notificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62/#与传统-api-的无缝集成)
* [ SwiftUI 集成方案 ](https://fatbobman.com/zh/posts/notificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62/#swiftui-集成方案)
* [ 迁移建议 ](https://fatbobman.com/zh/posts/notificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62/#迁移建议)
* [ 总结 ](https://fatbobman.com/zh/posts/notificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62/#总结)

探索 SwiftUI ZStack 中的 layoutPriority 奥秘

⌥ + ←

[ ](https://x.com/intent/tweet?text=NotificationCenter.Message%EF%BC%9ASwift%206.2%20%E5%B9%B6%E5%8F%91%E5%AE%89%E5%85%A8%E9%80%9A%E7%9F%A5%E7%9A%84%E5%85%A8%E6%96%B0%E4%BD%93%E9%AA%8C&url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fnotificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62%2F&hashtags=ios%2Cswiftlang&via=fatbobman) 

Share on X

Share on BlueSky

[ ](https://www.facebook.com/sharer/sharer.php?u=https%253A%252F%252Ffatbobman.com%252Fzh%252Fposts%252Fnotificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62%252F) 

Share on Facebook

[ ](https://service.weibo.com/share/share.php?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fnotificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62%2F&title=NotificationCenter.Message%EF%BC%9ASwift%206.2%20%E5%B9%B6%E5%8F%91%E5%AE%89%E5%85%A8%E9%80%9A%E7%9F%A5%E7%9A%84%E5%85%A8%E6%96%B0%E4%BD%93%E9%AA%8C%0A%20%23ios%20%23swift%0A%20by%20%40fatbobman%0A) 

Share on Weibo

[ ](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fnotificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62%2F) 

Share on Linkedin

[ ](https://fatbobman.com/zh/posts/notificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62/mailto:?subject=%E6%96%87%E7%AB%A0%3A%E3%80%90NotificationCenter.Message%EF%BC%9ASwift%206.2%20%E5%B9%B6%E5%8F%91%E5%AE%89%E5%85%A8%E9%80%9A%E7%9F%A5%E7%9A%84%E5%85%A8%E6%96%B0%E4%BD%93%E9%AA%8C%E3%80%91&body=%E5%88%86%E4%BA%AB%E4%B8%80%E7%AF%87%E6%96%87%E7%AB%A0%E7%BB%99%E4%BD%A0%EF%BC%9A%0A%0ANotificationCenter.Message%EF%BC%9ASwift%206.2%20%E5%B9%B6%E5%8F%91%E5%AE%89%E5%85%A8%E9%80%9A%E7%9F%A5%E7%9A%84%E5%85%A8%E6%96%B0%E4%BD%93%E9%AA%8C%0A%0A%E9%93%BE%E6%8E%A5%3A%20https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fnotificationcentermessage-a-new-concurrency-safe-notification-experience-in-swift-62%2F) 

Share via Email

Share Your Thoughts

Back to Top