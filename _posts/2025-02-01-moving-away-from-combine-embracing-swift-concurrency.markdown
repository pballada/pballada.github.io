---
layout: post
title: "Moving Away from Combine: Embracing Swift Concurrency"
date: 2025-02-01
categories: [iOS, Swift]
header_image: /assets/images/swift-concurrency.jpg
---

Over the past year, I’ve been migrating away from Combine and fully embracing Swift Concurrency across production codebases. Combine was powerful and expressive—but it also introduced complexity, cryptic error messages, and steep onboarding costs for junior developers. Swift Concurrency, on the other hand, feels like Swift finally becoming what it was meant to be: a language with built-in, ergonomic, and native async capabilities.

This post is a deep dive into real-world Combine patterns and how I’ve translated them to modern Swift Concurrency. This isn't just surface-level—it's about bringing clarity and long-term maintainability to async workflows.

---

## Why Replace Combine?

Combine helped bridge the async gap before Swift had built-in tools to do it. But Combine:

- Is difficult to debug  
- Requires verbose type erasure  
- Relies heavily on chains of cryptic operators  
- Encourages overengineering for simple workflows  

Swift Concurrency, on the other hand:

- Is native to the language  
- Reads top-to-bottom  
- Encourages structured thinking  
- Has better tooling and diagnostics  

Let’s explore what that looks like in real-world code.

---

## 1. Standard Networking Request

### 🧱 Combine

```swift
func fetchItems() -> AnyPublisher<[Item], Error> {
    URLSession.shared.dataTaskPublisher(for: url)
        .map(\.data)
        .decode(type: [Item].self, decoder: JSONDecoder())
        .receive(on: DispatchQueue.main)
        .eraseToAnyPublisher()
}
```

This chain performs a network request, decodes the data, switches to the main thread, and erases the publisher type. It’s concise but hard to debug, especially if the pipeline gets longer or you need intermediate breakpoints.

### 🔁 Swift Concurrency

```swift
func fetchItems() async throws -> [Item] {
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode([Item].self, from: data)
}
```

✅ **Why this is better**:  
- Straight-line code mirrors the logic exactly.  
- Easier to debug and catch decode errors.  
- No need to jump between `.map`, `.decode`, `.receive`, etc.  

---

## 2. Retrying on Failure

### 🧱 Combine

```swift
somePublisher
    .retry(3)
    .sink(receiveCompletion: {...}, receiveValue: {...})
    .store(in: &cancellables)
```

### 🔁 Swift Concurrency

```swift
func fetchWithRetry<T>(_ operation: @escaping () async throws -> T, retries: Int = 3) async throws -> T {
    var attempts = 0
    while true {
        do {
            return try await operation()
        } catch {
            attempts += 1
            if attempts >= retries { throw error }
            try await Task.sleep(nanoseconds: 500_000_000)
        }
    }
}
```

✅ **Why this is better**:  
- You control retry logic, backoff timing, and error filtering.  
- Works with `try await` operations directly.  

---

## 3. Combining Multiple Values (`combineLatest`)

### 🧱 Combine

```swift
Publishers.CombineLatest($email, $password)
    .map { !$0.isEmpty && !$1.isEmpty }
    .assign(to: &$isLoginEnabled)
```

### 🔁 Swift Concurrency

```swift
@Observable class LoginViewModel {
    var email: String = ""
    var password: String = ""
    var isLoginEnabled: Bool {
        !email.isEmpty && !password.isEmpty
    }
}
```

✅ **Why this is better**:  
- Swift Concurrency integrates deeply into `@Observable` in SwiftUI.  
- You get dependency-tracked recomputation without subscriptions.  

---

## 4. Merging Multiple Streams

### 🧱 Combine

```swift
publisherA.merge(with: publisherB)
```

### 🔁 Swift Concurrency

```swift
func merge<T>(_ streams: [AsyncStream<T>]) -> AsyncStream<T> {
    AsyncStream { continuation in
        for stream in streams {
            Task {
                for await value in stream {
                    continuation.yield(value)
                }
            }
        }
    }
}
```

✅ **Why this is better**:  
- True concurrent emission.  
- You can customize timing, priority, or filtering.  

---

## 5. Periodic Polling

### 🧱 Combine

```swift
Timer.publish(every: 10, on: .main, in: .common)
    .autoconnect()
    .flatMap { _ in self.fetchStatus() }
    .sink { self.status = $0 }
```

### 🔁 Swift Concurrency

```swift
func startPolling() {
    Task {
        while !Task.isCancelled {
            do {
                let result = try await fetchStatus()
                await MainActor.run {
                    self.status = result
                }
            } catch {
                print("Polling failed: \(error)")
            }
            try await Task.sleep(nanoseconds: 10_000_000_000)
        }
    }
}
```

