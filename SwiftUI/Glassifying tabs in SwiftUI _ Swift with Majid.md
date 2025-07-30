## Glassifying tabs in SwiftUI

24 Jun 2025 

One of the most important changes presented during WWDC 25 was the new design language used across all Apple platforms called Liquid Glass. Tabs play a significant role in the new design and provide new ways of interacting with them. This week, we will learn about new APIs that SwiftUI provides us to handle new tab interactions.

**Enhancing the Xcode Simulators.**   
Compare designs, show rulers, add a grid, quick actions for recent builds. Create recordings with touches & audio, trim and export them into MP4 or GIF and share them anywhere using drag & drop. Add bezels to screenshots and videos.[Try now](https://gumroad.com/a/931293139/ftvbh) 

SwiftUI introduced a slightly new API for managing tabs earlier. This year, that API was adopted to the new Liquid Glass design. You still can use the previous API, but in that case, you can’t take advantage of all the new interactions. So, I suggest you migrate to the new Tab API if you support iOS 18 and later.

```
struct ContentView: View {
    var body: some View {
        TabView {
            Tab("feed", systemImage: "list.star") {
                // content
            }
            
            Tab("settings", systemImage: "gear") {
                // content
            }
        }
    }
}

```

As you can see in the example above, we still use the old_TabView_, but instead of placing tab content inside the_TabView_we wrap it with_Tab_type. We can also programmatically control the tab selection by using_TabView_with_Tab_in pair.

```
struct ContentView: View {
    @SceneStorage("selectedTab") private var selectedTabIndex = 0
    
    var body: some View {
        TabView(selection: $selectedTabIndex) {
            Tab("feed", systemImage: "list.star", value: 0) {
                // content
            }
            
            Tab("settings", systemImage: "gear", value: 1) {
                // content
            }
        }
    }
}

```

This way we can bind the tab selection to the_selectedTabIndex_which uses the_SceneStorage_property wrapper to rescue the selected tab while state restoration of the app.

```
struct ContentView: View {
    @SceneStorage("selectedTab") private var selectedTabIndex = 0
    
    var body: some View {
        TabView(selection: $selectedTabIndex) {
            Tab("feed", systemImage: "ruler", value: 0) {
                // content
            }
            
            // Other tabs
            
            Tab("search", systemImage: "magnifyingglass", value: 1, role: .search) {
                // content
            }
            
            Tab("settings", systemImage: "ruler", value: 2) {
                // content
            }
        }
    }
}

```

The_Tab_type also provides us the_role_parameter which we can use to set a specific role on the tab. At the moment, the only available instance of the_TabRole_type is the_search_. The search role allows the system to display search tab in a different way. For example, it is separated on iOS from all other tabs.

![](https://swiftwithmajid.com/public/glassy-tabs.png)

Most of my apps use tabs on iOS and watchOS while using sidebar navigation on macOS and iPadOS. SwiftUI simplifies this significantly by introducing the new_sidebarAdaptable_style. You don’t need to manually create the sidebar, it automatically replaces tabs with sidebar on iPadOS and macOS.

```
struct ContentView: View {
    @SceneStorage("selectedTab") private var selectedTabIndex = 0
    
    var body: some View {
        TabView(selection: $selectedTabIndex) {
            Tab("feed", systemImage: "list.star", value: 0) {
                // content
            }
            
            Tab("settings", systemImage: "gear", value: 2) {
                // content
            }
        }
        .tabViewStyle(.sidebarAdaptable)
    }
}

```

As you can see, all you need to do is use the_tabViewStyle_view modifier with the_sidebarAdaptable_value. Another interesting design concept that the Liquid Glass introduced is the tab accessory view. Tab accessory view is displayed on top of tabs and is always visible. It is approachable for global status or global action. For example, it indicates the current playing song in Apple Music apps and controls to pause or skip it.

```
struct ContentView: View {
    @SceneStorage("selectedTab") private var selectedTabIndex = 0
    
    var body: some View {
        TabView(selection: $selectedTabIndex) {
            Tab("feed", systemImage: "list.star", value: 0) {
                // content
            }
            
            Tab("settings", systemImage: "gear", value: 2) {
                // content
            }
        }
        .tabViewStyle(.sidebarAdaptable)
        .tabViewBottomAccessory {
            Button("Do Action") {
                
            }
        }
    }
}

```

SwiftUI provides the new_tabViewBottomAccessory_view modifier, allowing us to set up a tab accessory view. Another interesting interaction that Liquid Glass introduced is the tab minimization. So, you can minimize tabs and display only the current one while the user is scrolling the content. We can control the tab minimization behavior by using the new_tabBarMinimizeBehavior_view modifier.

```
struct ContentView: View {
    @SceneStorage("selectedTab") private var selectedTabIndex = 0
    
    var body: some View {
        TabView(selection: $selectedTabIndex) {
            Tab("feed", systemImage: "list.star", value: 0) {
                // content
            }
            
            Tab("settings", systemImage: "gear", value: 2) {
                // content
            }
        }
        .tabViewStyle(.sidebarAdaptable)
        .tabBarMinimizeBehavior(.onScrollDown)
        .tabViewBottomAccessory {
            Button("Do Action") {
                
            }
        }
    }
}

```

Today we learned how to adapt the new Liquid Glass design language to the tab navigation which is the most popular way of structuring mobile apps. I hope you enjoy the post. Feel free to follow me on[Twitter](https://twitter.com/mecid)and ask your questions related to this post. Thanks for reading, and see you next week!