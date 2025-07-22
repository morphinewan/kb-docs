[ ![Give your simulator superpowers](https://www.avanderlee.com/wp-content/themes/casper/img/rocketsim_icon.png)veloper Tool as recommended by Apple ](https://www.rocketsim.app?utm%5Fsource=swiftlee&utm%5Fmedium=web&utm%5Fcampaign=rocketsim-sticky) 

[SwiftLee](https://www.avanderlee.com/) \> [Concurrency](https://www.avanderlee.com/category/concurrency/) \> Modern Swift Lock: Mutex & the Synchronization Framework

[ ](https://twitter.com/intent/tweet?via=twannl&hashtags=swiftlee,ios,swiftlang&url=https%3A%2F%2Fwww.avanderlee.com%2Fconcurrency%2Fmodern-swift-lock-mutex-the-synchronization-framework%2F&text=Modern+Swift+Lock%3A+Mutex+%26+the+Synchronization+Framework "Share on Twitter") [ ](https://www.linkedin.com/feed/?shareActive=true&text=Modern+Swift+Lock%3A+Mutex+%26+the+Synchronization+Framework+by+%40Antoine+van+der+Lee%3A%0A%0Ahttps%3A%2F%2Fwww.avanderlee.com%2Fconcurrency%2Fmodern-swift-lock-mutex-the-synchronization-framework%2F "Share on LinkedIn") 

In this article

[ Swift Concurrency Course](https://www.swiftconcurrencycourse.com?utm%5Fsource=swiftlee&utm%5Fmedium=blog&utm%5Fcampaign=article-toc "Learn more about Swift Concurrency using this in-depth course.") 

[Concurrency](https://www.avanderlee.com/category/concurrency/ "View all posts in Concurrency") [Jul 15, 2025Jul 14, 2025](https://www.avanderlee.com/concurrency/modern-swift-lock-mutex-the-synchronization-framework/) •  4 min read 

# [Modern Swift Lock: Mutex & the Synchronization Framework](https://www.avanderlee.com/concurrency/modern-swift-lock-mutex-the-synchronization-framework/)

Swift offers several solutions to lock access to mutable content and prevent so-called [data races](https://www.avanderlee.com/swift/race-condition-vs-data-race/). Locks like NSLock, DispatchSemaphore, or a serial DispatchQueue are a popular choice for many. Some articles compare their performance and tell you which one works best, but I’d like to introduce you to a modern Swift lock variant introduced via [SE-433 Synchronous Mutual Exclusion Lock](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0433-mutex.md).

This article won’t tell you which lock performs best, and I also won’t compare them to this new variant. Each lock can have a different performance profile and features to offer. In this article, we’re going to look at a standardized version of a so-called Mutex lock.

[Transform Your Career with the iOS Lead Essentials — Limited Offer](https://iosacademy.essentialdeveloper.com/p/ios-architect-crash-course-slb07b/?utm%5Fsource=swiftlee&utm%5Fmedium=advertisement&utm%5Fcampaign=essentialdeveloper%5Fvariant%5Fa)Unlock over 40 hours of expert training, mentorship, and community support to secure your place among the best devs. [**Click now**](https://iosacademy.essentialdeveloper.com/p/ios-architect-crash-course-slb07b/?utm%5Fsource=swiftlee&utm%5Fmedium=advertisement&utm%5Fcampaign=essentialdeveloper%5Fvariant%5Fa) for early access to this limited offer and a FREE crash course!

## [ ](https://www.avanderlee.com/concurrency/modern-swift-lock-mutex-the-synchronization-framework/?utm_source=substack&utm_medium=email/#what-is-a-swift-lock)What is a Swift Lock?

A Swift Lock, or locks in general, allow you to ensure only one thread or task can access a piece of data at a time. It ensures thread-safety access and prevents exceptions caused by data being accessed simultaneously. The latter is called a data race, and it’s a common cause of crashes in applications. You can read more about detecting data races in your code by reading [Thread Sanitizer explained: Data Races in Swift](https://www.avanderlee.com/swift/thread-sanitizer-data-races/).

### [ ](https://www.avanderlee.com/concurrency/modern-swift-lock-mutex-the-synchronization-framework/?utm_source=substack&utm_medium=email/#what-is-the-difference-between-a-mutex-and-a-lock)What is the difference between a Mutex and a Lock?

All mutexes are locks, but not all locks are mutexes. Mutex is a shorthand for mutual exclusion, and it’s a specific type of lock that strictly enforces mutual exclusion, meaning only one thread can own it at a time. This ownership means it can only be unlocked by the same thread or task that locked it.

This strict ownership model makes mutexes reliable and helps prevent specific bugs related to incorrect unlocking. On the other hand, the term “lock” is broader and can refer to different synchronization tools, like reentrant locks, reader-writer locks, or unfair locks. While both ensure exclusive access to shared resources, mutexes focus on simplicity and strict ownership, whereas locks offer more flexibility and can be tailored to different concurrency scenarios.

The Essential Swift Concurrency Course for a Seamless Swift 6 Migration.

Learn all about Swift Concurrency in my flagship course offering 58+ lessons, videos, code examples, and an official certificate of completion.   
  
Updated for Swift 6.2 and now available with a **limited-time** launch offer.

[Go to the course](https://www.swiftconcurrencycourse.com?utm%5Fsource=swiftlee&utm%5Fmedium=article&utm%5Fcampaign=inline-banner) 

## [ ](https://www.avanderlee.com/concurrency/modern-swift-lock-mutex-the-synchronization-framework/?utm_source=substack&utm_medium=email/#using-swifts-mutex-lock-from-the-synchronization-framework)Using Swift’s Mutex lock from the Synchronization framework

Now that we know the difference between a lock and a mutex, it’s time to dive into Apple’s Synchronization framework. This framework was announced during WWDC 24 and is available from iOS 18 and macOS 15\. This is important for apps that still have to support older OS versions.

In this example, we’re going to use a Mutex Swift lock to protect a counter. This is a classic example to demonstrate the concept of a Mutex:

```
final class Counter {
    
    /// Use the Mutex to protect the count value.
    private let count = Mutex<Int>(0)
    
    /// Provide a public accessor to read the current count value.
    var currentCount: Int {
        count.withLock { currentCount in
            return currentCount
        }
    }
    
    func increment() {
        count.withLock { currentCount in
            currentCount += 1
        }
    }
    
    func decrement() {
        count.withLock { currentCount in
            currentCount -= 1
        }
    }
}
```

As you can see, we’ve used the Mutex to protect the stored count value. Doing so offers a thread-safe way to keep track of a count. We’ve created public accessors to communicate with the count and the Mutex enforces us to make use of the `withLock` method.

This method provides an `inout` access to the count property. This basically means that we can directly interact with the mutable value of `count`. Therefore, we can update the count using:

```
currentCount += 1
```

The `withLock` method forwards any returned values. This is why we can return the current count by simply returning the closure parameter:

```
var currentCount: Int {
    count.withLock { currentCount in
        return currentCount
    }
}
```

### [ ](https://www.avanderlee.com/concurrency/modern-swift-lock-mutex-the-synchronization-framework/?utm_source=substack&utm_medium=email/#throwing-errors-from-within-a-mutex)Throwing errors from within a Mutex

It’s also possible to throw an error from within the `withLock` closure. For example, we could decide to throw a `reachedZero` error when someone tries to decrement below zero:

```
func decrement() throws {
    try count.withLock { currentCount in
        guard currentCount > 0 else {
            throw Error.reachedZero
        }
        currentCount -= 1
    }
}
```

## [ ](https://www.avanderlee.com/concurrency/modern-swift-lock-mutex-the-synchronization-framework/?utm_source=substack&utm_medium=email/#a-lock-that-works-great-with-swift-concurrency)A lock that works great with Swift Concurrency

The Mutex is a Swift lock that works great with Swift Concurrency. It’s unconditionally `Sendable`, which means that it provides thread-safe (Sendable) access to any non-Sendable value.

Recently, I was working with `NSBezierPath,` which is a mutable non-Sendable type. I used this path to store touches for Simulator recordings in [RocketSim](https://www.rocketsim.app). I was able to work safely with this instance by wrapping it inside a Mutex:

```
final class TouchesCapturer: Sendable {
    let path = Mutex<NSBezierPath>(NSBezierPath())
    
    func storeTouch(_ point: NSPoint) {
        path.withLock { path in
            path.move(to: point)
        }
    }
}
```

This made my `TouchesCapturer` conform to `Sendable` and demonstrates how you can write a solution to work with non-Sendable types.

We can demonstrate this same concept by looking at a search history example in which we store search queries inside a mutable array:

![](https://www.avanderlee.com/wp-content/smush-webp/2025/07/swift_lock_mutex-1024x201.png.webp) 

Using a Swift lock like a mutex can help you create sendable access to data.

## [ ](https://www.avanderlee.com/concurrency/modern-swift-lock-mutex-the-synchronization-framework/?utm_source=substack&utm_medium=email/#shouldnt-i-use-an-actor-instead-of-locks-in-swift-concurrency)Shouldn’t I use an actor instead of locks in Swift Concurrency?

I bet many of you are wondering why Apple introduced this Mutex while it’s also working on modern concurrency APIs. Shouldn’t you use an actor in these cases?

Actors are indeed a fantastic tool for protecting mutable state in many scenarios, but they aren’t always the right fit. There are situations where you need synchronous, immediate access to data without introducing the async keyword or suspension points. Sometimes, your code needs to interact with APIs or legacy code that doesn’t support Swift Concurrency at all, making actors impractical or impossible to adopt.

Moreover, actors come with certain design trade-offs — they isolate state and enforce exclusive access through asynchronous messages, which is great for safety but can introduce additional overhead and complexity when low-level, fine-grained locking is required. In these cases, a Mutex offers a lightweight and familiar synchronization primitive that can be used without changing your code to be asynchronous or refactoring everything around await.

Ultimately, it’s not about choosing one over the other universally — it’s about picking the right tool for the job. Actors shine when you can adopt an async model and benefit from clear logical isolation, whereas mutexes fill the gaps where synchronous, immediate access, and minimal disruption are necessary.

[Transform Your Career with the iOS Lead Essentials — Limited Offer](https://iosacademy.essentialdeveloper.com/p/ios-architect-crash-course-slb07b/?utm%5Fsource=swiftlee&utm%5Fmedium=advertisement&utm%5Fcampaign=essentialdeveloper%5Fvariant%5Fa)Unlock over 40 hours of expert training, mentorship, and community support to secure your place among the best devs. [**Click now**](https://iosacademy.essentialdeveloper.com/p/ios-architect-crash-course-slb07b/?utm%5Fsource=swiftlee&utm%5Fmedium=advertisement&utm%5Fcampaign=essentialdeveloper%5Fvariant%5Fa) for early access to this limited offer and a FREE crash course!

### [ ](https://www.avanderlee.com/concurrency/modern-swift-lock-mutex-the-synchronization-framework/?utm_source=substack&utm_medium=email/#conclusion)**Conclusion**

The Synchronization framework introduces a Mutex which is a modern Swift lock to create mutually exclusive access to data. It works great with Swift Concurrency and provides a solution to non-Sendable types without introducing the overhead of an actor.

If you want to learn more about Swift Concurrency, I invite you to follow my dedicated course at [www.swiftconcurrencycourse.com](http://www.swiftconcurrencycourse.com). It dives deeper into topics like locks, actors, and `Sendable`.

Thanks!

![](https://www.avanderlee.com/wp-content/uploads/2024/12/Avatar_2022_masked.webp) 

Written by

#### [Antoine van der Lee](https://www.avanderlee.com "Visit Antoine van der Lee’s website")

iOS Developer since 2010, former Staff iOS Engineer at [WeTransfer](https://www.wetransfer.com) and currently full-time Indie Developer & Founder at SwiftLee. Writing a new blog post every week related to Swift, iOS and Xcode. Regular speaker and workshop host.

[Book a 1:1 with me](https://intro.co/antoinevanderlee?utm%5Fsource=swiftlee&utm%5Fmedium=blog&utm%5Fcampaign=article-author-button) [Follow](https://twitter.com/intent/user?original%5Freferer=https%3A%2F%2Fwww.avanderlee.com&region=count%5Flink&screen%5Fname=twannl&tw%5Fp=followbutton) 

Are you ready to

#### Turn your side projects into independence?

 Learn my proven steps to transform your passion into profit.

[Going Indie Course](https://www.going-indie.com?utm%5Fsource=swiftlee&utm%5Fmedium=article%5Ffooter&utm%5Fcampaign=author%5Fsection) [Going Indie Podcast](https://podcast.going-indie.com?utm%5Fsource=swiftlee&utm%5Fmedium=article%5Ffooter&utm%5Fcampaign=author%5Fsection) 

### Other also read

### [MainActor usage in Swift explained to dispatch to the main thread](https://www.avanderlee.com/swift/mainactor-dispatch-main-thread/)

 MainActor is a new attribute introduced in Swift 5.5 as a global actor providing an executor that performs its tasks ...  
[Read More](https://www.avanderlee.com/swift/mainactor-dispatch-main-thread/)

Nov 11, 2024 [Concurrency](https://www.avanderlee.com/category/concurrency/ "View all posts in Concurrency") [Swift](https://www.avanderlee.com/category/swift/ "View all posts in Swift") 

### [Swift Concurrency & Swift 6 Course (Launch offer)](https://www.avanderlee.com/swift/swift-concurrency-course-tutorial-book/)

 A Swift Concurrency Course that helps you learn all the fundamentals of Swift Concurrency and migrate your projects smoothly to ...  
[Read More](https://www.avanderlee.com/swift/swift-concurrency-course-tutorial-book/)

Jul 08, 2025 [Concurrency](https://www.avanderlee.com/category/concurrency/ "View all posts in Concurrency") [Swift](https://www.avanderlee.com/category/swift/ "View all posts in Swift") 

### [Swift Tutorials: Learn Swift with Easy-to-Follow Code Examples](https://www.avanderlee.com/swift/swift-tutorials-learn-swift-code-examples/)

 Swift tutorials help you to get started building apps for Apple’s platforms. The best way to learn Swift is by ...  
[Read More](https://www.avanderlee.com/swift/swift-tutorials-learn-swift-code-examples/)

Dec 23, 2024 [Swift](https://www.avanderlee.com/category/swift/ "View all posts in Swift") 

[Subscribe!](https://www.avanderlee.com/feed/) 

### Categories

* [Swift](https://www.avanderlee.com/category/swift)
* [SwiftUI](https://www.avanderlee.com/category/swiftui)
* [Swift Testing](https://www.avanderlee.com/category/swift-testing)
* [Concurrency](https://www.avanderlee.com/category/concurrency)
* [Xcode](https://www.avanderlee.com/category/xcode)
* [Debugging](https://www.avanderlee.com/category/debugging)

### Follow

* [Newsletter](https://www.avanderlee.com/swiftlee-weekly-subscribe/)
* [Twitter](https://www.twitter.com/twannl)
* [Bluesky](https://bsky.app/profile/avanderlee.com)
* [LinkedIn](https://nl.linkedin.com/in/ajvanderlee)
* [GitHub](https://www.github.com/avdlee)
* [RocketSim](https://www.twitter.com/rocketsim%5Fapp)
* [RSS](https://www.avanderlee.com/feed)

### Info

* [About](https://www.avanderlee.com/about)
* [Shop](https://shop.spreadshirt.nl/swiftlee/)
* [Sponsor](https://www.avanderlee.com/sponsor)
* [Contact](https://www.avanderlee.com/concurrency/modern-swift-lock-mutex-the-synchronization-framework/?utm_source=substack&utm_medium=email/mailto:contact@avanderlee.com)
* [Book a 1:1 with Antoine](https://intro.co/antoinevanderlee?utm%5Fsource=swiftlee&utm%5Fmedium=blog&utm%5Fcampaign=site-footer)

 Copyright © 2015-2025 [SwiftLee](https://www.avanderlee.com). All Rights Reserved.