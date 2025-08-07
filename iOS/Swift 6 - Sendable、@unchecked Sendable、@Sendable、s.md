---
title: "Swift 6 - Sendable、@unchecked Sendable、@Sendable、sending and nonsending"
url: "https://fatbobman.com/zh/posts/sendable-sending-nonsending/"
description: "详解 Swift 6 中 Sendable、@Sendable、sending、nonsending 等关键字的作用与区别，助你编写更安全的并发代码，避免数据竞争与线程问题。"
author: "@fatbobman"
site_name: "fatbobman.com"
language: "zh"
published_time: "2025-08-06T14:00:00.000Z"
image: "https://cdn.fatbobman.com/card/sendable-sending-nonsending.webp"
generated_at: "2025-08-07T07:13:16.770Z"
---

[#Swift](/zh/tags/Swift/) [#English](/en/posts/sendable-sending-nonsending/)

Swift 6: Sendable、@unchecked Sendable、@Sendable、sending and nonsending
======================================================================

[东坡肘子](https://x.com/intent/follow?screen_name=fatbobman)

发表于 2025 年 8 月 6 日

Swift 的并发模型引入了众多关键字，其中一些在命名和用途上颇为相似，容易让开发者感到困惑。本文将对 Swift 并发中与跨隔离域传递相关的几个关键字：`Sendable`、`@unchecked Sendable`、`@Sendable`、`sending` 和 `nonsending` 进行梳理，帮助大家理解它们各自的作用和使用场景。

[

马上加入

![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)

代码相连，创意无限 - 加入我们的开发者社区!

加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长!





](https://t.ly/gzxeh)

🚀 马上加入，享受交流的快乐 →

隔离域（Isolation Domain）
---------------------

在探讨这些关键字之前，我们需要先了解 Swift 并发模型中的隔离域概念。

数据竞争一直是并发编程中的棘手问题，稍有不慎就会导致数据错误甚至应用崩溃。为了从根本上解决这个问题，Swift 在构建新的并发模型时引入了隔离域的概念。开发者可以通过 `actor`、`@MainActor` 或其他自定义的 Global Actor 来声明一个隔离域。每个隔离域实例都会通过其 Executor 来维护和管理一个串行队列，保证同一时刻只有一个任务在访问受保护的状态，从而将数据竞争从运行时错误转变为编译时错误。

正因为有了明确的隔离域边界，开发者就需要为数据类型或接收端添加特定的标注，让编译器能够检查数据是否可以在不同的隔离域之间安全传递。因此，一系列包含 “Send” 语义的关键字和协议应运而生。

Sendable
--------

值类型在 Swift 中被广泛使用，其复制语义天然适合在不同的隔离域之间传递。但由于自定义类型中可能包含不能安全传递的成员，因此我们可以通过添加 `Sendable` 标注，让编译器帮助验证其安全性。虽然 `Sendable` 以协议形式出现，但本质上它是一个编译时契约，声明某个类型可以安全地跨隔离域传递，编译器会验证这个声明的真实性。

Swift

Copied!

    // Sendable 是一个标记协议（marker protocol）
    protocol Sendable { }
    
    // 它告诉编译器："这个类型跨隔离域传递是安全的"

对于明显符合 `Sendable` 特性的类型，编译器会自动推断，即使不显式添加 `Sendable` 标注，编译器仍会确保其满足跨隔离域安全传递的要求：

Swift

Copied!

    func getSendable<T:Sendable>(value:T) {}
    
    struct People { // 值类型
        var name:String // 属性都是 Sendable
        var age:Int
    }
    
    let people = People(name: "fat", age: 10)
    getSendable(value: people) // ✅ 自动推断为 Sendable

需要注意的是，在 Swift 6 中，某些之前可以自动推断的场景不再被支持：

Swift

Copied!

    final class NoNeedMarkSendable {
        let name = "fat"
    }
    
    let a = NoNeedMarkSendable()
    getSendable(value: a) // Swift 6 之前或没有开启严格并发检查 ✅
    getSendable(value: a) // Swift 6 之后 ❌： Type 'NoNeedMarkSendable' does not conform to the 'Sendable' protocol

因此，为了确保代码的健壮性，建议开发者显式添加 `Sendable` 标注。

另外，如果一个类型具有明确的隔离域（如 actor 类型或标记为 @MainActor 的类），它本身就被视为 Sendable，因为其内部状态已由 Swift 并发模型提供隔离保障：

Swift

Copied!

    @MainActor
    final class NoNeedMarkSendable {
        let name = "fat"
    }
    
    Task { @MainActor in
      let a = NoNeedMarkSendable()
      getSendable(value: a) // ✅ @MainActor 类型自动 Sendable
    }

`Sendable` 通常作为泛型约束使用，在编译时确保只接受可安全传递的类型。

@unchecked Sendable
-------------------

在某些场景中，特别是处理遗留代码时，我们可能已经通过其他机制（如锁、队列等）确保了线程安全，但编译器无法理解这些实现。此时，开发者可以通过 `@unchecked Sendable` 向编译器保证类型的安全性，从而绕过编译器的自动检查：

Swift

Copied!

    final class ThreadSafeCache: @unchecked Sendable {
        private var cache: [String: Sendable] = [:]
        // 虽然有可变状态，但通过队列保证了线程安全
        private let queue = DispatchQueue(label: "cache", attributes: .concurrent)
        
        func get(_ key: String) -> Sendable? {
            queue.sync {
                cache[key]
            }
        }
        
        func set(_ key: String, value: Sendable) {
            queue.async(flags: .barrier) {
                self.cache[key] = value
            }
        }
    }

需要警惕的是，一些开发者为了让代码通过编译会滥用 `@unchecked Sendable`，这会使严格并发检查失去意义。

从 Swift 6 开始，Synchronization 框架提供的 `Mutex` 类型可以实现真正的 `Sendable`，无需再依赖 `@unchecked`：

Swift

Copied!

    import Synchronization
    
    final class ThreadSafeCache: Sendable {
        private let cache = Mutex<[String: Sendable]>([:])
    
        func get(_ key: String) -> Sendable? {
            cache.withLock {
                $0[key]
            }
        }
    
        func set(_ key: String, value: Sendable) {
            cache.withLock {
                $0[key] = value
            }
        }
    }

@Sendable
---------

由于 `Sendable` 协议只能用于类型声明，Swift 为闭包提供了专门的属性 `@Sendable`。它表示闭包可以安全地跨隔离域传递：

Swift

Copied!

    func getSendableClosure(perform: @escaping @Sendable () -> Void ) {}

编译器会在编译阶段检查闭包的安全性：

Swift

Copied!

    final class NoNeedMarkSendable {
        let name = "fat"
    }
    
    let a = NoNeedMarkSendable()
    getSendableClosure(perform: { a }) // ❌ Capture of 'a' with non-Sendable type 'NoNeedMarkSendable' in a '@Sendable' closure
    
    getSendableClosure(perform: {
       let a = NoNeedMarkSendable() // ✅ 在闭包内部创建，不存在跨隔离域传递
    })

sending
-------

`@Sendable` 闭包的限制有时过于严格。考虑以下场景：

Swift

Copied!

    Task {
        let nonSendable = NonSendableClass()
        getSendableClosure {
            nonSendable.value += 1 // Capture of 'nonSendable' with non-Sendable type 'NonSendableClass' in a '@Sendable' closure
        }
        // 实际上这里是安全的，因为 nonSendable 不会在其他地方被修改
    }

Swift 6 引入了 `sending` 参数修饰符来解决这个问题：

Swift

Copied!

    // 原来的
    func getSendableClosure(perform: @escaping @Sendable  () -> Void) {}
    
    // 新的
    func getSendableClosure(sending perform: @escaping  () -> Void) {}
    
    Task {
        let nonSendable = NonSendableClass()
        getSendableClosure {
            nonSendable.value += 1 // ✅ 换成 `sending` 后
        }
    }

但 `sending` 并非像 `@unchecked Sendable` 一样，会绕过 Swift 编译器的检查。如果出现了不安全的使用场景，编译器仍然会提醒我们：

Swift

Copied!

    actor MyActor {
        var storage: NonSendableClass?
        
        // 使用 sending 允许接收非 Sendable 参数
        func store(_ object: sending NonSendableClass) {
            storage = object 
        }
    }
    
    Task {
        let obj = NonSendableClass()
        let myActor = MyActor()
        await myActor.store(obj) // ✅ 所有权转移给 actor
        // obj.value += 1 // ❌ 'obj' used after being passed as a 'sending' parameter
    }

这是因为，`sending` 的核心概念是“**转移所有权**”，尽管编译器不再对 `Sendable` 进行检查，但是会对所有权进行检查。在转移后，我们不能在其他地方再使用已转移的值！

因此，`sending` 更适合的场景是：

*   迁移非 Sendable 的遗留代码
*   实现构建者模式
*   任务结果传递（Swift 6 中，Task 的实现已从 `@Sendable` 改为 `sending`）

> `@Sendable` 是类型安全地“声明线程安全”，而 `sending` 是所有权语义上的“保证单一使用权”，二者并非互斥，而是适用于不同上下文。

nonsending
----------

`nonsending` 与前面的关键字在语义上有本质区别。它与 `nonisolated` 配合使用：`nonisolated(nonsending)`，表示异步方法应该继承调用者的隔离域，而不是脱离当前隔离域运行。

这是 Swift 6.2 对 `nonisolated` 行为的重要调整。在 Swift 6.2 之前：

Swift

Copied!

    @MainActor
    class RunInMainActor {    
        var name: String = "example"
        
        nonisolated func processData() async {
            // Swift 6.2 之前：总是脱离 MainActor 运行（非主线程）
        }
    }

使用 `nonisolated(nonsending)` 后：

Swift

Copied!

    @MainActor
    class RunInMainActor {    
        var name: String = "example"
        
        nonisolated(nonsending) func processData() async {
            // Swift 6.2：继承调用者的隔离域
            // 从 MainActor 调用时运行在主线程
            // 从其他 actor 调用时运行在对应的隔离域
        }
    }

开发者还可以通过启用 `NonisolatedNonsendingByDefault` 标志使该行为成为默认设置。

关于 `nonisolated(nonsending)` 和 Swift 6.2 的更多变化，请参阅 [Default Actor Isolation：好初衷带来的新问题](/zh/posts/default-actor-isolation/)

[

马上加入

![Join Discord Logo](https://cdn.fatbobman.com/ads/apple-100.svg)

代码相连，创意无限 - 加入我们的开发者社区!

加入我们活跃的 Discord 苹果开发者社区! 与 2000 多名中文开发者一起深入探讨 Swift、SwiftUI 和 SwiftData 的最新进展。分享经验，解决难题，共同成长!





](https://t.ly/gzxeh)

🚀 马上加入，享受交流的快乐 →

总结
--

**🔑** **关键字**

**🧭** **用途**

**🎯** **典型使用场景**

**🛡️** **编译器行为**

**✅** **安全保证方式**

Sendable

类型标记

跨隔离域传递类型

✅ 编译器验证类型是否线程安全

✅ 自动或显式声明

@unchecked Sendable

跳过类型验证

遗留代码、手动保障线程安全

⚠️ 绕过检查，完全依赖开发者判断

⚠️ 依赖开发者保证

@Sendable

闭包属性标记

跨隔离域传递闭包

✅ 编译器检查捕获值是否 Sendable

✅ 自动检查

sending

参数修饰

所有权转移、构建器模式

🚫 不检查 Sendable，但禁止再次使用

🚫 所有权语义（防止误用）

nonsending

隔离继承修饰符

保持调用者隔离域，适配异步调用

✅ 控制运行时隔离上下文

✅ 行为由调用者上下文决定

诚然，Swift 并发模型中众多的关键字增加了学习曲线的陡峭度。但理解这些概念对于编写安全的并发代码至关重要。即使某些关键字在日常开发中使用频率不高，掌握它们的含义也能帮助我们在遇到问题时快速定位和解决。

> **公告：** 又到了每年休暑假 ⛱️ 的时候，接下来博客文章将会停更 4-5 周，期间《肘子的 Swift 周报》仍正常更新。

> "加入我们的 [Discord 社区](https://discord.com/invite/7FedN5E2QQ)，与超过 2000 名苹果生态的中文开发者一起交流！"

[Swifter and Swifty：掌握 Swift Testing 框架](/zh/posts/mastering-the-swift-testing-framework/)[Default Actor Isolation：好初衷带来的新问题](/zh/posts/default-actor-isolation/)[SwiftUI 中的 UserDefaults 与 Observation：如何实现精准响应](/zh/posts/userdefaults-and-observation/)[AttributedString —— 不仅仅让文字更漂亮](/zh/posts/attributedstring/)[从基础到进阶：Swift 中的 KeyPath 完全指南](/zh/posts/comprehensive-guide-to-mastering-keypath-in-swift/)[Swift Predicate: 用法、构成及注意事项](/zh/posts/swift-predicate-usage-composition-and-considerations/)

每周精选 Swift 与 SwiftUI 精华！

链接已复制

.copy-icon { position: absolute; left: -30px; /\* 根据您的布局调整位置 \*/ top: 50%; /\* 将图标定位到标题的中间高度 \*/ transform: translateY(-50%); /\* 垂直居中对齐图标 \*/ cursor: pointer; transition: opacity 0.6s ease; pointer-events: all; /\* 始终响应鼠标事件 \*/ } article h2, article h3, article h4, article h5 { position: relative; } #copy-alert { position: fixed; bottom: -100px; /\* 初始在视窗下方 \*/ right: 20px; transition: bottom 0.5s ease-in-out; } /\* 进入动画 \*/ @keyframes slideIn { from { bottom: -100px; } to { bottom: 0px; } } /\* 消失动画 \*/ @keyframes slideOut { from { bottom: 20px; } to { bottom: -100px; } } var hideTimeout; // 用于保存自动隐藏定时器的引用 document.querySelectorAll('article h2, article h3, article h4, article h5').forEach(function (header) { var copyIcon = document.createElement('span'); copyIcon.innerHTML = '<svg fill="currentColor" aria-hidden="true" stroke="currentColor" class="w-5 h-5 hidden md:block hover:text-secondary" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg"><path d="M861.866667 164.266667c-87.466667-87.466667-230.4-89.6-320-2.133334l-68.266667 68.266667c-12.8 12.8-12.8 32 0 44.8s32 12.8 44.8 0l68.266667-68.266667c64-61.866667 166.4-61.866667 230.4 2.133334 64 64 64 168.533333 2.133333 232.533333l-117.333333 119.466667c-34.133333 34.133333-81.066667 51.2-128 49.066666-46.933333-4.266667-91.733333-27.733333-119.466667-66.133333-10.666667-14.933333-29.866667-17.066667-44.8-6.4-14.933333 10.666667-17.066667 29.866667-6.4 44.8 40.533333 53.333333 100.266667 87.466667 166.4 91.733333h17.066667c59.733333 0 119.466667-23.466667 162.133333-68.266666l117.333333-119.466667c83.2-89.6 83.2-234.666667-4.266666-322.133333z"></path><path d="M505.6 750.933333l-66.133333 68.266667c-64 61.866667-166.4 61.866667-230.4-2.133333-64-64-64-168.533333-2.133334-232.533334l117.333334-119.466666c34.133333-34.133333 81.066667-51.2 128-49.066667 46.933333 4.266667 91.733333 27.733333 119.466666 66.133333 10.666667 14.933333 29.866667 17.066667 44.8 6.4 14.933333-10.666667 17.066667-29.866667 6.4-44.8-40.533333-53.333333-100.266667-87.466667-166.4-91.733333-66.133333-4.266667-130.133333 19.2-177.066666 66.133333l-117.333334 119.466667c-85.333333 89.6-85.333333 234.666667 2.133334 322.133333 44.8 44.8 102.4 66.133333 162.133333 66.133334 57.6 0 115.2-21.333333 160-64l66.133333-68.266667c12.8-12.8 12.8-32 0-44.8-14.933333-10.666667-34.133333-10.666667-46.933333 2.133333z"></path></svg>'; // 使用您选择的图标 copyIcon.className = 'copy-icon'; copyIcon.style.opacity = '0'; // 默认透明 header.insertBefore(copyIcon, header.firstChild); header.addEventListener('mouseover', function () { copyIcon.style.opacity = '1'; }); header.addEventListener('mouseout', function () { copyIcon.style.opacity = '0'; }); copyIcon.addEventListener('click', function () { var url = window.location.href.split('#')\[0\] + '#' + header.id; navigator.clipboard.writeText(url).then(function () { var alertDiv = document.getElementById('copy-alert'); if (alertDiv) { alertDiv.classList.remove('hidden'); // 确保提示框是可见的 alertDiv.style.display = 'block'; // 显示提示框 clearTimeout(hideTimeout); // 清除之前的定时器 alertDiv.style.animation = 'none'; // 先清除之前的动画 setTimeout(function () { alertDiv.style.animation = 'slideIn 0.5s forwards'; // 应用新动画 }, 10); // 设置自动隐藏定时器 hideTimeout = setTimeout(function () { alertDiv.style.animation = 'slideOut 0.5s forwards'; // 应用消失动画 setTimeout(function () { alertDiv.style.display = 'none'; }, 500); }, 3000); } }); }); }); function copyCode(button, event) { event.preventDefault(); const codeBlock = button.parentNode.nextElementSibling.textContent; // 获取代码内容 navigator.clipboard.writeText(codeBlock).then(() => { // 显示 "Copied" button.querySelector('.copy-text').style.display = 'none'; button.querySelector('.copied-text').style.display = 'inline'; // 3秒后恢复 setTimeout(() => { button.querySelector('.copy-text').style.display = 'inline'; button.querySelector('.copied-text').style.display = 'none'; }, 3000); }); } (function(){const url = "https://fatbobman.com/ads/zh"; // 根据中英文对应不同的 adCacheKey const adCacheKey = 'dynamicAdCache' + (window.location.pathname.includes('/zh/') ? 'ZH' : 'EN'); // 尝试从缓存中获取广告 const cachedAd = JSON.parse(localStorage.getItem(adCacheKey)); const now = Date.now(); if (cachedAd && cachedAd.expiresAt > now) { console.log('Using cached ad'); console.log(cachedAd.expiresAt - now); // 如果缓存有效，直接使用缓存的广告 renderAd(cachedAd.content); } else { console.log('Fetching ad from the server'); // 如果缓存无效，从服务器获取广告 fetch(url, { headers: { 'Accept': 'text/html', }, cache: 'default' }) .then(response => { const maxAgeMatch = response.headers.get('Cache-Control')?.match(/max-age=(\\d+)/); const maxAge = maxAgeMatch ? parseInt(maxAgeMatch\[1\], 10) : 0; if (!response.ok || maxAge <= 0) { throw new Error('Invalid response or missing Cache-Control'); } return response.text().then(adContent => { const expiresAt = now + maxAge \* 1000; // 计算失效时间 // 缓存广告内容 localStorage.setItem(adCacheKey, JSON.stringify({ content: adContent, expiresAt })); // 渲染广告 renderAd(adContent); }); }) .catch(error => console.error('Error loading the ad:', error)); } // 渲染广告函数 function renderAd(html) { const adSlots = document.querySelectorAll('.dynamic-ad-container'); adSlots.forEach(slot => { slot.innerHTML = html; }); } })(); var c=document.querySelectorAll("article");c.forEach(function(r){var a=r.querySelectorAll("a");a.forEach(function(e){var t=e.getAttribute("href");t&&t.startsWith("http")&&!t.includes("fatbobman.com")&&!t.includes("fatbobman.github.com")&&!t.includes("localhost")&&e.setAttribute("target","\_blank")})});

document.addEventListener('DOMContentLoaded', async function () { const container = document.getElementById('custom-substack-embed'); let anchor = ''; if (container) { anchor = container.getAttribute('data-anchor') || ''; } // 提前检查是否需要加载评论 try { const response = await fetch('https://fatbobman.com/status/comment'); const data = await response.text(); console.log(data, "comment"); if (data !== '1') { console.log('评论功能未启用，跳过监控和加载操作。'); return; // 提前返回，跳过后续逻辑 } } catch (error) { console.error('检查评论状态时出错：', error); return; // 出现错误时也跳过后续逻辑 } // 如果返回值为 "1"，则启动观察器 const observerTarget = document.getElementById(anchor); if (observerTarget) { const observer = new IntersectionObserver(async (entries, observer) => { const entry = entries\[0\]; if (entry.isIntersecting) { console.log("load comment") // 停止观察，避免重复触发 observer.unobserve(observerTarget); // 动态加载评论区脚本 await loadGitalk(); } }, { root: null, // 相对于视口 rootMargin: '20%', // 提前 20% 的位置触发 threshold: 0.1, // 元素显示 10% 即触发 }); observer.observe(observerTarget); } // 动态加载 Gitalk 脚本并初始化评论区 async function loadGitalk() { const container = document.getElementById('gitalk-container'); if (container) { const commentID = container.getAttribute('data-comment-id'); const lang = container.getAttribute('data-lang'); const repo = container.getAttribute('data-repo') const createIssueManually = container.getAttribute('data-createIssueManually') console.log(repo); console.log(createIssueManually) // 动态加载 Gitalk 脚本 const script = document.createElement('script'); script.src = 'https://cdn.jsdelivr.net/npm/gitalk@1/dist/gitalk.min.js'; script.defer = true; script.onload = () => { const gitalk = new Gitalk({ clientID: 'fcf61195c7f73253dc8b', clientSecret: '0ac2907be08248a1fcb5312e27480ad535c682e5', repo: repo, owner: 'fatbobman', admin: \['fatbobman'\], id: commentID.split('/').pop().substring(0, 49), distractionFreeMode: true, createIssueManually: createIssueManually, language: lang, }); gitalk.render('gitalk-container'); }; document.body.appendChild(script); } } });

*   [隔离域（Isolation Domain）](#隔离域isolation-domain)
    
*   [Sendable](#sendable)
    
*   [@unchecked Sendable](#unchecked-sendable)
    
*   [@Sendable](#sendable)
    
*   [sending](#sending)
    
*   [nonsending](#nonsending)
    
*   [总结](#总结)
    

Default Actor Isolation：好初衷带来的新问题

⌥ + ←

(function(){const url = "/zh/posts/default-actor-isolation/"; const isHidden = false; document.addEventListener('keydown', (e) => { // Check if Alt+RightArrow or just RightArrow was pressed if (e.altKey && e.key === 'ArrowLeft') { // Only navigate if there's a URL and the button isn't hidden if (url && !isHidden) { e.preventDefault(); // Prevent default browser behavior window.location.href = url; } } }); })();

(function(){const url = "#"; const isHidden = true; document.addEventListener('keydown', (e) => { // Check if Alt+RightArrow or just RightArrow was pressed if (e.altKey && e.key === 'ArrowRight') { // Only navigate if there's a URL and the button isn't hidden if (url && !isHidden) { e.preventDefault(); // Prevent default browser behavior window.location.href = url; } } }); })();

[](https://x.com/intent/tweet?text=Swift%206%3A%20Sendable%E3%80%81unchecked%20Sendable%E3%80%81Sendable%E3%80%81sending%20and%20nonsending&url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fsendable-sending-nonsending%2F&hashtags=ios%2Cswiftlang&via=fatbobman)

Share on X

Share on BlueSky

[](https://www.facebook.com/sharer/sharer.php?u=https%253A%252F%252Ffatbobman.com%252Fzh%252Fposts%252Fsendable-sending-nonsending%252F)

Share on Facebook

[](https://service.weibo.com/share/share.php?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fsendable-sending-nonsending%2F&title=Swift%206%3A%20Sendable%E3%80%81unchecked%20Sendable%E3%80%81Sendable%E3%80%81sending%20and%20nonsending%0A%20%23ios%20%23swift%0A%20by%20%40fatbobman%0A)

Share on Weibo

[](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fsendable-sending-nonsending%2F)

Share on Linkedin

[](mailto:?subject=%E6%96%87%E7%AB%A0%3A%E3%80%90Swift%206%3A%20Sendable%E3%80%81unchecked%20Sendable%E3%80%81Sendable%E3%80%81sending%20and%20nonsending%E3%80%91&body=%E5%88%86%E4%BA%AB%E4%B8%80%E7%AF%87%E6%96%87%E7%AB%A0%E7%BB%99%E4%BD%A0%EF%BC%9A%0A%0ASwift%206%3A%20Sendable%E3%80%81unchecked%20Sendable%E3%80%81Sendable%E3%80%81sending%20and%20nonsending%0A%0A%E9%93%BE%E6%8E%A5%3A%20https%3A%2F%2Ffatbobman.com%2Fzh%2Fposts%2Fsendable-sending-nonsending%2F)

Share via Email

Share Your Thoughts

const SCROLL\_OFFSET = 80; document.addEventListener('DOMContentLoaded', function() { const commentButton = document.getElementById('comment-button'); if (commentButton) { commentButton.addEventListener('click', function() { const gitalkContainer = document.getElementById('gitalk-container'); if (gitalkContainer) { const topPosition = gitalkContainer.getBoundingClientRect().top + window.scrollY - SCROLL\_OFFSET; window.scrollTo({ top: topPosition, behavior: 'smooth' }); } }); } });

Back to Top