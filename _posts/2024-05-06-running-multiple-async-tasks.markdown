---
layout: post
title: "Swift Concurrency: An Intro to Async/Await"
date: 2024-02-10 09:00:00 +0000
categories: [Swift, Concurrency]
---

# Swift Concurrency: An Intro to Async/Await

Swift Concurrency is more than just a new language feature—it’s part of a broader effort to simplify how we work with asynchronous code. Before `async/await`, many iOS developers used completion handlers, delegation patterns, or even Combine to manage tasks running in the background. While these older techniques still work, they can lead to tangled code if you’re juggling multiple tasks simultaneously.

In this post, we’ll look at why Swift Concurrency matters, how `async/await` compares to older approaches, and a few considerations to keep in mind as you incorporate this feature into your projects.

## A Brief History

Concurrency has always been a challenge in software development. Apple provided Grand Central Dispatch (GCD) for multithreaded tasks, which helped, but it still required explicit queue management and extensive callback code. Over time, Swift evolved to include Combine, which introduced a declarative pattern for handling data streams and asynchronous sequences. But even Combine has a learning curve, especially when it comes to chaining multiple publishers.

Swift Concurrency, introduced around iOS 15 and Swift 5.5, builds on top of GCD, but it’s designed to feel more natural for day-to-day coding. By writing `async` functions and calling them with `await`, you describe concurrency in a way that looks synchronous in source code, even though your app is still doing things in parallel behind the scenes.

## How `async/await` Works

### The `async` Keyword

Marking a function with `async` indicates it may pause its execution to wait for a result. For instance, any network call or heavy I/O task could be marked `async` because it might need time to complete.

```swift
func performDataFetch() async -> Data {
    // Implementation goes here
}
```

### The `await` Keyword

When you call an `async` function, you use `await` to suspend the current function until the result is ready. The suspension is transparent in your code—you don’t see explicit callbacks—but the compiler ensures the function can safely pause and resume.

```swift
let data = await performDataFetch()
```

This single line communicates that your code won’t move forward until the data is fetched, simplifying the flow while still allowing concurrency under the hood.

## Why Drop Completion Handlers?

1. **Ease of Reading**: With `async/await`, your code follows a straightforward top-to-bottom flow. Nested closures and callback pyramids (commonly known as “callback hell”) become a thing of the past.
2. **Error Handling**: Traditional completion handlers often require you to pass around `Result` types or optional error parameters. `async/await` integrates seamlessly with `throws`, so you can use standard `do/try/catch` blocks.
3. **Better Organization**: Instead of sprinkling completion blocks throughout your codebase, you can centralize asynchronous logic into smaller, clearly defined `async` functions.

## Quick Example

Imagine a function that loads a user profile from a remote API:

```swift
func fetchUserProfile() async throws -> UserProfile {
    try await Task.sleep(nanoseconds: 1_000_000_000)  // Simulated delay
    // Fetch data from the network and decode
    return UserProfile(id: 123, name: "Alice")
}
```

And how you might call it:

```swift
func handleUserProfile() async {
    do {
        let profile = try await fetchUserProfile()
        print("Fetched Profile:", profile.name)
    } catch {
        print("Failed to fetch user profile:", error)
    }
}
```

Everything reads in a single, linear flow, even though under the hood the code is still asynchronous.

## Potential Pitfalls

- **Race Conditions**: `async/await` doesn’t automatically prevent shared-data issues. If multiple tasks access the same data, you still need to ensure thread safety (e.g., by using actors or other synchronization methods).
- **Task Overload**: It’s easy to launch too many tasks without realizing it. Swift Concurrency uses a cooperative thread pool, but if you create tasks carelessly, you might still hit performance bottlenecks.
- **Backwards Compatibility**: If your project must support older iOS versions, you’ll need a deployment strategy. Swift does offer back-deployment for concurrency features, but confirm it aligns with your version requirements.

## Structured Concurrency and Actors

Swift Concurrency also introduces structured concurrency and the actor model, which complement `async/await`:

- **Structured Concurrency**: Encourages you to create tasks in a structured way (like child tasks) so you can manage their lifecycles. 
- **Actors**: Protect mutable state by isolating it within a single concurrent context, reducing the chance of data races.

While not mandatory, these features can help keep your codebase organized and prevent common mistakes.

## When To Use It

Swift Concurrency is well-suited for:

- Network requests and other I/O operations.  
- Parallelizing large computations that can run independently.  
- Background tasks that update the UI once finished.  

For purely synchronous logic—or very simple one-off tasks—`async/await` might be overkill. That said, as apps grow, concurrency becomes unavoidable, so learning to integrate `async/await` effectively can save headaches later.

## Final Thoughts

Swift Concurrency holds a lot of promise: it declutters your code, helps avoid the pitfalls of deeply nested callbacks, and integrates naturally with Swift’s error-handling system. That said, it’s important to remain vigilant about potential performance pitfalls and the complexities of multi-threaded environments. Even with `async/await`, concurrency is rarely a fire-and-forget matter—careful design and testing remain vital.

If you’ve been dealing with complex callback hierarchies or labyrinthine Combine pipelines, give Swift Concurrency a try. A little investment in learning this new model often pays off in more maintainable and straightforward code.

