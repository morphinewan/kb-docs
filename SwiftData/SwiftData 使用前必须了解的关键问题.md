[ #SwiftData ](https://fatbobman.com/zh/tags/SwiftData/) [ #English ](https://fatbobman.com/en/posts/key-considerations-before-using-swiftdata/) 

# SwiftData 使用前必须了解的关键问题

[东坡肘子](https://x.com/intent/follow?screen%5Fname=fatbobman) 

 发表于 2025 年 3 月 12 日 

在刚刚结束的 Let’s Vision 2025 大会上，我收到了许多关于 SwiftData 的提问：“SwiftData 是否已经足够成熟，可以用于实际项目？”、“作为初学者，如何高效地使用 SwiftData？”。这些问题反映了开发者对苹果最新数据持久化框架的浓厚兴趣，但也透露出技术选型时的犹豫。

本文旨在为对 SwiftData 感兴趣的开发者提供一份指南，帮助你了解 SwiftData 的优势与局限，并根据项目需求做出明智的技术选择。无论你是考虑在新项目中引入 SwiftData，还是计划从其他持久化方案迁移，以下内容都将为你的决策提供有价值的参考。

[ 马上加入 ![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)的开发者社区!  加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长! ](https://t.ly/gzxeh) 

 🚀 马上加入，享受交流的快乐 →

## SwiftData 和 GRDB/SQLite.swift 这类框架有什么不同？

几周前，[Point-Free](https://www.pointfree.co/?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 发布了开源库 [SharingGRDB](https://github.com/pointfreeco/sharing-grdb?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web)，极大地优化了 [GRDB.swift](https://github.com/groue/grdb.swift?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 在 SwiftUI 项目中的使用体验。这让不少开发者产生了疑问：既然 SharingGRDB 如此便捷，是否还有必要继续使用 SwiftData？

事实上，SwiftData/Core Data 与 GRDB.swift/SQLite.swift 这两类工具的核心区别，并不在于代码风格或 API 形式，而在于**数据抽象层次与组织方式的根本差异**。

SwiftData 秉承“**数据即对象**”的理念，其数据模型与 Swift 语言的面向对象编程模式无缝融合，并天然适配 SwiftUI 的声明式编程范式。相比之下，GRDB 和 SQLite.swift 等工具则更倾向于“**数据即数据库**”的思路，更适合以数据库为中心的开发流程。

如果你是一名熟悉 SQL 数据库的开发者，可能会更青睐 GRDB 等工具的使用方式。它们赋予你更多的自主性和灵活性，让你能够充分利用 SQLite 的强大功能，对数据进行精细的组织和优化。你可能会对“表”、“列”、“行”等概念感到亲切，并习惯于将这些数据库结构明确映射到 Swift 代码中。

而如果你更习惯面向对象编程思想，或者不希望（甚至不需要）关心底层持久化的实现细节，那么 SwiftData 无疑是更合适的选择。它让你能够以近乎原生的 Swift 方式管理数据对象及其关系，而“表”、“列”、“行”等数据库概念则被框架自动隐藏。

除了上述核心区别外，SwiftData/Core Data 还具备一些第三方框架难以比拟的优势：

* **免费集成基于 iCloud 的云端备份与数据同步功能**，这对于中小型或独立开发者来说，几乎是开发苹果生态应用的标配需求。
* **系统原生框架的全面支持**，在小组件、App Intent 等资源受限的场景中，SwiftData 和 Core Data 都能顺畅运行，并支持数据变化的实时响应（通过 SwiftData History 或 Persistent History Tracking 功能实现）。

因此，如果你的项目仅需本地数据存储，可以根据开发团队背景、项目需求和技术偏好选择适合的工具。但如果项目涉及资源受限场景中的持久化数据使用，选择 SwiftData 将为你省去不少麻烦。

## SwiftData 的对象（实体）≠ Swift 的对象

苹果为 SwiftData 提供了 `@Model` 宏来简化数据建模过程。许多入门教程会强调这一工具的便利性，导致一些初学者误以为可以像使用普通 Swift 对象一样构建数据模型。然而，尽管 SwiftData 尽力隐藏底层持久化细节，它仍然受到 SQLite 的物理限制：所有数据类型最终都会被转换为 SQLite 支持的格式（SwiftData/Core Data 目前并未完全利用 SQLite 支持的所有数据格式）。

在使用 SwiftData 构建数据模型时，初学者应注意以下几点：

* **优先使用基础类型**：如 `Int`、`Double`、`Date`、`URL`、`String` 等，作为模型的属性类型。
* **选择稳定的复杂类型**：对于符合 `Codable` 协议的复杂类型，尽量选择广泛使用且格式稳定的类型，例如 `CGPoint`、`CLLocation`。
* **谨慎使用自定义 `Codable` 类型**：除非确定不会再调整，否则尽量避免直接使用自定义的 `Codable` 类型。
* **枚举类型的使用**：具有 `RawValue` 的枚举类型可以直接使用。
* **处理复杂 `Codable` 类型**：对于尚未定型或较复杂的 `Codable` 类型，建议手动将其转换为基础类型进行保存，并通过计算属性简化读写操作。
* **排序与筛选的限制**：`Codable` 类型中的属性可以用于排序，但不能用于筛选条件。如果某些属性需要作为筛选条件，建议将其抽象为单独的实体，并通过关系进行关联。
* **属性与关系的可选性**：尽量将属性设置为可选类型或提供默认值，同时建议将关系也设置为可选类型。
* **避免在初始化方法中赋值关系属性**：SwiftData 采用先创建对象、后插入上下文的模式，因此不要在模型的初始化方法中直接为关系属性赋值。

遵循以上建议，可以有效降低未来因模型调整带来的兼容性问题。

> 更多相关内容，请参阅：[在 SwiftData 模型中使用 Codable 和枚举的注意事项](https://fatbobman.com/zh/posts/considerations-for-using-codable-and-enums-in-swiftdata-models/#codable-%E7%9A%84%E6%8C%81%E4%B9%85%E5%8C%96%E7%AD%96%E7%95%A5)、[揭秘 SwiftData 的数据建模原理](https://fatbobman.com/zh/posts/unveiling-the-data-modeling-principles-of-swiftdata/)。

## 数据同步

免费集成基于 iCloud 的云端备份与数据同步功能，是许多开发者选择 SwiftData/Core Data 的重要原因。与常见的网络数据同步方案不同，SwiftData 的同步机制具有以下几个特点：

### 本地存储为主，云同步为辅

SwiftData/Core Data 的云同步功能是建立在本地存储基础之上的附加功能。具体来说，苹果提供了一种自动机制，将 SwiftData/Core Data 在 SQLite 中保存的数据转换为 CloudKit 支持的数据格式。开发者完全可以选择不使用这一内置功能，而是自行实现云端同步机制。实际上，在苹果推出 Core Data with CloudKit 之前，许多开发者正是采用这种方式。

### CloudKit 不是实时数据库

数据格式的转换是异步进行的，苹果会根据设备的当前状态（如网络连接、电量水平、用户的同步设置等）动态调整同步频率。因此，开发者不应将 SwiftData 的云同步与实时数据库的性能相提并论。实际上，SwiftData 的云同步表现更接近苹果系统应用（如照片、备忘录）的同步效率。

### 基于 iCloud 的鉴权：优势与限制并存

SwiftData 的云同步依赖于 iCloud 的账户鉴权机制，这意味着用户无需额外登录，即可在所有使用相同 iCloud 账户的设备间同步数据。然而，这也带来了一定的限制：内置云同步功能与特定的 iCloud 账户紧密绑定。如果用户切换 iCloud 账户，与原账户相关的本地数据将被清除，并替换为新账户的数据。因此，开发者有必要向用户清晰地解释这一机制，以避免误解。

### 同步对模型的特殊要求

由于 CloudKit 和 SwiftData 的数据格式并非完全一致，为确保本地数据与云端兼容，创建 SwiftData 模型时需遵循以下原则：

1. **避免使用唯一性约束**：不要使用 `@Attribute(.unique)`。
2. **属性需为可选类型或提供默认值**：所有属性必须支持可选或具备默认值。
3. **关系的删除规则限制**：不支持将关系的删除规则设置为 `Deny`。
4. **关系必须为可选类型**：所有关系属性需定义为可选类型。

一旦启用 SwiftData 的内置云同步功能，**任何破坏轻量迁移兼容性的模型更改都是不允许的**。开发者应在项目初期就明确这一点。

### 自定义同步机制

事实上，苹果的许多系统应用采用了 SwiftData/Core Data 与纯 CloudKit API 结合的方案。这种方式比内置同步功能更加灵活，能够满足更复杂的同步需求。如果开发者发现 SwiftData 的内置同步功能无法满足需求，但仍希望利用 iCloud 的存储与鉴权机制，可以使用 [CKSyncEngine](https://developer.apple.com/documentation/cloudkit/cksyncengine-5sie5?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 自定义同步逻辑。不过，这种方案通常适用于特定场景，通用性较低且实现难度较大。

如果项目对数据同步的实时性或效率要求较高，建议考虑更高效的第三方同步方案，而将 SwiftData 仅作为本地存储使用，甚至可以直接选择 GRDB 或 SQLite.swift 等框架替代 SwiftData。

## 性能

通常情况下，性能与抽象程度呈反比关系。由于 SwiftData 提供了更高层次的抽象，其在数据读写性能方面明显逊于直接操作 SQLite 的框架。就纯粹的数据读写性能而言，性能排序为：**直接操作 SQLite 的框架 > Core Data > SwiftData**。

然而，如果综合考虑开发效率、自动化内存管理以及适用场景等因素，在当前硬件性能普遍较高的背景下，SwiftData 完全能够胜任主流移动和桌面应用的开发需求。

值得注意的是，SwiftData 与 Core Data 使用完全相同的底层存储格式。当 SwiftData 无法满足特定场景的性能需求时，开发者可以通过结合使用 Core Data 进行优化，例如批量数据操作或某些 SwiftData 尚未支持的高级 SQL 检索功能。

对于初学者而言，只需知道存在这种优化途径即可，除非真正遇到无法接受的性能瓶颈，否则无需在项目初期就考虑混合使用 SwiftData 与 Core Data。尽管二者底层数据同源，但同时使用两个框架对开发者仍具有一定挑战性。

> 关于批量操作和高级检索，推荐阅读：[如何在 Core Data 中进行批量操作](https://fatbobman.com/zh/posts/batchprocessingincoredata/)、[在 Core Data 中查询和使用 count 的若干方法](https://fatbobman.com/zh/posts/countincoredata/)。

## 功能

相较于 Core Data，目前 SwiftData 仍存在一些功能上的欠缺，主要表现为以下几点：

* **谓词功能支持不完整**：无法完全支持 Core Data 中的所有谓词功能。[基于 SQL 的高级检索](https://fatbobman.com/zh/posts/swift-predicate-usage-composition-and-considerations/#expression-%E5%AE%8F) 和 [谓词动态合并](https://fatbobman.com/zh/posts/how-to-dynamically-construct-complex-predicates-for-swiftdata/#ios-174--macos-144) 均需更高的系统版本才能部分实现。
* **批量操作支持有限**：尚不支持完整的批量数据操作（如批量读取、批量更新等）。
* **云同步功能限制**：还不支持 [公共数据库](https://fatbobman.com/zh/posts/coredatawithcloudkit-5/) 和 [跨 iCloud 账户的数据共享](https://fatbobman.com/zh/posts/coredatawithcloudkit-6/) 功能。
* **部分同步不支持**：不支持实体数据的部分同步（在 Core Data 中可将同一实体模型的数据分别保存在不同的存储文件中）。

此外，考虑到 SwiftData 的设计目标，Core Data 中许多针对数据对象和上下文的精细控制功能，可能不会在未来被 SwiftData 所支持。

尽管如此，当前版本的 SwiftData 已经能够满足大多数应用场景的需求。

## 稳定性

尽管 SwiftData 至今仅发布了两个主要版本，但在第二个版本（iOS 18）中，苹果对其内部进行了较为显著的调整，部分改动甚至是颠覆性的。这使得 SwiftData 在 iOS 18 上的稳定性表现有时反而不如 iOS 17。除了系统本身的 Bug 等不可控因素外，以下建议将有助于提升 SwiftData 项目的稳定性：

* **优先使用 `@ModelActor` 宏**：在 SwiftData 中，只有 `PersistentIdentifier` 和 `ModelContainer` 符合 `Sendable` 协议，因此建议优先使用 `@ModelActor` 宏来实现 [并发操作](https://fatbobman.com/zh/posts/concurret-programming-in-swiftdata/)。
* **避免在初始化方法中赋值关系属性**：SwiftData 采用先创建数据对象、后显式插入上下文的模式，因此不要在模型的初始化方法中为关系属性赋值。
* **抽象复杂属性**：若属性为 `Codable` 类型且需要排序或筛选，建议将其抽象为独立的实体，并通过关系建立关联。
* **视图刷新问题**：在 iOS 18 中，使用 `@ModelActor` 对数据进行更新后，视图并不会自动刷新（删除、添加操作则不受影响）。对于需要即时反映的数据更新操作，可考虑 [在视图上下文中进行](https://fatbobman.com/zh/posts/practical-swiftdata-building-swiftui-applications-with-modern-approaches/#%E6%96%B0%E7%9A%84%E9%97%AE%E9%A2%98%E6%95%B0%E6%8D%AE%E6%9B%B4%E6%96%B0%E5%90%8E%E8%A7%86%E5%9B%BE%E6%B2%A1%E6%9C%89%E5%88%B7%E6%96%B0)。

目前，获取和反馈 SwiftData 问题最及时的渠道是苹果官方开发者论坛。建议大家 [订阅与 SwiftData 相关的内容](https://developer.apple.com/forums/tags/rssFeed/swiftdata?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web)，以便第一时间了解 Bug 和相应的解决方案。

[ 马上加入 ![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)的开发者社区!  加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长! ](https://t.ly/gzxeh) 

 🚀 马上加入，享受交流的快乐 →

## 谋定而后动

SwiftData 以其“易学易用”的表象，很容易让初学者产生误解，匆忙将其引入生产环境。然而，这种仓促的决策往往会带来意想不到的后果：只有当兼容性问题或稳定性危机爆发时，开发者才会意识到自己对这一框架的理解还远远不够。这种“先上车再说”的做法，在数据持久化这样的基础设施选择上尤为危险。

对于 SwiftData 这类核心框架，明智的做法是遵循“谋定而后动”的原则——深入了解其特性与局限，权衡利弊，并通过小规模试验验证其可行性，最终再做出技术选型决策。尤其是在 AI 辅助编程工具迅速普及、代码生成效率大幅提升的今天，这种谨慎态度显得尤为重要。毕竟，技术债务的积累速度可能远超我们的预期。

作为未来十年苹果生态中最为关键的数据持久化解决方案之一，SwiftData 无疑值得开发者投入时间和精力深入研究。它不仅代表了苹果对数据处理的最新思考，其演进方向也将持续影响整个生态系统的开发范式。

选择 SwiftData，还是其他替代方案？答案并不完全取决于技术本身，而在于你对项目需求和团队能力的清晰认知。希望本文能为你的决策提供有价值的参考，助你在技术选型的道路上少走弯路。

> "加入我们的 [Discord 社区](https://discord.com/invite/7FedN5E2QQ)，与超过 2000 名苹果生态的中文开发者一起交流！"

[关于 Core Data 并发编程的几点提示](https://fatbobman.com/zh/posts/concurrencyofcoredata/)[在 Spotlight 中展示应用中的 Core Data 数据](https://fatbobman.com/zh/posts/spotlight/)[WWDC 2023 Core Data 有哪些新变化](https://fatbobman.com/zh/posts/what-s-new-in-core-data-in-wwdc23/)[以 SwiftData 之道，重塑 Core Data 开发](https://fatbobman.com/zh/posts/reinventing-core-data-development-with-swiftdata-principles/)[SwiftData 实战：用现代方法构建 SwiftUI 应用](https://fatbobman.com/zh/posts/practical-swiftdata-building-swiftui-applications-with-modern-approaches/)[SwiftData in WWDC 2024：革命仍在继续、稳定还需时日](https://fatbobman.com/zh/posts/swiftdata-in-wwdc2024/)

每周精选 Swift 与 SwiftUI 精华！

链接已复制

* [ SwiftData 和 GRDB/SQLite.swift 这类框架有什么不同？ ](https://fatbobman.com/zh/posts/key-considerations-before-using-swiftdata/#swiftdata-和-grdbsqliteswift-这类框架有什么不同)
* [ SwiftData 的对象（实体）≠ Swift 的对象 ](https://fatbobman.com/zh/posts/key-considerations-before-using-swiftdata/#swiftdata-的对象实体≠-swift-的对象)
* [ 数据同步 ](https://fatbobman.com/zh/posts/key-considerations-before-using-swiftdata/#数据同步)
* [ 本地存储为主，云同步为辅 ](https://fatbobman.com/zh/posts/key-considerations-before-using-swiftdata/#本地存储为主云同步为辅)
* [ CloudKit 不是实时数据库 ](https://fatbobman.com/zh/posts/key-considerations-before-using-swiftdata/#cloudkit-不是实时数据库)
* [ 基于 iCloud 的鉴权：优势与限制并存 ](https://fatbobman.com/zh/posts/key-considerations-before-using-swiftdata/#基于-icloud-的鉴权优势与限制并存)
* [ 同步对模型的特殊要求 ](https://fatbobman.com/zh/posts/key-considerations-before-using-swiftdata/#同步对模型的特殊要求)
* [ 自定义同步机制 ](https://fatbobman.com/zh/posts/key-considerations-before-using-swiftdata/#自定义同步机制)
* [ 性能 ](https://fatbobman.com/zh/posts/key-considerations-before-using-swiftdata/#性能)
* [ 功能 ](https://fatbobman.com/zh/posts/key-considerations-before-using-swiftdata/#功能)
* [ 稳定性 ](https://fatbobman.com/zh/posts/key-considerations-before-using-swiftdata/#稳定性)
* [ 谋定而后动 ](https://fatbobman.com/zh/posts/key-considerations-before-using-swiftdata/#谋定而后动)

让 @State 实现懒加载

⌥ + ←

使用 Proxyman 拦截和模拟 iPhone 应用的网络请求

⌥ + →

[ ](https://x.com/intent/tweet?text=SwiftData%20%E4%BD%BF%E7%94%A8%E5%89%8D%E5%BF%85%E9%A1%BB%E4%BA%86%E8%A7%A3%E7%9A%84%E5%85%B3%E9%94%AE%E9%97%AE%E9%A2%98&url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fkey-considerations-before-using-swiftdata%2F&hashtags=ios%2Cswiftlang&via=fatbobman) 

Share on X

Share on BlueSky

[ ](https://www.facebook.com/sharer/sharer.php?u=https%253A%252F%252Ffatbobman.com%252Fzh%252Fposts%252Fkey-considerations-before-using-swiftdata%252F) 

Share on Facebook

[ ](https://service.weibo.com/share/share.php?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fkey-considerations-before-using-swiftdata%2F&title=SwiftData%20%E4%BD%BF%E7%94%A8%E5%89%8D%E5%BF%85%E9%A1%BB%E4%BA%86%E8%A7%A3%E7%9A%84%E5%85%B3%E9%94%AE%E9%97%AE%E9%A2%98%0A%20%23ios%20%23swift%0A%20by%20%40fatbobman%0A) 

Share on Weibo

[ ](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fkey-considerations-before-using-swiftdata%2F) 

Share on Linkedin

[ ](https://fatbobman.com/zh/posts/key-considerations-before-using-swiftdata/mailto:?subject=%E6%96%87%E7%AB%A0%3A%E3%80%90SwiftData%20%E4%BD%BF%E7%94%A8%E5%89%8D%E5%BF%85%E9%A1%BB%E4%BA%86%E8%A7%A3%E7%9A%84%E5%85%B3%E9%94%AE%E9%97%AE%E9%A2%98%E3%80%91&body=%E5%88%86%E4%BA%AB%E4%B8%80%E7%AF%87%E6%96%87%E7%AB%A0%E7%BB%99%E4%BD%A0%EF%BC%9A%0A%0ASwiftData%20%E4%BD%BF%E7%94%A8%E5%89%8D%E5%BF%85%E9%A1%BB%E4%BA%86%E8%A7%A3%E7%9A%84%E5%85%B3%E9%94%AE%E9%97%AE%E9%A2%98%0A%0A%E9%93%BE%E6%8E%A5%3A%20https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fkey-considerations-before-using-swiftdata%2F) 

Share via Email

Share Your Thoughts

Back to Top