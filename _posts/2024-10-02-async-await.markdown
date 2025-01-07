---
layout: post
title: "Efficiently Running Multiple Async Tasks with Task Groups"
date: 2024-05-06 12:00:00 +0000
categories: [iOS, Swift, Async]
---

Swift’s concurrency model provides a set of tools for running tasks in parallel while keeping code clean and organized. One of the more powerful features of this model is **Task Groups**, which let you create and manage multiple concurrent tasks under a single structured context. In this article, we’ll dig deeper into Task Groups, demonstrate how to safely combine their results, and walk through a few practical examples.

## Why Task Groups Matter

When you need to work with multiple async operations—such as fetching data from different endpoints, loading resources from disk, or performing CPU-bound tasks—concurrency can significantly reduce total execution time. However, dealing with a flurry of asynchronous work without a clear structure can lead to deeply nested callbacks, complicated state management, and error-handling nightmares.

Task Groups solve these issues by:

- **Keeping concurrency structured**: You define each task within a dedicated group rather than scattering them throughout your code.
- **Easier error handling**: If one task throws an error, the entire group can be canceled.
- **Managed cancellation**: Cancelling a group’s parent task automatically cancels all its child tasks.
- **Result aggregation**: You can combine or merge results once all tasks finish, keeping data flow straightforward.

## Basic Anatomy of Task Groups

### withThrowingTaskGroup

The primary API we’ll use is `withThrowingTaskGroup(of:returning:body:)`. This function creates a throwing task group, which allows tasks to `throw` errors. The group automatically manages the lifecycle of these tasks.

```swift
try await withThrowingTaskGroup(of: T.self) { group in
    // group.addTask { ... }
    // group.addTask { ... }
    
    for try await taskResult in group {
        // process each result as it becomes available
    }
}
```

- **of: T.Type**: The type of value each task in the group returns.
- **group.addTask**: Creates a new child task within the group.
- **for try await taskResult in group**: Iterates over the results as child tasks finish.

## Example 1: Fetching Different Data Models in Parallel

A common scenario is loading multiple data models at once—such as `User` and `Post`—from different network endpoints.

```swift
struct User {
    let id: Int
    let name: String
}

struct Post {
    let id: Int
    let userId: Int
    let title: String
}

enum NetworkError: Error {
    case dataNotFound
}

func fetchData() async throws -> (User, [Post]) {
    var fetchedUser: User?
    var fetchedPosts: [Post]?

    try await withThrowingTaskGroup(of: Void.self) { group in
        // Task 1: Fetch user
        group.addTask {
            fetchedUser = try await fetchUser()
        }

        // Task 2: Fetch posts
        group.addTask {
            fetchedPosts = try await fetchPosts()
        }

        // Wait for all tasks to complete
        for try await _ in group {}
    }

    guard let user = fetchedUser, let posts = fetchedPosts else {
        throw NetworkError.dataNotFound
    }
    return (user, posts)
}

func fetchUser() async throws -> User {
    // Simulate network call
    // In a real scenario, you'd parse JSON and throw if decoding fails.
    await Task.sleep(1_000_000_000) // 1 second delay
    return User(id: 1, name: "Taylor")
}

func fetchPosts() async throws -> [Post] {
    // Simulate network call
    await Task.sleep(1_000_000_000) // 1 second delay
    return [
        Post(id: 1, userId: 1, title: "Concurrency in Swift"),
        Post(id: 2, userId: 1, title: "Task Groups in Depth")
    ]
}
```

**Key Points**:

1. We used `withThrowingTaskGroup(of: Void.self)` because each task sets a shared variable rather than returning a direct result.
2. If either `fetchUser()` or `fetchPosts()` throws an error, the entire group is canceled, and the error is propagated up.
3. After the group completes, we verify that both `fetchedUser` and `fetchedPosts` are non-nil, then return them.

## Example 2: Aggregating Results from Multiple Independent Calls

Sometimes, you might not just want to fetch data in parallel, but also handle each result independently and gather them into a collection.

```swift
/// A function that retrieves a single user by ID
func fetchUser(byID id: Int) async throws -> User {
    // Imagine a network call here
    return User(id: id, name: "User \(id)")
}

/// A function that retrieves users for multiple IDs
func fetchAllUsers() async throws -> [User] {
    let userIDs = [1, 2, 3, 4, 5]
    
    return try await withThrowingTaskGroup(of: User.self) { group in
        // Add tasks for each user ID
        for id in userIDs {
            group.addTask {
                return try await fetchUser(byID: id)
            }
        }
        
        var allUsers = [User]()
        
        // Collect each result
        for try await user in group {
            allUsers.append(user)
        }
        
        return allUsers
    }
}
```

**Key Points**:

1. This time, we use `withThrowingTaskGroup(of: User.self)` because each task returns a `User`.
2. We add a task for each user ID, allowing concurrent fetching for all of them.
3. We iterate over the group’s results, appending each `User` to `allUsers`.

## Example 3: Handling Cancellations and Errors

Task Groups integrate seamlessly with Swift’s cancellation model. If a task inside a group detects an error, you can cancel the group and propagate the error.

```swift
enum DownloadError: Error {
    case fileNotFound
}

/// A mock file download function that might throw an error
func downloadFile(named: String) async throws -> String {
    // Simulate potential error
    if named.contains("Invalid") {
        throw DownloadError.fileNotFound
    }
    
    // Simulate download
    await Task.sleep(500_000_000) // 0.5 second
    return "Contents of \(named)"
}

/// Orchestrates the download of multiple files
func downloadMultipleFiles(_ fileNames: [String]) async throws -> [String] {
    var fileContents = [String]()

    try await withThrowingTaskGroup(of: String.self) { group in
        for name in fileNames {
            group.addTask {
                return try await downloadFile(named: name)
            }
        }
        
        do {
            for try await content in group {
                fileContents.append(content)
            }
        } catch {
            // Cancels remaining tasks if an error is thrown
            group.cancelAll()
            throw error
        }
    }

    return fileContents
}
```

**Key Points**:

1. If any of the child tasks throws `fileNotFound`, the `catch` block is executed, and we call `group.cancelAll()`.
2. Canceling the group stops all remaining downloads that haven’t completed yet.
3. The error is then rethrown, giving your higher-level functions a chance to handle it.

## Performance Considerations

### Balancing Task Creation

Creating a large number of tasks can overwhelm system resources if done carelessly. While Task Groups manage concurrency, you may need to batch work in chunks to avoid saturating the system. For example, if you have a list of 10,000 items, consider processing them in smaller batches.

### Priorities

Child tasks inherit the priority of their parent by default, but you can adjust priorities if certain tasks need to run more urgently than others. Swift’s concurrency runtime will do its best to run higher-priority tasks first, though exact scheduling remains up to the system.

### Structured vs. Unstructured Tasks

In addition to Task Groups (structured concurrency), you can also create unstructured tasks with `Task { ... }`. Unstructured tasks don’t have the same built-in error propagation or cancellation behavior. It’s often advantageous to use Task Groups for complex workflows where you want a well-defined scope and robust error handling.

## Wrapping Up

Task Groups offer a powerful mechanism to run multiple concurrent operations without sacrificing maintainability. By adopting structured concurrency in your project, you can streamline the way you fetch data, process information, and handle errors. The ability to gather task results and manage errors at a single point keeps code both readable and resilient.

Practicing these patterns early in your project’s architecture can prevent headaches later on. Whether you’re parallelizing multiple network requests, performing heavy computations, or a mix of both, Task Groups can help you structure concurrency in a clear and efficient manner.
