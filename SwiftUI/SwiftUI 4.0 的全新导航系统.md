---
title: "SwiftUI 4.0 的全新导航系统"
url: "https://fatbobman.com/zh/posts/new_navigator_of_swiftui_4/"
description: "SwiftUI 4.0 推出全新导航系统，使用 NavigationStack 和 NavigationSplitView 实现编程式导航，支持视图堆栈管理、Deep Link 跳转。了解新API！"
author: "@fatbobman"
site_name: "fatbobman.com"
language: "zh"
published_time: "2022-06-14T00:12:00.000Z"
image: "https://cdn.fatbobman.com/oc/new_navigator_of_SwiftUI_4.png"
generated_at: "2025-08-07T07:13:24.553Z"
---

[#SwiftUI](/zh/tags/SwiftUI/) [#English](/en/posts/new_navigator_of_swiftui_4/)

SwiftUI 4.0 的全新导航系统
===================

[东坡肘子](https://x.com/intent/follow?screen_name=fatbobman)

发表于 2022 年 6 月 14 日

长久以来，开发者对 SwiftUI 的导航系统颇有微词。受 NavigationView 的能力限制，开发者需要动用各种技巧乃至黑科技才能实现一些本应具备基本功能（例如：返回根视图、向堆栈添加任意视图、返回任意层级视图、Deep Link 跳转等 ）。SwiftUI 4.0（ iOS 16+ 、macOS 13+ ）对导航系统作出了重大改变，提供了以视图堆栈为管理对象的新 API ，让开发者可以轻松实现编程式导航。本文将对新的导航系统作以介绍。

[

马上加入

![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)

代码相连，创意无限 - 加入我们的开发者社区!

加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长!





](https://t.ly/gzxeh)

🚀 马上加入，享受交流的快乐 →

一分为二
----

新的导航系统最直接的变化是废弃了 NavigationView，将其功能分成了两个单独的控件 NavigationStack 和 NavigationSplitView。

NavigationStack 针对的是单栏的使用场景，例如 iPhone 、Apple TV、Apple Watch：

Swift

Copied!

    NavigationStack {}
    // 相当于
    NavigationView{}
        .navigationViewStyle(.stack)

NavigationSplitView 则针对的是多栏场景，例如 ：iPadOS 、macOS：

Swift

Copied!

    NavigationSplitView {
       SideBarView()
    } detail: {
       DetailView()
    }
    
    // 对应的是双列场景
    
    NavigationView {
        SideBarView()
        DetailView()
    }
    .navigationViewStyle(.columns)

   ![navigationSplitView_2_demo](https://cdn.fatbobman.com/navigationSplitView_2_demo.PNG)

Swift

Copied!

    NavigationSplitView {
        SideBarView()
    } content: {
        ContentView()
    } detail: {
        DetailView
    }
    
    // 对应的是三列场景
    NavigationView {
        SideBarView()
        ContentView()
        DetailView()
    }
    .navigationViewStyle(.columns)

   ![navigationSplitView_3_demo](https://cdn.fatbobman.com/navigationSplitView_3_demo.png)

相较于通过 navigationViewStyle 设定 NavigationView 样式的做法，一分为二的方式将让布局表达更加清晰，同时也会强迫开发者为 SwiftUI 应用对 iPadOS 和 macOS 做更多的适配。

> 在 iPhone 这类设备中，NavigationSplitView 会自动进行单栏适配。但是无论是切换动画、编程式 API 接口等多方面都与 NavigationStack 明显不同。因此对于支持多硬件平台的应用来说，最好针对不同的场景分别使用对应的导航控件。

两个组件两种逻辑
--------

相较于控件名称上的改变，编程式导航 API 才是本次更新的最大亮点。使用新的编程式 API ，开发者可以轻松地实现例如：返回根视图、在当前视图堆栈中添加任意视图（ 视图跳转 ）、视图外跳转（ Deep Link ）等功能。

苹果为 NavigationStack 和 NavigationSplitView 提供了两种不同逻辑的 API ，这点或许会给部分开发者造成困扰。

### NavigationView 的编程式导航

NavigationView 其实是具备一定的编程式导航能力的，比如，我们可以通过以下两种 NavigationLink 的构造方法来实现有限的编程式跳转：

Swift

Copied!

    init<S>(_ title: S, isActive: Binding<Bool>, @ViewBuilder destination: () -> Destination)
    init<S, V>(_ title: S, tag: V, selection: Binding<V?>, @ViewBuilder destination: () -> Destination)

上述两种方法有一定的局限性：

*   需要逐级视图进行绑定，开发者如想实现返回任意层级视图则需要自行管理状态
*   在声明 NavigationLink 时仍需设定目标视图，会造成不必要的实例创建开销
*   较难实现从视图外调用导航功能

“能用，但不好用” 可能就是对老版本编程式导航比较贴切地总结。

### NavigationStack

NavigationStack 从两个角度入手以解决上述问题。

#### 基于类型的响应式目标视图处理机制

比如下面的代码是在老版本（ 4.0 之前 ）SwiftUI 中使用编程式跳转的一种方式：

Swift

Copied!

    struct NavigationViewDemo: View {
        @State var selectedTarget: Target?
        @State var target: Int?
        var body: some View {
            NavigationView{
                List{
                    NavigationLink("SubView1", destination: SubView1(), tag: Target.subView1, selection: $selectedTarget) // SwiftUI 在进入当前视图时，无论是否进入目标视图，均将创建其实例（ 不对 body 求值 ）
                    NavigationLink("SubView2", destination: SubView2(), tag: Target.subView1, selection: $selectedTarget)
                    NavigationLink("SubView3", destination: SubView3(), tag: 3, selection: $target)
                    NavigationLink("SubView4", destination: SubView4(), tag: 4, selection: $target)
                }
            }
        }
    
        enum Target {
            case subView1,subView2
        }
    }

NavigationStack 实现上述功能将更加地清晰、灵活和高效。

Swift

Copied!

    struct NavigationStackDemo: View {
        var body: some View {
            NavigationStack {
                List {
                    NavigationLink("SubView1", value: Target.subView1) // 只声明关联的状态值
                    NavigationLink("SubView2", value: Target.subView2)
                    NavigationLink("SubView3", value: 3)
                    NavigationLink("SubView4", value: 4)
                }
                .navigationDestination(for: Target.self){ target in // 对同一类型进行统一处理，返回目标视图
                    switch target {
                        case .subView1:
                            SubView1()
                        case .subView2:
                            SubView2()
                    }
                }
                .navigationDestination(for: Int.self) { target in  // 为不同的类型添加多个处理模块
                    switch target {
                    case 3:
                        SubView3()
                    default:
                        SubView4()
                    }
                }
            }
        }
    
        enum Target {
            case subView1,subView2
        }
    }

NavigationStack 的处理方式有以下特点和优势：

*   由于无需在 NavigationLink 中指定目标视图，因此无须创建多余的视图实例
*   对由同一类型的值驱动的目标进行统一管理（ 可以将堆栈中所有视图的 NavigationLink 处理程序统一到根视图中 ），有利于复杂的逻辑判断，也方便剥离代码
*   NavigationLink 将优先使用最接近的类型目标管理代码。例如根视图，与第三层视图都通过 navigationDestination 定义了对 Int 的响应，那么第三层及其之上的视图将使用第三层的处理逻辑

#### 可管理的视图堆栈系统

相较于基于类型的响应式目标视图处理机制，可管理的视图堆栈系统才是新导航系统的杀手锏。

NavigationStack 支持两种堆栈管理类型：

*   NavigationPath
    
    通过添加多个的 navigationDestination ，NavigationStack 可以对多种类型值（ Hashable ）进行响应，使用 `removeLast(_ k: Int = 1)` 返回指定的层级，使用 `append` 进入新的层级
    

Swift

Copied!

    class PathManager:ObservableObject{
        @Published var path = NavigationPath()
    }
    
    struct NavigationViewDemo1: View {
        @StateObject var pathManager = PathManager()
        var body: some View {
            NavigationStack(path:$pathManager.path) {
                List {
                    NavigationLink("SubView1", value: 1)
                    NavigationLink("SubView2", value: Target.subView2)
                    NavigationLink("SubView3", value: 3)
                    NavigationLink("SubView4", value: 4)
                }
                .navigationDestination(for: Target.self) { target in
                    switch target {
                    case .subView1:
                        SubView1()
                    case .subView2:
                        SubView2()
                    }
                }
                .navigationDestination(for: Int.self) { target in
                    switch target {
                    case 1:
                        SubView1()
                    case 3:
                        SubView3()
                    default:
                        SubView4()
                    }
                }
            }
            .environmentObject(pathManager)
            .task{
                // 使用 append 可以跳入指定层级，下面将为 root -> SubView3 -> SubView1 -> SubView2 ，在初始状态添加层级将屏蔽动画
                pathManager.path.append(3)
                pathManager.path.append(1)
                pathManager.path.append(Target.subView2)
            }
        }
    }
    
    enum Target {
        case subView1, subView2
    }
    
    struct SubView1: View {
        @EnvironmentObject var pathManager:PathManager
        var body: some View {
            List{
                // 仍然可以使用此种形式的 NavigationLink，目标视图的处理在根视图对应的 navigationDestination 中
                NavigationLink("SubView2", destination: Target.subView2 )
                NavigationLink("subView3",value: 3)
                Button("go to SubView3"){
                    pathManager.path.append(3) // 效果与上面的 NavigationLink("subView3",value: 3) 一样
                }
                Button("返回根视图"){
                    pathManager.path.removeLast(pathManager.path.count)       
                }
                Button("返回上层视图"){
                    pathManager.path.removeLast() 
                }
            }
        }
    }

*   元素为符合 Hashable 的单一类型序列
    
    采用此种堆栈，NavigationStack 将只能响应该序列元素的特定类型
    

Swift

Copied!

    class PathManager:ObservableObject{
        @Published var path:[Int] = [] // Hashable 序列
    }
    
    struct NavigationViewDemo1: View {
        @StateObject var pathManager = PathManager()
        var body: some View {
            NavigationStack(path:$pathManager.path) {
                List {
                    NavigationLink("SubView1", value: 1)
                    NavigationLink("SubView3", value: 3)
                    NavigationLink("SubView4", value: 4)
                }
                // 只能响应序列元素类型
                .navigationDestination(for: Int.self) { target in
                    switch target {
                    case 1:
                        SubView1()
                    case 3:
                        SubView3()
                    default:
                        SubView4()
                    }
                }
            }
            .environmentObject(pathManager)
            .task{
                pathManager.path = [3,4]  // 直接跳转到指定层级，赋值更加方便
            }
        }
    }
    
    struct SubView1: View {
        @EnvironmentObject var pathManager:PathManager
        var body: some View {
            List{
                NavigationLink("subView3",value: 3)
                Button("go to SubView3"){
                    pathManager.path.append(3) // 效果与上面的 NavigationLink("subView3",value: 3) 一样
                }
                Button("返回根视图"){
                    pathManager.path.removeAll()
                }
                Button("返回上层视图"){
                    if pathManager.path.count > 0 {
                        pathManager.path.removeLast()
                    }
                }
                Button("响应 Deep Link，重置 Path Stack "){
                    pathManager.path = [3,1,1] // 会自动屏蔽动画
                }
                
            }
        }
    }

开发者可以根据自己的需求选择对应的视图堆栈类型。

> ⚠️ 在使用堆栈管理系统的情况下，请不要在编程式导航中混用声明式导航，这样会破坏当前的视图堆栈数据

下面的代码，如果点击声明式导航，将导致堆栈数据重置。

Swift

Copied!

    NavigationLink("SubView3",value: 3)
    NavigationLink("SubView4", destination: { SubView4() }) // 不要在编程式导航中混用声明式导航

### NavigationSplitView

如果说 NavigationStack 是在三维的空间里堆叠视图，那么 NavigationSplitView 便是在二维的空间中于不同的栏之间动态切换视图。

#### 分栏布局

在 SwiftUI 4.0 之前的版本，可以这样使用 NavigationView 来创建拥有左右两个栏的编程式导航视图：

Swift

Copied!

    class MyStore: ObservableObject {
        @Published var selection: Int?
    }
    
    struct NavigationViewDoubleColumnView: View {
        @StateObject var store = MyStore()
        var body: some View {
            NavigationView {
                SideBarView()
                DetailView()
            }
            .environmentObject(store)
        }
    }
    
    struct SideBarView: View {
        @EnvironmentObject var store: MyStore
        var body: some View {
            List(0..<30, id: \.self) { i in
                // 此处我们没有使用 NavigationLink 来切换右侧视图，而是改变了 seletion 的值，让右侧视图响应该值的变化                      
                Button("ID: \(i)") {
                    store.selection = i
                }
            }
        }
    }
    
    struct DetailView: View {
        @EnvironmentObject var store: MyStore
        var body: some View {
            if let selection = store.selection {
                Text("视图：\(selection)")
            } else {
                Text("请选择")
            }
        }
    }

   ![double_colunm_2022-06-11_10.16.38](https://cdn.fatbobman.com/double_colunm_2022-06-11_10.16.38.png)

用 NavigationSplitView 实现上面的代码基本上一样。最大的区别是，SwiftUI 4.0 为我们提供了在 NavigationSplitView 中通过 List 快速绑定数据的能力。

Swift

Copied!

    struct NavigationSplitViewDoubleColumnView: View {
        @StateObject var store = MyStore()
        var body: some View {
            NavigationSplitView {
                SideBarView()
            } detail: {
                DetailView()
            }
            .environmentObject(store)
        }
    }
    
    struct SideBarView: View {
        @EnvironmentObject var store: MyStore
        var body: some View {
            // 可以在 List 中直接绑定数据，无需通过 Button 显式进行修改
            List(0..<30, id: \.self, selection: $store.selection) { i in
                NavigationLink("ID: \(i)", value: i)  // 使用编程式的 NavigationLink
            }
        }
    }

   ![image-20220611104123815](https://cdn.fatbobman.com/image-20220611104123815.png)

由于 SwiftUI 4.0 为 List 提供了进一步的加强，我们还可以不使用 NavigationLink ，改写成下面的代码：

Swift

Copied!

    struct SideBarView: View {
        @EnvironmentObject var store: MyStore
        var body: some View {
            List(0..<30, id: \.self, selection: $store.selection) { i in
                Text("ID: \(i)")  // 也可以换成 Label 或其他视图 ，但不能是 Button
    //            NavigationLink("ID: \(i)", value: i)
            }
        }
    }

> SwiftUI 4.0 中，在 List 绑定了数据后，通过 List 构造方法创建的循环或 ForEach 创建的循环中的内容（ 不能自带点击属性，例如 Button 或 onTapGesture ），将被隐式添加 tag 修饰符，从而具备点击后可更改绑定数据的能力

无论将 List 放置在 NavigationSplitView 的最左侧一栏（ 双栏模式 ）还是左侧两栏中（ 三栏模式 ），都可以通过 List 的绑定数据进行导航。这是 NavigationSplitView 的独有功能。

> 从 iOS 16.1 开始，开发者可以无需通过 List 的绑定模式来跳转视图了。通过在 NavigationSplitView 侧边栏里放一个. navigationDestination，这样侧边栏里的 NavigationLink 就会取代详细栏的根视图。

Swift

Copied!

    NavigationSplitView {
        LazyVStack {
            NavigationLink("link", value: 213)
        }
        .navigationDestination(for: Int.self) { i in
            Text("The value is \(i)")
        }
    } detail: {
        Text("Click an item")
    }

#### 与 NavigationStack 合作

在 SwiftUI 4.0 之前，对于多栏的 NavigationView ，如果我们想在 SideBar 栏内实现堆栈跳转的话，可以使用如下代码：

Swift

Copied!

    struct SideBarView: View {
        @EnvironmentObject var store: MyStore
        var body: some View {
            List(0..<30, id: \.self) { i in
                NavigationLink("ID: \(i)", destination: Text("\(i)")) // 必须使用 NavigationLink
                    .isDetailLink(false) // 指定 destination 不要显示在 Detail 列中
            }
        }
    }

但如果，我们想在 Detail 栏中也想嵌入一个可以实现堆栈跳转的 NavigationView 则会有很大的问题。此时在 Detail 栏中将出现两个 NavigationTitle 以及两个 Toolbar 。

Swift

Copied!

    struct NavigationViewDoubleColumnView: View {
        @StateObject var store = MyStore()
        var body: some View {
            NavigationView {
                SideBarView()
                DetailView()
                    .navigationTitle("Detail")  // 为 Detail 栏定义 title
                    .toolbar{
                        EditButton()  // 在 Detail 栏创建按钮
                    }
            }
            .environmentObject(store)
        }
    }
    
    struct SideBarView: View {
        @EnvironmentObject var store: MyStore
        var body: some View {
            List(0..<30, id: \.self) { i in
                Button("ID: \(i)") {
                    store.selection = i
                }
            }
        }
    }
    
    struct DetailView: View {
        @EnvironmentObject var store: MyStore
        var body: some View {
            NavigationView {
                VStack {
                    if let selection = store.selection {
                        NavigationLink("查看详情", destination: Text("\(selection)"))
                    } else {
                        Text("请选择")
                    }
                }
                .toolbar{
                    EditButton()  // 在 Detail 栏中的 NavigationView 创建按钮
                }
                .navigationTitle("Detail") // 为 Detail 栏中的 NavigationView 定义 Title
                .navigationBarTitleDisplayMode(.inline)
            }
            .navigationViewStyle(.stack)
        }
    }

   ![image-20220611110657857](https://cdn.fatbobman.com/image-20220611110657857.png)

> 为此，我之前不得已在 iPad 版本的应用程序中，使用 HStack 来避免出现上述问题。详情请参阅 [在 SwiftUI 下对 iPad 进行适配](/zh/posts/swiftui-ipad/)

NavigationSpiteView 已经解决了上述问题，它现在可以同 NavigationStack 进行完美的合作。

Swift

Copied!

    class MyStore: ObservableObject {
        @Published var selection: Int?
    }
    
    struct NavigationSplitViewDoubleColumnView: View {
        @StateObject var store = MyStore()
        var body: some View {
            NavigationSplitView {
                SideBarView()
            } detail: {
                DetailView()
                    .toolbar {
                        EditButton() // 在 Detail 栏中的 NavigationView 创建按钮
                    }
                    .navigationTitle("Detail")
            }
            .environmentObject(store)
        }
    }
    
    struct SideBarView: View {
        @EnvironmentObject var store: MyStore
        var body: some View {
            List(0..<30, id: \.self, selection: $store.selection) { i in
                Text("ID: \(i)")
            }
        }
    }
    
    struct DetailView: View {
        @EnvironmentObject var store: MyStore
        var body: some View {
            NavigationStack {
                VStack {
                    if let selection = store.selection {
                        NavigationLink("查看详情", value: selection)
                    } else {
                        Text("请选择")
                    }
                }
                .navigationDestination(for: Int.self, destination: {
                    Text("\($0)")
                })
                .toolbar {
                    RenameButton() // 在 Detail 栏中的 NavigationView 创建按钮 
                }
                .navigationTitle("Detail inLine")
                .navigationBarTitleDisplayMode(.inline)
            }
        }
    }

NavigationSplitView 会保留最近的 Title 设定，并对分别在 NavigationSplitView 和 NavigationStack 中为 Detail 栏添加的 Toolbar 按钮进行合并。

   ![image-20220611134247340](https://cdn.fatbobman.com/image-20220611134247340.png)

通过在 NavigationSplitView 中使用 NavigationStack ，开发者拥有了更加丰富的视图调度能力。

#### 动态控制多栏显示状态

另一个之前困扰多栏 NavigationView 的问题就是，无法通过编程的手段动态地控制多栏显示状态。NavigationSplitView 在构造方法中提供了 columnVisibility 参数 （ NavigationSplitViewVisibility 类型 ），通过设置该参数，开发者拥有了对导航栏显示状态的控制能力。

Swift

Copied!

    struct NavigationSplitViewDoubleColumnView: View {
        @StateObject var store = MyStore()
        @State var mode: NavigationSplitViewVisibility = .all 
        var body: some View {
            NavigationSplitView(columnVisibility: $mode) {
                SideBarView()
            }
        content: {
                ContentColumnView()
            }
        detail: {
                DetailView()
            }
            .environmentObject(store)
        }
    }

   ![three_column_2022-06-11_13.52.10](https://cdn.fatbobman.com/three_column_2022-06-11_13.52.10.png)

*   detailOnly 只显示 Detail 栏（ 最右侧栏 ）
*   doubleColumn 在三栏状态下隐藏 Sidebar （ 最左侧 ）栏
*   all 显示所有的栏
*   automatic 根据当前的上下文自动决定显示行为

_上述选项并非适用于所有的平台，例如，在 macOS 上，detalOnly 不会起作用_

> 如果想在 SwiftUI 4.0 之前的版本上使用类似的功能，可以参考我在 [用 NavigationViewKit 增强 SwiftUI 的导航视图](/zh/posts/navigationviewkit/) 一文中的实现方法

其他增强
----

除了上述的功能，新的导航系统还在很多其他的地方也进行了增强。

### 设置栏宽度

NavigationSplitView 为栏中的视图提供了一个新的修饰符 navigationSplitViewColumnWidth ，通过它开发者可以修改栏的默认宽度：

Swift

Copied!

    struct NavigationSplitViewDemo: View {
        @State var mode: NavigationSplitViewVisibility = .all
        var body: some View {
            NavigationSplitView(columnVisibility: $mode) {
                SideBarView()
                    .navigationSplitViewColumnWidth(200)
            }
        content: {
                ContentColumnView()
                    .navigationSplitViewColumnWidth(min: 100, ideal: 150, max: 200)
            }
        detail: {
                DetailView()
            }
        }
    }

### 设置 NavigationSplitView 的样式

使用 navigationSplitViewStyle 可以设置 NavigationSplitView 的样式

Swift

Copied!

    struct NavigationSplitViewDemo: View {
        @State var mode: NavigationSplitViewVisibility = .all
        var body: some View {
            NavigationSplitView(columnVisibility: $mode) {
                SideBarView()
            }
        content: {
                ContentColumnView()
            }
        detail: {
                DetailView()
            }
            .navigationSplitViewStyle(.balanced) // 设置样式
        }
    }

*   prominentDetail
    
    无论左侧栏显示与否，保持右侧的 Detail 栏尺寸不变（ 通常是全屏 ）。iPad 在 Portrait 显示状态下，默认即为此种模式
    
*   balanced
    
    在显示左侧栏的时候，缩小右侧 Detail 栏的尺寸。iPad 在 landscape 显示状态下，默认即为此种模式
    
*   automatic
    
    默认值，根据上下文自动调整外观样式
    

### 在 NavigationTitle 中添加菜单

使用新的 navigationTitle 构造方法，可以将菜单嵌入到标题栏中。

Swift

Copied!

    .navigationTitle( Text("Setting"), actions: {
                    Button("Action1"){}
                    Button("Action2"){}
                    Button("Action3"){}
                })

   ![image-20220612085945286](https://cdn.fatbobman.com/image-20220612085945286.png)

### 更改 NavigationBar 背景色

Swift

Copied!

    NavigationStack{
        List(0..<30,id:\.self){ i in
            Text("\(i)")
        }
        .listStyle(.plain)
        .navigationTitle("Hello")
        .toolbarBackground(.pink, in: .navigationBar)
    }

   ![RocketSim_Screenshot_iPhone_13_Pro_Max_2022-06-12_09.12.01](https://cdn.fatbobman.com/RocketSim_Screenshot_iPhone_13_Pro_Max_2022-06-12_09.12.01.png)

NavigationStack 的 toolbar 背景色只有在视图上滚时才会显示。

> SwiftUI 4.0 中，将 toolbar 的认定范围扩大到了 TabView 。在 toolbar 的设置中，通过 placement 可以设置适用的对象

### 隐藏 toolbar

Swift

Copied!

    NavigationStack {
        ContentView()
            .toolbar(.hidden, in: .navigationBar)
    }

### 设置 toolbar 的色彩外观（ Color Scheme ）

Swift

Copied!

    .toolbarColorScheme(.dark, for: .navigationBar)

   ![RocketSim_Screenshot_iPhone_13_Pro_Max_2022-06-12_09.21.29](https://cdn.fatbobman.com/RocketSim_Screenshot_iPhone_13_Pro_Max_2022-06-12_09.21.29.png)

### Toolbar 角色

使用 toolbarRole 设置当前 toolbar 的角色身份。不同的角色将让 toolbar 的外观和排版有所不同（ 视设备而异 ）。

Swift

Copied!

    struct ToolRoleTest: View {
        var body: some View {
            NavigationStack {
                List(0..<10, id: \.self) {
                    NavigationLink("ID: \($0)", value: $0)
                }
                .navigationDestination(for: Int.self) {
                    Text("\($0)")
                        .navigationTitle("Title for \($0)")
                        .toolbarRole(.editor)
                }
                .navigationTitle("Title")
                .navigationBarTitleDisplayMode(.inline)
                .toolbarRole(.browser)
                .toolbar {
                    ToolbarItem(placement: .primaryAction) {
                        EditButton()
                    }
                }
            }
        }
    }

*   navigationStack
    
    默认角色，长按可显示视图堆栈列表
    
*   browser
    
    在 iPad 下，当前视图的 Title 将显示在左侧
    

   ![image-20220612190914949](https://cdn.fatbobman.com/image-20220612190914949.png)

*   editor
    
    不显示返回按钮旁边的上页视图 Title
    

   ![image-20220612191040190](https://cdn.fatbobman.com/image-20220612191040190.png)

### 定制 NavigationLink 样式

在之前版本的 SwiftUI 中，NavigationLink 其实一直都是作为一种特殊的 Button 存在的。到了 SwiftUI 4.0 版本后，SwiftUI 已经将其真正的视为了 Button 。

Swift

Copied!

    NavigationStack {
        VStack {
            NavigationLink("Hello world", value: "sub1")
                .buttonStyle(.bordered)
                .controlSize(.large)
            NavigationLink("Goto next", destination: Text("Next"))
                .buttonStyle(.borderedProminent)
                .controlSize(.large)
                .tint(.red)
        }
        .navigationDestination(for: String.self){ _ in
            Text("Sub View")
        }
    }

   ![image-20220613220926715](https://cdn.fatbobman.com/image-20220613220926715.png)

[

马上加入

![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)

代码相连，创意无限 - 加入我们的开发者社区!

加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长!





](https://t.ly/gzxeh)

🚀 马上加入，享受交流的快乐 →

总结
--

SwiftUI 4.0 导航系统的变化如此之大，开发者在惊喜的同时，也要冷静的面对事实。相当一部分开发者由于版本适配的原因并不会使用新的 API ，因此，每个人都需要认真考虑如下问题：

*   如何从新 API 中获得灵感
*   如何在老版本中运用编程式导航思想
*   如何让新老版本的程序都能享受系统提供的便利

另一方面，新导航系统也向每一个开发者传递了明确的信号，苹果希望应用能够为 iPad 和 macOS 提供更加符合各自设备特点的 UI 界面。这种信号会越来越强，苹果也为此会提供越来越多的 API。

> 目前已经有人实现了 NavigationStack 在低版本 SwiftUI 下的仿制品 —— [NavigationBackport](https://github.com/johnpatrickmorgan/NavigationBackport?utm_source=Fatbobman%20Blog&utm_medium=web) ，有兴趣的朋友可以参考作者的实现方式

> "加入我们的 [Discord 社区](https://discord.com/invite/7FedN5E2QQ)，与超过 2000 名苹果生态的中文开发者一起交流！"

[在 SwiftUI 中创建自适应的程序化导航方案](/zh/posts/adaptive-navigation-scheme/)[在 SwiftUI 下对 iPad 进行适配](/zh/posts/swiftui-ipad/)[在多包项目中统一管理资源](/zh/posts/unified_management_of_resources_in_multi-package_projects/)[解析 SwiftUI 中两处由状态更新滞后引发的严重 Bug](/zh/posts/serious-issues-caused-by-delayed-state-updates-in-swiftui/)[onAppear 的调用时机](/zh/posts/onappear-call-timing/)

每周精选 Swift 与 SwiftUI 精华！

链接已复制

.copy-icon { position: absolute; left: -30px; /\* 根据您的布局调整位置 \*/ top: 50%; /\* 将图标定位到标题的中间高度 \*/ transform: translateY(-50%); /\* 垂直居中对齐图标 \*/ cursor: pointer; transition: opacity 0.6s ease; pointer-events: all; /\* 始终响应鼠标事件 \*/ } article h2, article h3, article h4, article h5 { position: relative; } #copy-alert { position: fixed; bottom: -100px; /\* 初始在视窗下方 \*/ right: 20px; transition: bottom 0.5s ease-in-out; } /\* 进入动画 \*/ @keyframes slideIn { from { bottom: -100px; } to { bottom: 0px; } } /\* 消失动画 \*/ @keyframes slideOut { from { bottom: 20px; } to { bottom: -100px; } } var hideTimeout; // 用于保存自动隐藏定时器的引用 document.querySelectorAll('article h2, article h3, article h4, article h5').forEach(function (header) { var copyIcon = document.createElement('span'); copyIcon.innerHTML = '<svg fill="currentColor" aria-hidden="true" stroke="currentColor" class="w-5 h-5 hidden md:block hover:text-secondary" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg"><path d="M861.866667 164.266667c-87.466667-87.466667-230.4-89.6-320-2.133334l-68.266667 68.266667c-12.8 12.8-12.8 32 0 44.8s32 12.8 44.8 0l68.266667-68.266667c64-61.866667 166.4-61.866667 230.4 2.133334 64 64 64 168.533333 2.133333 232.533333l-117.333333 119.466667c-34.133333 34.133333-81.066667 51.2-128 49.066666-46.933333-4.266667-91.733333-27.733333-119.466667-66.133333-10.666667-14.933333-29.866667-17.066667-44.8-6.4-14.933333 10.666667-17.066667 29.866667-6.4 44.8 40.533333 53.333333 100.266667 87.466667 166.4 91.733333h17.066667c59.733333 0 119.466667-23.466667 162.133333-68.266666l117.333333-119.466667c83.2-89.6 83.2-234.666667-4.266666-322.133333z"></path><path d="M505.6 750.933333l-66.133333 68.266667c-64 61.866667-166.4 61.866667-230.4-2.133333-64-64-64-168.533333-2.133334-232.533334l117.333334-119.466666c34.133333-34.133333 81.066667-51.2 128-49.066667 46.933333 4.266667 91.733333 27.733333 119.466666 66.133333 10.666667 14.933333 29.866667 17.066667 44.8 6.4 14.933333-10.666667 17.066667-29.866667 6.4-44.8-40.533333-53.333333-100.266667-87.466667-166.4-91.733333-66.133333-4.266667-130.133333 19.2-177.066666 66.133333l-117.333334 119.466667c-85.333333 89.6-85.333333 234.666667 2.133334 322.133333 44.8 44.8 102.4 66.133333 162.133333 66.133334 57.6 0 115.2-21.333333 160-64l66.133333-68.266667c12.8-12.8 12.8-32 0-44.8-14.933333-10.666667-34.133333-10.666667-46.933333 2.133333z"></path></svg>'; // 使用您选择的图标 copyIcon.className = 'copy-icon'; copyIcon.style.opacity = '0'; // 默认透明 header.insertBefore(copyIcon, header.firstChild); header.addEventListener('mouseover', function () { copyIcon.style.opacity = '1'; }); header.addEventListener('mouseout', function () { copyIcon.style.opacity = '0'; }); copyIcon.addEventListener('click', function () { var url = window.location.href.split('#')\[0\] + '#' + header.id; navigator.clipboard.writeText(url).then(function () { var alertDiv = document.getElementById('copy-alert'); if (alertDiv) { alertDiv.classList.remove('hidden'); // 确保提示框是可见的 alertDiv.style.display = 'block'; // 显示提示框 clearTimeout(hideTimeout); // 清除之前的定时器 alertDiv.style.animation = 'none'; // 先清除之前的动画 setTimeout(function () { alertDiv.style.animation = 'slideIn 0.5s forwards'; // 应用新动画 }, 10); // 设置自动隐藏定时器 hideTimeout = setTimeout(function () { alertDiv.style.animation = 'slideOut 0.5s forwards'; // 应用消失动画 setTimeout(function () { alertDiv.style.display = 'none'; }, 500); }, 3000); } }); }); }); function copyCode(button, event) { event.preventDefault(); const codeBlock = button.parentNode.nextElementSibling.textContent; // 获取代码内容 navigator.clipboard.writeText(codeBlock).then(() => { // 显示 "Copied" button.querySelector('.copy-text').style.display = 'none'; button.querySelector('.copied-text').style.display = 'inline'; // 3秒后恢复 setTimeout(() => { button.querySelector('.copy-text').style.display = 'inline'; button.querySelector('.copied-text').style.display = 'none'; }, 3000); }); } (function(){const url = "https://fatbobman.com/ads/zh"; // 根据中英文对应不同的 adCacheKey const adCacheKey = 'dynamicAdCache' + (window.location.pathname.includes('/zh/') ? 'ZH' : 'EN'); // 尝试从缓存中获取广告 const cachedAd = JSON.parse(localStorage.getItem(adCacheKey)); const now = Date.now(); if (cachedAd && cachedAd.expiresAt > now) { console.log('Using cached ad'); console.log(cachedAd.expiresAt - now); // 如果缓存有效，直接使用缓存的广告 renderAd(cachedAd.content); } else { console.log('Fetching ad from the server'); // 如果缓存无效，从服务器获取广告 fetch(url, { headers: { 'Accept': 'text/html', }, cache: 'default' }) .then(response => { const maxAgeMatch = response.headers.get('Cache-Control')?.match(/max-age=(\\d+)/); const maxAge = maxAgeMatch ? parseInt(maxAgeMatch\[1\], 10) : 0; if (!response.ok || maxAge <= 0) { throw new Error('Invalid response or missing Cache-Control'); } return response.text().then(adContent => { const expiresAt = now + maxAge \* 1000; // 计算失效时间 // 缓存广告内容 localStorage.setItem(adCacheKey, JSON.stringify({ content: adContent, expiresAt })); // 渲染广告 renderAd(adContent); }); }) .catch(error => console.error('Error loading the ad:', error)); } // 渲染广告函数 function renderAd(html) { const adSlots = document.querySelectorAll('.dynamic-ad-container'); adSlots.forEach(slot => { slot.innerHTML = html; }); } })(); var c=document.querySelectorAll("article");c.forEach(function(r){var a=r.querySelectorAll("a");a.forEach(function(e){var t=e.getAttribute("href");t&&t.startsWith("http")&&!t.includes("fatbobman.com")&&!t.includes("fatbobman.github.com")&&!t.includes("localhost")&&e.setAttribute("target","\_blank")})});

document.addEventListener('DOMContentLoaded', async function () { const container = document.getElementById('custom-substack-embed'); let anchor = ''; if (container) { anchor = container.getAttribute('data-anchor') || ''; } // 提前检查是否需要加载评论 try { const response = await fetch('https://fatbobman.com/status/comment'); const data = await response.text(); console.log(data, "comment"); if (data !== '1') { console.log('评论功能未启用，跳过监控和加载操作。'); return; // 提前返回，跳过后续逻辑 } } catch (error) { console.error('检查评论状态时出错：', error); return; // 出现错误时也跳过后续逻辑 } // 如果返回值为 "1"，则启动观察器 const observerTarget = document.getElementById(anchor); if (observerTarget) { const observer = new IntersectionObserver(async (entries, observer) => { const entry = entries\[0\]; if (entry.isIntersecting) { console.log("load comment") // 停止观察，避免重复触发 observer.unobserve(observerTarget); // 动态加载评论区脚本 await loadGitalk(); } }, { root: null, // 相对于视口 rootMargin: '20%', // 提前 20% 的位置触发 threshold: 0.1, // 元素显示 10% 即触发 }); observer.observe(observerTarget); } // 动态加载 Gitalk 脚本并初始化评论区 async function loadGitalk() { const container = document.getElementById('gitalk-container'); if (container) { const commentID = container.getAttribute('data-comment-id'); const lang = container.getAttribute('data-lang'); const repo = container.getAttribute('data-repo') const createIssueManually = container.getAttribute('data-createIssueManually') console.log(repo); console.log(createIssueManually) // 动态加载 Gitalk 脚本 const script = document.createElement('script'); script.src = 'https://cdn.jsdelivr.net/npm/gitalk@1/dist/gitalk.min.js'; script.defer = true; script.onload = () => { const gitalk = new Gitalk({ clientID: 'fcf61195c7f73253dc8b', clientSecret: '0ac2907be08248a1fcb5312e27480ad535c682e5', repo: repo, owner: 'fatbobman', admin: \['fatbobman'\], id: commentID.split('/').pop().substring(0, 49), distractionFreeMode: true, createIssueManually: createIssueManually, language: lang, }); gitalk.render('gitalk-container'); }; document.body.appendChild(script); } } });

*   [一分为二](#一分为二)
    
*   [两个组件两种逻辑](#两个组件两种逻辑)
    
*   [NavigationView 的编程式导航](#navigationview-的编程式导航)
    
*   [NavigationStack](#navigationstack)
    
*   [NavigationSplitView](#navigationsplitview)
    
*   [其他增强](#其他增强)
    
*   [设置栏宽度](#设置栏宽度)
    
*   [设置 NavigationSplitView 的样式](#设置-navigationsplitview-的样式)
    
*   [在 NavigationTitle 中添加菜单](#在-navigationtitle-中添加菜单)
    
*   [更改 NavigationBar 背景色](#更改-navigationbar-背景色)
    
*   [隐藏 toolbar](#隐藏-toolbar)
    
*   [设置 toolbar 的色彩外观（ Color Scheme ）](#设置-toolbar-的色彩外观-color-scheme)
    
*   [Toolbar 角色](#toolbar-角色)
    
*   [定制 NavigationLink 样式](#定制-navigationlink-样式)
    
*   [总结](#总结)
    

如何在 Core Data 中进行批量操作

⌥ + ←

(function(){const url = "/zh/posts/batchprocessingincoredata/"; const isHidden = false; document.addEventListener('keydown', (e) => { // Check if Alt+RightArrow or just RightArrow was pressed if (e.altKey && e.key === 'ArrowLeft') { // Only navigate if there's a URL and the button isn't hidden if (url && !isHidden) { e.preventDefault(); // Prevent default browser behavior window.location.href = url; } } }); })();

用 Table 在 SwiftUI 下创建表格

⌥ + →

(function(){const url = "/zh/posts/table\_in\_swiftui/"; const isHidden = false; document.addEventListener('keydown', (e) => { // Check if Alt+RightArrow or just RightArrow was pressed if (e.altKey && e.key === 'ArrowRight') { // Only navigate if there's a URL and the button isn't hidden if (url && !isHidden) { e.preventDefault(); // Prevent default browser behavior window.location.href = url; } } }); })();

[](https://x.com/intent/tweet?text=SwiftUI%204.0%20%E7%9A%84%E5%85%A8%E6%96%B0%E5%AF%BC%E8%88%AA%E7%B3%BB%E7%BB%9F&url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fnew_navigator_of_swiftui_4%2F&hashtags=ios%2Cswiftlang&via=fatbobman)

Share on X

Share on BlueSky

[](https://www.facebook.com/sharer/sharer.php?u=https%253A%252F%252Ffatbobman.com%252Fzh%252Fposts%252Fnew_navigator_of_swiftui_4%252F)

Share on Facebook

[](https://service.weibo.com/share/share.php?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fnew_navigator_of_swiftui_4%2F&title=SwiftUI%204.0%20%E7%9A%84%E5%85%A8%E6%96%B0%E5%AF%BC%E8%88%AA%E7%B3%BB%E7%BB%9F%0A%20%23ios%20%23swift%0A%20by%20%40fatbobman%0A)

Share on Weibo

[](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fnew_navigator_of_swiftui_4%2F)

Share on Linkedin

[](mailto:?subject=%E6%96%87%E7%AB%A0%3A%E3%80%90SwiftUI%204.0%20%E7%9A%84%E5%85%A8%E6%96%B0%E5%AF%BC%E8%88%AA%E7%B3%BB%E7%BB%9F%E3%80%91&body=%E5%88%86%E4%BA%AB%E4%B8%80%E7%AF%87%E6%96%87%E7%AB%A0%E7%BB%99%E4%BD%A0%EF%BC%9A%0A%0ASwiftUI%204.0%20%E7%9A%84%E5%85%A8%E6%96%B0%E5%AF%BC%E8%88%AA%E7%B3%BB%E7%BB%9F%0A%0A%E9%93%BE%E6%8E%A5%3A%20https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fnew_navigator_of_swiftui_4%2F)

Share via Email

Share Your Thoughts

const SCROLL\_OFFSET = 80; document.addEventListener('DOMContentLoaded', function() { const commentButton = document.getElementById('comment-button'); if (commentButton) { commentButton.addEventListener('click', function() { const gitalkContainer = document.getElementById('gitalk-container'); if (gitalkContainer) { const topPosition = gitalkContainer.getBoundingClientRect().top + window.scrollY - SCROLL\_OFFSET; window.scrollTo({ top: topPosition, behavior: 'smooth' }); } }); } });

Back to Top