✅ **Why this is better**:  
- Lifecycle-aware with cancelation built-in.  
- Clear and testable logic.  

---

## 6. Form Validation

### 🧱 Combine

```swift
Publishers.CombineLatest3($name, $email, $password)
    .map { ... }
    .assign(to: &$isValid)
```

### 🔁 Swift Concurrency

```swift
@Observable class FormViewModel {
    var name = ""
    var email = ""
    var password = ""

    var isValid: Bool {
        !name.isEmpty && email.contains("@") && password.count > 5
    }
}
```

✅ **Why this is better**:  
- You don’t need to manage subscriptions for validation.  
- The logic is co-located and declarative.  

---

## 7. Bridging Combine to Concurrency (and Back)

### Combine → AsyncSequence

```swift
for await value in somePublisher.values {
    print("Received \(value)")
}
```

### Swift Concurrency → Publisher

```swift
func asyncToPublisher<T>(_ operation: @escaping () async throws -> T) -> AnyPublisher<T, Error> {
    Deferred {
        Future { promise in
            Task {
                do {
                    let result = try await operation()
                    promise(.success(result))
                } catch {
                    promise(.failure(error))
                }
            }
        }
    }.eraseToAnyPublisher()
}
```

✅ **Why this is better**:  
- Enables smooth, piecemeal migration from Combine.  
- Keeps interop alive for SDKs and older code.  

---

## 8. Debounce with Swift Concurrency

### 🧱 Combine

```swift
$text
    .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
    .removeDuplicates()
    .sink { self.search($0) }
```

### 🔁 Swift Concurrency

```swift
func debounce<Input: Equatable>(
    for interval: TimeInterval,
    in stream: AsyncStream<Input>
) -> AsyncStream<Input> {
    AsyncStream { continuation in
        var lastValue: Input?
        var debounceTask: Task<Void, Never>?

        Task {
            for await value in stream {
                if value == lastValue { continue }
                lastValue = value

                debounceTask?.cancel()
                debounceTask = Task {
                    try? await Task.sleep(nanoseconds: UInt64(interval * 1_000_000_000))
                    continuation.yield(value)
                }
            }
        }

        continuation.onTermination = { _ in
            debounceTask?.cancel()
        }
    }
}
```

✅ **Why this is better**:  
- Full control over debouncing behavior.  
- Easily reused across different types and contexts.  

---

## 9. Throttle with Swift Concurrency

### 🧱 Combine

```swift
$scrollOffset
    .throttle(for: .milliseconds(200), scheduler: RunLoop.main, latest: true)
    .sink { self.update($0) }
```

### 🔁 Swift Concurrency

```swift
func throttle<T>(
    for interval: TimeInterval,
    in stream: AsyncStream<T>
) -> AsyncStream<T> {
    AsyncStream { continuation in
        var lastEmission = Date.distantPast

        Task {
            for await value in stream {
                let now = Date()
                let elapsed = now.timeIntervalSince(lastEmission)

                if elapsed >= interval {
                    continuation.yield(value)
                    lastEmission = now
                }
            }
        }
    }
}
```

✅ **Why this is better**:  
- Adjust logic per use case (leading vs trailing, skipping vs queueing).  
- Easier to integrate with animation cycles or performance goals.  

---

## General Comparison

| Goal                        | Combine                           | Swift Concurrency             |
|-----------------------------|-----------------------------------|-------------------------------|
| Async logic                 | Declarative pipeline               | Structured flow               |
| Retry logic                 | Limited operator config            | Fully customizable            |
| Debounce / Throttle         | Built-in                          | Easily built + tunable        |
| CombineLatest / Zip         | Complex setup                     | Often unnecessary             |
| Cancellation                | Manual (`AnyCancellable`)         | Built-in (`Task`)             |
| Debuggability               | Poor stack traces                 | Native and understandable     |
| SwiftUI compatibility       | Yes, but bolted-on                | Designed for it               |

---

## Final Thoughts

Combine was powerful, but it’s time to move on.

Swift Concurrency brings clarity, safety, and a sense of ownership back to async code. It removes the need for a mental gymnastics course every time you want to do something simple like debounce user input or merge network requests.

I’ve moved almost all my production code to Swift Concurrency—and I haven’t looked back.

Next up: how to build reusable `AsyncStream` utilities, structure `Actors` for async-safe state, and use `@Observable` for clean MVVM in SwiftUI.

Until then—refactor one Combine pipeline. Just one.  
You'll feel the difference immediately.

