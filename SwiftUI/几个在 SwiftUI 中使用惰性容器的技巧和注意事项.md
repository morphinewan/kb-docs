[ #SwiftUI ](https://fatbobman.com/zh/tags/SwiftUI/) [ #English ](https://fatbobman.com/en/posts/tips-and-considerations-for-using-lazy-containers-in-swiftui/) 

# 几个在 SwiftUI 中使用惰性容器的技巧和注意事项

[东坡肘子](https://x.com/intent/follow?screen%5Fname=fatbobman) 

 发表于 2024 年 3 月 14 日 

在 SwiftUI 的框架中，惰性布局容器，如 List 和 LazyVStack，提供了一种高效展示大型数据集的方法。这些容器的设计精妙，它们仅在必要时才动态地构建和加载视图，从而显著优化了应用的性能和内存使用效率。本文将探讨一些实用技巧和重要注意事项，旨在赋予开发者利用 SwiftUI 惰性容器时增强应用响应性和资源管理的能力。

> 关于 `List` 和 `LazyVStack` 的选择策略，建议阅读 [List 还是 LazyVStack：SwiftUI 中的惰性容器选择](https://fatbobman.com/zh/posts/list-or-lazyvstack/)。

[ 马上加入 ![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)的开发者社区!  加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长! ](https://t.ly/gzxeh) 

 🚀 马上加入，享受交流的快乐 →

## 自定义遵循 RandomAccessCollection 的实现

> 从 iOS 18 开始，ForEach 的内部刷新机制发生了调整，这个技巧已经无效了。

在某些情况下，我们的数据源可能无法直接与 SwiftUI 的 ForEach 构造器兼容。由于 ForEach 需要数据源遵循 `RandomAccessCollection` 协议以支持高效的随机访问索引操作, 因此我们需要为不兼容的数据源自定义一个遵循该协议的数据类型, 从而优化性能和内存使用。

以 [Swift Collection](https://github.com/apple/swift-collections?utm%5Fsource=Fatbobman%20Blog&utm%5Fmedium=web) 库中的 OrderedDictionary 为例, 它结合了字典高效的键值存储和有序集合的特性。如果直接将它转换为数组, 可能会导致不必要的内存增加, 尤其在处理大型数据集时更为明显。通过自定义一个符合 RandomAccessCollection 协议的类型, 我们可以有效地管理内存, 同时保持高效的数据处理能力。

下面是一个实现和使用这样自定义集合的示例:

Swift 

```
// 创建 DictDataSource,符合 RandomAccessCollection 的最小实现
final class DictDataSource<Key, Value>: RandomAccessCollection, ObservableObject where Key: Hashable {
    typealias Index = Int
    
    private var dict: OrderedDictionary<Key, Value>
    
    init(dict: OrderedDictionary<Key, Value>) {
        self.dict = dict
    }
    
    var startIndex: Int {
        0
    }
    
    var endIndex: Int {
        dict.count
    }
    
    subscript(position: Int) -> (key: Key, value: Value) {
        dict.elements[position]
    }
}

// 使用方式
let responses: OrderedDictionary<Int, String> = [
    200: "OK",
    403: "Access forbidden", 
    404: "File not found",
    500: "Internal server error",
]

struct OrderedCollectionsDemo: View {
    @StateObject var dataSource = DictDataSource(dict: responses)
    var body: some View {
        List {
            ForEach(dataSource, id: \.key) { element in
                Text("\(element.key) : \(element.value)")
            }
        }
    }
}
```

通过这种方式, 我们不仅优化了大型数据集的处理效率, 还减少了内存占用, 提高了应用的整体响应能力。除了 OrderedDictionary, 其他自定义数据结构如链表、树等, 同样可以通过类似方式实现随机访问的数据源适配器, 以更好地配合 ForEach 的使用。

## 实现无限数据加载

在开发过程中，我们经常遇到需要实现无限数据加载的场景，这意味着根据用户的滚动或需求动态地加载新的数据集。这种机制不仅有助于优化内存使用，还能显著提升用户体验，特别是在处理大量数据时。

通过引入自定义的 `RandomAccessCollection` 实现，我们可以直接在数据源层面嵌入动态加载逻辑。以下展示的 `DynamicDataSource` 类遵循 `RandomAccessCollection` 协议，并在数据接近末尾时自动触发更多数据的加载。

Swift 

```
struct Item: Identifiable, Sendable {
  let id = UUID()
  let number: Int
}

final class DynamicDataSource: RandomAccessCollection, ObservableObject {
  typealias Element = Item
  typealias Index = Int

  @Published private var items: [Item]
  private var isLoadingMoreData = false
  private let threshold = 10 // 设置触发加载更多数据的阈值

  init(initialItems: [Item] = []) {
    items = initialItems
  }

  var startIndex: Int {
    items.startIndex
  }

  var endIndex: Int {
    items.endIndex
  }

  func formIndex(after i: inout Int) {
    i += 1
    // 触发条件，在访问到小于最后数据 10 的索引值数据时加载新数据
    if i >= (items.count - threshold) && !isLoadingMoreData && !items.isEmpty {
      loadMoreData()
    }
  }

  subscript(position: Int) -> Item {
    items[position]
  }

  private func loadMoreData() {
    guard !isLoadingMoreData else { return }

    isLoadingMoreData = true

    // 模拟异步加载新数据
    Task {
      try? await Task.sleep(for: .seconds(1.5))
      let newItems = (0 ..< 30).map { _ in Item(number: Int.random(in: 0 ..< 1000)) }
      await MainActor.run { [weak self] in
        self?.items.append(contentsOf: newItems)
        self?.isLoadingMoreData = false
      }
    }
  }
}
```

在视图中的使用演示如下，随着用户的滚动，新数据会被动态加载到列表中：

Swift 

```
struct DynamicDataLoader: View {
  @StateObject var dataSource = DynamicDataSource(initialItems: (0 ..< 50).map { _ in Item(number: Int.random(in: 0 ..< 1000)) })
  var body: some View {
    List {
      ForEach(dataSource) { item in
        Text("\(item.number)")
      }
    }
  }
}
```

通过上述方法，我们成功实现了动态数据加载逻辑与视图渲染逻辑的分离，从而使代码结构更加清晰和易于维护。这种实现模式的适用性极广，不仅可以用于数据模拟，还能轻松扩展以适应更多实际需求，如从网络加载数据、实现数据的分页加载。

## id 修饰器对 List 懒加载机制的影响

在 SwiftUI 中，List 视图依赖懒加载机制来优化性能和用户体验。懒加载机制确保只有在子视图即将出现在可视区域时，SwiftUI 才会创建这些视图的实例并将它们加载到视图树中。然而，在一些特定情况下，这一优化机制可能会被意外破坏，尤其是当子视图使用了 `id` 修饰器时。

考虑下面的示例，我们给 `ItemSubView` 添加了 `id` 修饰器。结果是，List 会立即为所有项目实例化子视图，虽然它只为当前可见的子视图渲染内容（ 对 body 求值 ）：

Swift 

```
struct LazyBrokenView: View {
  @State var items = (0 ..< 50).map { Item(number: $0) }
  var body: some View {
    List {
      ForEach(items) { item in
        ItemSubView(item: item)
          .id(item.id)
      }
    }
  }
}

struct ItemSubView: View {
  let item: Item
  init(item: Item) {
    self.item = item
    print("\(item.number) init")
  }

  var body: some View {
    let _ = print("\(item.number) update")
    Text("\(item.number)")
      .frame(height: 30)
  }
}
```

尽管 List 的懒加载机制仅对可见视图渲染，但在数据量庞大时，立即实例化所有子视图会导致巨大的性能开销，严重影响应用的初始加载效率。

> 在 `ScrollViewReader` \+ `List` 的应用场景中，应考虑让 `ForEach` 的迭代数据同时符合 `Identifiable` 和 `Hashable` 协议，以避免使用 `id` 来显式标注标识符。

虽然 `id` 修饰器会影响 List 的懒加载特性，但它在 `LazyVStack`、`LazyVGrid` 等其他惰性布局容器中的使用是安全的。

综上所述，在充分利用 SwiftUI 的惰性容器优势时，我们必须注意这些潜在的陷阱，特别是在需要展示大量数据的场景中，否则可能会严重影响应用程序的性能和用户体验。

## 在惰性容器中，SwiftUI 仅保留 ForEach 子视图最顶层的状态

在 SwiftUI 的惰性容器使用中，特别是当子视图由多层视图结构组成并通过 `ForEach` 构建时，一个重要的细节需要注意：这些视图在离开可视区域并再次出现时，SwiftUI 只会保留最顶层视图中的状态。

这意味着，在如下代码示例中，`RootView` 作为 `ForEach` 循环中的子视图，包含了另一个视图 `ChildView`。根据 SwiftUI 的设计，当 `RootView` 离开后重新进入可视区域，只有位于 `RootView` 中的状态会被保持，而嵌套在 `ChildView` 内部的状态将被重置：

Swift 

```
struct StateLoss: View {
  var body: some View {
    List {
      ForEach(0 ..< 100) { i in
        RootView(i: i)
      }
    }
  }
}

struct RootView: View {
  @State var topState = false
  let i: Int
  var body: some View {
    VStack {
      Text("\(i)")
      Toggle("Top State", isOn: $topState)
      ChildView()
    }
  }
}

struct ChildView: View {
  @State var childState = false
  var body: some View {
    VStack {
      Toggle("Child State", isOn: $childState)
    }
  }
}
```

我最初认为这种行为应该是 Bug，但根据与苹果工程师的交流反馈，这实际上是一种有意为之的设计决策。从 SwiftUI 的设计哲学来看，这种选择有其合理性。为了在渲染效率和资源消耗之间取得平衡，保持子视图树中所有状态对于拥有复杂视图树和大量子视图的惰性容器来说，可能会大大降低效率。因此，决定仅维护最顶层子视图的状态，算是一种合理的设计选择。

因此，对于拥有多层且状态丰富的子视图结构，推荐的做法是将所有相关状态提升到顶层视图中，以确保状态的正确保持：

Swift 

```
struct RootView: View {
  @State var topState = false
  @State var childState = false
  let i: Int
  var body: some View {
    VStack {
      Text("\(i)")
      Toggle("Top State", isOn: $topState)
      ChildView(childState: $childState)
    }
  }
}

struct ChildView: View {
  @Binding var childState: Bool
  var body: some View {
    VStack {
      Toggle("Child State", isOn: $childState)
    }
  }
}
```

这样的做法不仅遵循 SwiftUI 的设计原则，还确保了即使在复杂的视图层次中，应用的状态也能被正确管理和保留。

## 针对特定类型的状态, SwiftUI 释放内存资源不够积极

在 Swift 开发者的普遍认识中，将可选值类型的变量设置为 `nil` 应当能够触发系统回收原有数据占用的资源。因此，一些开发者在面对可能造成重大资源占用的惰性视图时, 尝试利用此机制来节省内存。下面的示例展示了这种做法：

Swift 

```
struct MemorySave: View {
  var body: some View {
    List {
      ForEach(0 ..< 100) { _ in
        ImageSubView()
      }
    }
  }
}

struct ImageSubView: View {
  @State var image: Image?
  var body: some View {
    VStack {
      if let image {
        image
          .resizable()
          .frame(width: 200, height: 200)
      } else {
        Rectangle().fill(.gray.gradient)
          .frame(width: 200, height: 200)
      }
    }
    .task {
      // 模拟加载不同的图片
      if let uiImage = createImage() {
        image = Image(uiImage: uiImage)
      }
    }
    .onDisappear {
      // 在离开可视区域时，设置为 nil
      image = nil
    }
  }

  func createImage() -> UIImage? {
    let colors: [UIColor] = [.black, .blue, .yellow, .cyan, .green, .magenta]
    let color = colors.randomElement()!
    let size = CGSize(width: 1000, height: 1000)
    UIGraphicsBeginImageContextWithOptions(size, false, 0)
    color.setFill()
    UIRectFill(CGRect(origin: .zero, size: size))
    let image = UIGraphicsGetImageFromCurrentImageContext()
    UIGraphicsEndImageContext()
    return image
  }
}
```

尽管状态被设置为 `nil`，但在惰性容器中，某些特定类型的数据（如 `Image`、`UIImage` 等）并不会立即释放其占用的内存空间，除非完全离开该惰性容器视图。这种情况下，通过转换状态类型，可以改善资源回收的积极性：

Swift 

```
struct ImageSubView: View {
  @State var data:Data?
  var body: some View {
    VStack {
      if let data,let uiImage = UIImage(data: data) {
        Image(uiImage: uiImage)
          .resizable()
          .frame(width: 200, height: 200)
      } else {
        Rectangle().fill(.gray.gradient)
          .frame(width: 200, height: 200)
      }
    }
    .task {
      // 模拟加载不同的图片，转换成的 Data 类型
      if let uiImage = await createImage() {
        data = uiImage
      }
    }
    .onDisappear {
      data = nil
    }
  }

  func createImage() async -> Data? {
    let colors: [UIColor] = [.black, .blue, .yellow, .cyan, .green, .magenta]
    let color = colors.randomElement()!
    let size = CGSize(width: 1000, height: 1000)
    UIGraphicsBeginImageContextWithOptions(size, false, 0)
    color.setFill()
    UIRectFill(CGRect(origin: .zero, size: size))
    let image = UIGraphicsGetImageFromCurrentImageContext()
    UIGraphicsEndImageContext()
    return image?.pngData()
  }
}
```

在上述代码中，通过改变状态持有的类型为 `Optional<Data>`，使得在状态设为 `nil` 后系统能够正常回收占用的资源，从而改善了内存使用情况。

> 详细的分析和问题探讨可以参见 [SwiftUI + Core Data App 的内存占用优化之旅](https://fatbobman.com/zh/posts/memory-usage-optimization/)。

通过这种方法，我们可以有效地控制内存使用，特别是在大量数据或资源被惰性视图加载时，确保应用的性能和响应速度。

> 一些读者可能会考虑利用本文介绍的惰性容器仅保留最顶层状态的特性来解决内存占用问题。然而，遗憾的是，对于这些特定的类型，即使状态被“遗忘”，它们所占用的内存也不会被及时释放。因此，这种方法并不适用于解决特定类型内存不被积极释放的问题。

## 在 List 中，子视图的顶层结构类型为 \_ConditionalContent 会破坏懒加载机制

在使用 `List` 时，许多开发者会根据数据源提供的数据进行条件判断，以显示不同的内容。这种做法很常见，但如果处理不当，可能会导致 `List` 的懒加载优化机制失效。例如，以下代码示例：

Swift 

```
struct ConditionViewDemo: View {
    var body: some View {
        List {
            ForEach(0 ..< 100) { i in
                ListSubView(i: i)
            }
        }
    }
}

struct ListSubView: View {
    let i: Int
    init(i: Int) {
        self.i = i
        print("init \(i)")
    }

    var body: some View {
        if i % 2 == 0 {
            Text("\(i) is even")
        } else {
            Text("\(i) is odd")
        }
    }
}
```

在此代码中，视图显示的内容会根据数据的奇偶性质变化。理论上，`List` 的优化机制应当仅构建略超过一屏所需的数据对应的视图。然而，在实际使用中，所有数据对应的视图都会被构建。

![](https://cdn.fatbobman.com/image-20240822105528320.webp) 

从实际观察来看，只要 `List` 的子视图的顶层结构类型为 `_ConditionalContent`，就会出现上述问题。

在文章 [ViewBuilder 研究（下）——从模仿中学习](https://fatbobman.com/zh/posts/viewbuilder2/) 中，我们介绍了视图中 `if` 或 `switch` 指令，最终都会被编译为 `_ConditionalContent` 类型。因此，以下代码也会破坏懒加载机制：

Swift 

```
// 使用 switch 语句
switch i % 2 {
case 0:
    Text("\(i) is even")
default:
    Text("\(i) is odd")
}

// 不使用 else 分支的 if 语句
if i % 2 == 0 {
    Text("\(i) is even")
}
```

目前尚不清楚为什么会出现这种现象。

不过，解决方案相对简单：只需在子视图的外部添加一个布局容器，如 `VStack`：

Swift 

```
ForEach(0 ..< 100) { i in
    VStack {
        ListSubView(i: i)
    }
}
```

需要注意的是：

* 布局容器应该添加在子视图的外层，内部添加无效。
* 必须使用具备布局功能的容器，例如 `VStack`，而 `Group` 是无效的。
* 该问题仅在 `List` 中出现，并不会对其他的惰性容器造成影响。

采用以上调整后，`List` 的懒加载机制应该可以恢复正常工作。

![](https://cdn.fatbobman.com/image-20240822111940838.webp) 

[ 马上加入 ![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)的开发者社区!  加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长! ](https://t.ly/gzxeh) 

 🚀 马上加入，享受交流的快乐 →

## 总结

虽然 SwiftUI 为开发者提供了极大的便利，使得创建动态和响应式的用户界面变得更加简单，但正确地理解和利用其惰性布局容器的工作机制，仍然是开发高效、性能优异的 SwiftUI 应用的关键。希望本文能够帮助开发者更好地掌握这些技巧，优化 SwiftUI 应用，确保既能提供丰富的功能，又能保持流畅的用户体验和高效的资源使用。

> "加入我们的 [Discord 社区](https://discord.com/invite/7FedN5E2QQ)，与超过 2000 名苹果生态的中文开发者一起交流！"

[避免 SwiftUI 视图的重复计算](https://fatbobman.com/zh/posts/avoid%5Frepeated%5Fcalculations%5Fof%5Fswiftui%5Fviews/)[优化在 SwiftUI List 中显示大数据集的响应效率](https://fatbobman.com/zh/posts/optimize%5Fthe%5Fresponse%5Fefficiency%5Fof%5Flist/)[解析 SwiftUI 中两处由状态更新滞后引发的严重 Bug](https://fatbobman.com/zh/posts/serious-issues-caused-by-delayed-state-updates-in-swiftui/)[SwiftUI 视图的生命周期研究](https://fatbobman.com/zh/posts/swiftuilifecycle/)[StateObject 与 ObservedObject](https://fatbobman.com/zh/posts/stateobject%5Fand%5Fobservedobject/)[一段因 @State 注入机制所产生的“灵异代码”](https://fatbobman.com/zh/posts/bug-code-by-state-inject/)

每周精选 Swift 与 SwiftUI 精华！

链接已复制

* [ 自定义遵循 RandomAccessCollection 的实现 ](https://fatbobman.com/zh/posts/tips-and-considerations-for-using-lazy-containers-in-swiftui/#自定义遵循-randomaccesscollection-的实现)
* [ 实现无限数据加载 ](https://fatbobman.com/zh/posts/tips-and-considerations-for-using-lazy-containers-in-swiftui/#实现无限数据加载)
* [ id 修饰器对 List 懒加载机制的影响 ](https://fatbobman.com/zh/posts/tips-and-considerations-for-using-lazy-containers-in-swiftui/#id-修饰器对-list-懒加载机制的影响)
* [ 在惰性容器中，SwiftUI 仅保留 ForEach 子视图最顶层的状态 ](https://fatbobman.com/zh/posts/tips-and-considerations-for-using-lazy-containers-in-swiftui/#在惰性容器中swiftui-仅保留-foreach-子视图最顶层的状态)
* [ 针对特定类型的状态, SwiftUI 释放内存资源不够积极 ](https://fatbobman.com/zh/posts/tips-and-considerations-for-using-lazy-containers-in-swiftui/#针对特定类型的状态-swiftui-释放内存资源不够积极)
* [ 在 List 中，子视图的顶层结构类型为 \_ConditionalContent 会破坏懒加载机制 ](https://fatbobman.com/zh/posts/tips-and-considerations-for-using-lazy-containers-in-swiftui/#在-list-中子视图的顶层结构类型为-%5Fconditionalcontent-会破坏懒加载机制)
* [ 总结 ](https://fatbobman.com/zh/posts/tips-and-considerations-for-using-lazy-containers-in-swiftui/#总结)

如何为 SwiftData 动态的构建复杂的谓词

⌥ + ←

SwiftData 实战：用现代方法构建 SwiftUI 应用

⌥ + →

[ ](https://x.com/intent/tweet?text=%E5%87%A0%E4%B8%AA%E5%9C%A8%20SwiftUI%20%E4%B8%AD%E4%BD%BF%E7%94%A8%E6%83%B0%E6%80%A7%E5%AE%B9%E5%99%A8%E7%9A%84%E6%8A%80%E5%B7%A7%E5%92%8C%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9&url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Ftips-and-considerations-for-using-lazy-containers-in-swiftui%2F&hashtags=ios%2Cswiftlang&via=fatbobman) 

Share on X

Share on BlueSky

[ ](https://www.facebook.com/sharer/sharer.php?u=https%253A%252F%252Ffatbobman.com%252Fzh%252Fposts%252Ftips-and-considerations-for-using-lazy-containers-in-swiftui%252F) 

Share on Facebook

[ ](https://service.weibo.com/share/share.php?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Ftips-and-considerations-for-using-lazy-containers-in-swiftui%2F&title=%E5%87%A0%E4%B8%AA%E5%9C%A8%20SwiftUI%20%E4%B8%AD%E4%BD%BF%E7%94%A8%E6%83%B0%E6%80%A7%E5%AE%B9%E5%99%A8%E7%9A%84%E6%8A%80%E5%B7%A7%E5%92%8C%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9%0A%20%23ios%20%23swift%0A%20by%20%40fatbobman%0A) 

Share on Weibo

[ ](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Ftips-and-considerations-for-using-lazy-containers-in-swiftui%2F) 

Share on Linkedin

[ ](https://fatbobman.com/zh/posts/tips-and-considerations-for-using-lazy-containers-in-swiftui/mailto:?subject=%E6%96%87%E7%AB%A0%3A%E3%80%90%E5%87%A0%E4%B8%AA%E5%9C%A8%20SwiftUI%20%E4%B8%AD%E4%BD%BF%E7%94%A8%E6%83%B0%E6%80%A7%E5%AE%B9%E5%99%A8%E7%9A%84%E6%8A%80%E5%B7%A7%E5%92%8C%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9%E3%80%91&body=%E5%88%86%E4%BA%AB%E4%B8%80%E7%AF%87%E6%96%87%E7%AB%A0%E7%BB%99%E4%BD%A0%EF%BC%9A%0A%0A%E5%87%A0%E4%B8%AA%E5%9C%A8%20SwiftUI%20%E4%B8%AD%E4%BD%BF%E7%94%A8%E6%83%B0%E6%80%A7%E5%AE%B9%E5%99%A8%E7%9A%84%E6%8A%80%E5%B7%A7%E5%92%8C%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9%0A%0A%E9%93%BE%E6%8E%A5%3A%20https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Ftips-and-considerations-for-using-lazy-containers-in-swiftui%2F) 

Share via Email

Share Your Thoughts

Back to Top