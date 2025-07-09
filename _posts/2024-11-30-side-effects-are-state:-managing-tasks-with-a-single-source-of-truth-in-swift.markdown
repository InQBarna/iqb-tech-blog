---
layout: post
title:  "Side Effects Are State: Managing Tasks with a Single Source of Truth in Swift"
date:   2024-11-30 21:31:41 +0200
categories: swift architecture
---

![Image]({{ site.baseurl }}/assets/images/mobile-development-effects-are-state.png)

In modern iOS development with Swift, concurrency is everywhere. Whether you’re fetching data, uploading files, or processing images, `Task` has become the de facto way to perform asynchronous work.

But there's a pattern we often see in codebases: async work is treated as something separate from the app’s state.

```swift
var isLoading = false
private var task: Task<Response, Never>?
```

These two variables try to describe a single concept — “work is in progress” — but each has a different concern. `isLoading` is for the UI. `task` is for cancellation. You now have *two sources of truth* for one fact, and the cost of inconsistency is high.

In this post, we’ll explore the idea that **side effects are state**, and how managing them explicitly with an `EffectsState` structure improves clarity, consistency, and control.

---

### **Why Treat Tasks as State?**

* ✅ To show accurate UI feedback
* ✅ To query if an effect is running, in order to ...
* ✅ ... cancel them later
* ✅ ... avoid duplicate requests

---

## **Introducing `EffectsState`**

Instead of tracking both flags and `Task` references, we propose having a single source of truth: a shared structure that manages running tasks. For the sake of simplicity, the following example only allows 1 running task/effect per EffectId, AND our first example is not thread-safe, wait for the following examples.

```swift
final class EffectsState<EffectID: Hashable> {
    private var tasks: [EffectID: Task<Void, Error>] = [:]

    deinit {
        cancelAll()
    }

    func isRunning(_ id: EffectID) -> Bool {
        tasks[id] != nil
    }

    func start(
        _ id: EffectID,
        operation: @escaping @Sendable () async throws -> Void
    ) {
        self.tasks[id] = Task { [weak self] in
            try await operation()
            self?.tasks.removeValue(forKey: id)
        }
    }

    func cancel(_ id: EffectID) {
        tasks[id]?.cancel()
        tasks.removeValue(forKey: id)
    }

    func cancelAll() {
        tasks.values.forEach { $0.cancel() }
        tasks.removeAll()
    }
}
```

---

## **Usage Example**

Let’s say we’re viewing a user profile and want to fetch profile data without duplication:

```swift
enum EffectID {
    case loadProfile
}

class MyModel: ObservableObject {
    let effects = EffectsState<EffectID>()
    let api: API = API()
    @State var profile: Profile = Profile(userName: "")
    
    func loadUserProfile() throws {

        // Do not execute simultaneous equal effects,
        //  we may either effects.cancel(.loadProfile)
        guard !effects.isRunning(.loadProfile) else {
            return
        }
        
        effects.start(.loadProfile) { [api] in
            let profile = try await api.fetchProfile()
            await MainActor.run {
                self.profile = profile
            }
        }
    }
}
```

This:

* Prevents double calls (`isRunning`)
* Manages task lifecycle
* Cleans up automatically when the task finishes
* Cancels automatically on `deinit`

---

## **Building the UI**

UI feedback can be calculated from the app's state. Remember SwiftUI's dogma `“A view is just the rendering of your state — nothing more, nothing less.”`

```swift
extension MyModel {
    struct Shows {
        let userName: String
        let isLoading: Bool
    }
    var shows: Shows {
        Shows(
            userName: profile.userName,
            isLoading: effects.isRunning(.loadProfile)
        )
    }
}
```

And further code, following the patterns proposer earlier in the blog:

```swift
struct MyView: View {
    var shows: EffectsAreState.MyModel.Shows
    var body: some View {
        Text(shows.userName)
        if shows.isLoading {
            ProgressView()
        }
    }
}
struct MyViewModel: View {
    @ObservedObject var model: MyModel = MyModel()
    var body: some View {
        MyView(shows: model.shows)
    }
}
```

---

## **Canceling Specific Effects**

Suppose the user taps "Cancel" while a background operation is running:

```swift
effects.cancel(.loadProfile)
```

Or if we want to cancel everything (e.g. when a screen disappears):

```swift
effects.cancelAll()
```

---

## **Keeping Thread Safety in Mind**

Tasks might start from:

* The main actor (UI-triggered actions)
* Background threads (system events, observers, etc.)

So, we need a more sophisticated EffectsState, thread safe, ready for corner cases, solid and fully tested.

We consider this a key point that should exist on every application, as part of the main architecture/framework. At some point taks management may be necessary, and using ad-hoc solutions on every part of the code may result in duplicated and no solid task management.

Please see a first step (still not complete but showing the therad-safety point) below:

* Use a `DispatchQueue` with `.barrier` for write safety
* Use `.sync` for reads (safe from any thread)
* Avoid direct mutation of shared dictionaries on other threads

```swift
final class EffectsState<EffectID: Hashable> {
    private let lock = DispatchQueue(label: "EffectsState.lock", attributes: .concurrent)
    private var tasks: [EffectID: Task<Void, Error>] = [:]

    deinit {
        cancelAll()
    }

    func isRunning(_ id: EffectID) -> Bool {
        lock.sync {
            tasks[id] != nil
        }
    }

    func start(
        _ id: EffectID,
        operation: @escaping @Sendable () async throws -> Void
    ) {
        let task = Task { try await operation() }

        lock.async(flags: .barrier) {
            self.tasks[id] = task
        }

        Task { [weak self] in
            _ = try await task.value
            self?.remove(id)
        }
    }

    func cancel(_ id: EffectID) {
        lock.sync {
            tasks[id]?.cancel()
        }
        remove(id)
    }

    func cancelAll() {
        lock.sync {
            tasks.values.forEach { $0.cancel() }
        }
        lock.async(flags: .barrier) {
            self.tasks.removeAll()
        }
    }

    private func remove(_ id: EffectID) {
        lock.async(flags: .barrier) {
            self.tasks.removeValue(forKey: id)
        }
    }
}
```

## **EffectsState in `Statoscope`**

We've build `Statoscope`as a basic library for mobile development, so of course `Statoscope` includes its own `EffectsState`. However it's evolved to a much more complicates example, covering

* Effects are not immediately running, instead an EffectsRunner orchestrates them
* Effects are either anonymous closures or types inheriting from a protocol `Effect`. Only concrete types can be queried or cancelled by type (instead of id). Anonymous have been created for simplicity.
* Effect types replaces `EffectId`s, so multiple effects with the same type can be executed if needed.
* Thread safety is built on top of modern Swift Actors

You can see the EffectsState interface [here](https://github.com/InQBarna/statoscope/blob/main/Sources/Statoscope/Effects/EffectsState.swift)

---

## **Conclusion**

If your async work matters, treat it like state. By managing side effects through a centralized `EffectsState` structure, you gain:

* A single, consistent place to track ongoing work
* Built-in cancellation
* Easier testability and UI feedback
* Cleaner resource management

This doesn’t eliminate complexity — it acknowledges it and gives it a name.

Let your app declare what’s happening, not just what it’s displaying.
