---
layout: post
title: "Understanding Swift Concurrency: AsyncQueue, AsyncStream, and AsyncSequence"
date: 2025-02-01
categories: [iOS, Swift]
header_image: /assets/images/swift-concurrency.jpg
---

Concurrency in Swift has come a long way. If you’ve been working in iOS for more than a couple of years, you’ve probably felt the pain of juggling `DispatchQueue`, `OperationQueue`, semaphores, or worse—nested completion handlers tangled into callback hell.

Structured concurrency, introduced in Swift 5.5, changed the game. With `async/await`, `Task`, and `actors`, our code became more readable and maintainable. But there are more advanced tools that deserve attention—particularly when you’re dealing with event streams, background queues, or building asynchronous APIs from scratch.

In this post, I’ll unpack three such tools: `AsyncQueue`, `AsyncStream`, and `AsyncSequence`. These are not just syntactic sugar—they allow us to build clean, robust, and testable async flows in Swift.

---

## 🔁 AsyncQueue: Ordered Execution in an Async World

### What is AsyncQueue?

`AsyncQueue` is a concurrency pattern that ensures async tasks are executed **serially**—in order—even though they’re defined using Swift’s structured concurrency tools.

This pattern is not part of the standard library but can be implemented easily using `Task` and `actor`. Think of it as an `OperationQueue` where each operation is an `async` closure, and operations are executed one after another.

### Why Use AsyncQueue?

Let’s say you need to:

- Process user events in order (e.g., keyboard input, game moves)
- Save user data one chunk at a time
- Sequentially animate UI elements
- Avoid race conditions without locking

A typical GCD serial queue could work, but using `Task` and `actor` makes the solution more idiomatic and testable in modern Swift.

### Implementation

```swift
actor AsyncQueue {
    private var lastTask: Task<Void, Never>?

    func enqueue(_ operation: @escaping () async -> Void) {
        let previousTask = lastTask
        lastTask = Task {
            await previousTask?.value
            await operation()
        }
    }

    func cancelAll() {
        lastTask?.cancel()
        lastTask = nil
    }
}
```

### Example: Saving Files in Order

```swift
let fileQueue = AsyncQueue()

func saveFile(named name: String, data: Data) {
    fileQueue.enqueue {
        let url = FileManager.default.temporaryDirectory.appendingPathComponent(name)
        try? data.write(to: url)
        print("Saved file: \(name)")
    }
}

saveFile(named: "a.txt", data: Data("Hello".utf8))
saveFile(named: "b.txt", data: Data("World".utf8))
```

Each `saveFile` call is async, but they are guaranteed to run in the order they were enqueued.

---

## 🌊 AsyncStream: Observing Asynchronous Events

### What is AsyncStream?

`AsyncStream` is a **bridge** between the old world (delegates, notifications, and callbacks) and the new world of `async/await`. It lets you create an asynchronous sequence of values that can be iterated over using `for await`.

It’s particularly useful when:

- Wrapping callback-based APIs
- Observing system notifications or events
- Handling user input streams
- Consuming WebSocket messages

### Basic Example: Wrapping a Timer

```swift
func makeTimerStream(interval: TimeInterval) -> AsyncStream<Date> {
    AsyncStream { continuation in
        let timer = Timer.scheduledTimer(withTimeInterval: interval, repeats: true) { _ in
            continuation.yield(Date())
        }

        continuation.onTermination = { @Sendable _ in
            timer.invalidate()
        }
    }
}
```

### Using the Stream

```swift
for await tick in makeTimerStream(interval: 1.0) {
    print("Tick: \(tick)")
}
```

This is more readable and maintainable than any delegate-based approach or closure chain.

### Example: Bridging NotificationCenter

```swift
func observeKeyboardNotifications() -> AsyncStream<Notification> {
    AsyncStream { continuation in
        let observer = NotificationCenter.default.addObserver(
            forName: UIResponder.keyboardWillShowNotification,
            object: nil,
            queue: .main
        ) { notification in
            continuation.yield(notification)
        }

        continuation.onTermination = { _ in
            NotificationCenter.default.removeObserver(observer)
        }
    }
}

Task {
    for await notification in observeKeyboardNotifications() {
        print("Keyboard will show: \(notification)")
    }
}
```

No more dealing with observer tokens or forgetting to unregister.

---

## 📜 AsyncSequence: Build Your Own Streams

### What is AsyncSequence?

`AsyncSequence` is the asynchronous equivalent of `Sequence`. It’s a protocol you conform to when you want to produce values *over time*, possibly involving suspensions.

It’s a low-level building block for things like:

- Custom data pipelines
- Throttled or debounced inputs
- Reading from a socket or file line-by-line
- Controlling backpressure and retries

### Anatomy of an AsyncSequence

To conform to `AsyncSequence`, you provide a type that returns an `AsyncIterator`. This iterator conforms to `AsyncIteratorProtocol`, which defines a `next()` async method.

### Custom Example: Debounced Input

```swift
struct DebouncedInput: AsyncSequence {
    typealias Element = String

    let inputs: [String]
    let interval: TimeInterval

    struct Iterator: AsyncIteratorProtocol {
        var inputs: [String]
        let interval: TimeInterval

        mutating func next() async -> String? {
            guard !inputs.isEmpty else { return nil }
            try? await Task.sleep(nanoseconds: UInt64(interval * 1_000_000_000))
            return inputs.removeFirst()
        }
    }

    func makeAsyncIterator() -> Iterator {
        Iterator(inputs: inputs, interval: interval)
    }
}
```

### Usage

```swift
let searchTerms = DebouncedInput(inputs: ["S", "Sw", "Swi", "Swif", "Swift"], interval: 0.5)

for await term in searchTerms {
    print("Searching for: \(term)")
}
```

Perfect for building custom debounced or throttled inputs, such as for a search bar.

---

## 🧠 When to Use What?

| Tool         | Best For                                                                 |
|--------------|--------------------------------------------------------------------------|
| `AsyncQueue` | Ensuring async tasks run **in order**, one at a time                     |
| `AsyncStream`| Bridging old-world callbacks or notifications into structured async flows|
| `AsyncSequence`| Building reusable async iterators with custom logic and flow control  |

---

## 🧪 Testing These Components

You can easily test these tools using `XCTestExpectation` or dependency injection.

For example, you can mock an `AsyncSequence` to simulate input for a ViewModel:

```swift
struct MockInputSequence: AsyncSequence {
    typealias Element = String

    let values: [String]

    struct Iterator: AsyncIteratorProtocol {
        var index = 0
        let values: [String]

        mutating func next() async -> String? {
            guard index < values.count else { return nil }
            defer { index += 1 }
            return values[index]
        }
    }

    func makeAsyncIterator() -> Iterator {
        Iterator(values: values)
    }
}
```

Use this in unit tests to simulate user input or background events.

---

## 🧭 Final Thoughts

Swift Concurrency is not just about replacing GCD. It’s about creating a fundamentally better model for expressing time-based, order-sensitive, and event-driven code.

- Use `AsyncQueue` when you need guaranteed serial execution.
- Reach for `AsyncStream` when bridging old APIs or dealing with events over time.
- Build `AsyncSequence` types when you need full control over async iteration.

When you embrace these tools, you’ll write code that’s not just cleaner—it’ll match the actual behavior and expectations of your apps. Fewer race conditions. No callbacks. Just clarity.

Swift's concurrency model still has its quirks, but tools like these make asynchronous programming feel like a natural part of the language.

Stay curious, and keep leveling up.

---