---
layout: post
title:  "Basic architectural needs for an mobile app"
date:   2025-12-15 21:31:41 +0200
categories: swift architecture
---

<div style="text-align: center;">
  <img src="{{ site.baseurl }}/assets/images/statoscope-icon-1.png" alt="Statoscope Icon" style="width:300px; height:300px; object-fit:contain;">
</div>

This is a **recap** of several architectural principles we've explored before:

- **[Comprehensive logs]({{ site.baseurl }}swiftui/2024/09/02/ios-mobile-apps-comprehensive-traceable-logging-system.html)** — A structured logging system is essential for observability.
- **[Dependency injection]({{ site.baseurl }}swift/dependency/injection/2024/09/04/injection-of-external-environmental-dependencies-and-boundaries-in-swift.html)** — External dependencies should be injected, not hard-coded.
- **[Single entry point for actions]({{ site.baseurl }}swift/action/functional/2024/09/05/swift-ios-apps-single-entry-point-for-action-when-events.html)** — We favor a centralized dispatch system to make control flow and testing more predictable.
- **[Automated testing]({{ site.baseurl }}swift/automated/testing/2024/09/27/automated-testing-for-mobile-apps-tradeoffs-best-practices.html)** — The architecture should simplify the path to meaningful tests.

In addition to those pillars, we’ve found the following concepts increasingly relevant for iOS developers in 2024:

- A functional programming mindset.
- A unidirectional architecture composed of **State**, **Actions** and **Effects**.

---

## Introducing Statoscope

[Statoscope](https://github.com/InQBarna/statoscope) is a lightweight Swift library built on top of those principles. It offers a simple, scalable foundation for modeling app state, reacting to user or system events, and handling asynchronous operations with clarity and control.

It helps you:

- Organize your application into **stateful contexts**
- React to inputs through a central **dispatcher**
- Trigger and observe **effects** that run asynchronously
- Structure your features into **scopes** for maintainability and testability

---

## Basic Concepts

### 1. State

State is the single source of truth for your feature or screen. In Statoscope, it's modeled as a simple `struct`, with no special requirements beyond being equatable and observable:

```swift
struct State: Equatable {
    var username: String = ""
    var isLoading: Bool = false
    var error: String?
}
````

### 2. When (Actions)

Instead of generic enums or types, we use a clearly defined set of actions called `When`, which model user or system events:

```swift
enum When {
    case onAppear
    case didTapRetry
    case usernameChanged(String)
}
```

This naming helps with test traceability: `"When user taps retry"` becomes an explicit trigger.

### 3. Effect

An effect represents any asynchronous operation or side effect. These are modeled using `async` functions or `Effect` structs:

```swift
let loadProfileEffect: Effect<State, When> = .init { state, send in
    state.isLoading = true
    do {
        let profile = try await profileService.fetch()
        send(.didLoadProfile(profile))
    } catch {
        send(.didFail(error.localizedDescription))
    }
    state.isLoading = false
}
```

You can inject dependencies using your own injection mechanism (e.g. environment structs, closures, or protocol witnesses).

---

## Scoping Your State

Statoscope encourages **composability** by allowing features to be scoped. A child feature defines its own `State`, `When`, and `Effect`, and can be embedded into a larger domain with mappings:

```swift
struct AppState {
    var login: Login.State
    var profile: Profile.State
}
```

You can lift child logic using `.mapState`, `.mapWhen`, and `.mapEffect` helpers to compose effects without tight coupling.

---

## One Dispatcher

Each feature declares a dispatcher responsible for:

* Receiving a `When`
* Mutating the `State`
* Returning an `Effect?`

This means all logic is centralized and easily testable:

```swift
let dispatcher: Dispatcher<State, When> = .init { state, when in
    switch when {
    case .onAppear:
        return .loadProfile
    case .usernameChanged(let newValue):
        state.username = newValue
        return nil
    }
}
```

You can pass `dispatcher.dispatch` to your view, or use `@ObservedScope` for reactive SwiftUI integration.

---

## SwiftUI Integration

Statoscope provides minimal integration with SwiftUI:

```swift
@ObservedScope var scope: Scope<State, When>

TextField("Username", text: $scope.username)
Button("Retry") {
    scope.dispatch(.didTapRetry)
}
```

You can observe only part of the state or structure nested scopes within a view hierarchy.

---

## Why We Built It

Most architecture libraries either:

* Are too rigid or boilerplate-heavy
* Force coupling with third-party frameworks
* Hide useful capabilities behind black-box abstractions

Statoscope is intentionally simple:

* You can **see and test** all state transitions
* You are **free to structure** state however you want
* You retain **control over your effects** and dependencies

It works great with Combine, Swift Concurrency, SwiftUI, or any test toolset — but doesn't force you to adopt anything new.

---

## Wrap-Up

Statoscope is now powering several of our apps. It has improved test coverage, reduced bugs, and made it easier to onboard new developers thanks to its clear mental model.

Let us know how you're using it — and what you'd love to see next.
