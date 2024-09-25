---
layout: post
title:  "Swift iOS App Single Entry Point for Action/When Events (Part2): Effects"
date:   2024-09-06 21:31:41 +0200
categories: swift action functional
---

This is the second part of the article [Swift iOS App Single Entry Point for Action/When Events]({{ site.baseurl }}swift/action/functional/2024/09/05/swift-ios-apps-single-entry-point-for-action-when-events.html). Let’s recap some key points from the previous post:

## **Why Use a Single Entry Point for Actions?**

1. **Centralized Action/`When` Handling**: By funneling all app events through a single entry point, you gain a centralized place to manage business logic, making the app easier to understand and extend.
2. **Enhanced Testability**: A single entry point allows you to send mock events and verify the state afterward, enabling robust and isolated testing scenarios.
3. **Centralized Logging, Analytics, and More**: Centralizing events makes it easier to log every event, enhancing your app’s analytics, crash reporting, and other global capabilities.
4. **Separation of Concerns**: This approach clearly separates business logic from side effects (e.g., network requests, database writes), which can be modeled as independent entities that trigger further actions.

---

## **How to Build a Single Synchronous Entry Point for asynchronous `Effect`s**

In the previous post, we learned how to define the `When` enum and the handler method. Let's now consider how to design an asynchronous action while keeping in mind the features we valued in the first article: synchronicity and enhanced testability via event + state checks.

### **Step 1: Define the `When` Enum**

```swift
enum When: Codable {
    case systemShowsUserProfileView
    // A completion `When` is created for the asynchronous network READ effect
    case networkFinishesLoadingProfile(Result<[String: String], CodableError>)
    case userTypesNewEmail(userEmail: String)
    case userTapsChangeEmailButton
    // A completion `When` is created for the asynchronous network WRITE effect
    case networkFinishesSavingProfile(Result<[String: String], CodableError>)
}
```

For most asynchronous tasks, we have two synchronous `When` cases: one for the effect triggering and another for the effect completion. While this might seem redundant, it emphasizes that **anything** can happen while the `Effect` is running, forcing developers to consider the effect’s completion at **any unexpected moment in time**.

### **Step 2: Define a Single Entry Point to Handle `When`s**

As a baseline, let's handle synchronous code first—normal `When`s and `Effect` completions—while leaving TODOs for `Effect` triggering.

```swift
final class MyViewModel: ObservableObject {
    @Published var loading: Bool = false
    @Published var email: String = ""
    func handle(_ when: When) {
        switch when {
        case .systemShowsUserProfileView:
            loading = true
            // TODO: trigger READ effect
        case .networkFinishesLoadingProfile(let profile),
             .networkFinishesSavingProfile(let profile):
            switch profile {
            case .success(let jsonProfile):
                if let email = jsonProfile["email"] {
                    self.email = email
                }
            case .failure:
                // TODO: Handle error. Out of the scope of the example
                break
            }
            self.loading = false
        case .userTypesNewEmail(let newEmail):
            self.email = newEmail
        case .userTapsChangeEmailButton:
            loading = true
            // TODO: Trigger WRITE effect
        }
    }
}
```

### **Step 3: Build the `Effect`s Handler**

```swift
// We're using an enum to define which effects we can handle
enum Effect {
    case performGetProfileNetworkRequest
    case performSaveProfileNetworkRequest(email: String)
}
// A closure is enough to abstract our Handler for dependency injection
typealias EffectHandler = (Effect) async throws -> When

// Creating a fake handler implementing the abstraction for the example
static var fakeEffectHandler: EffectHandler = { effect in
    // Simulate network delay and response
    try await Task.sleep(nanoseconds: 1_000_000_000)
    switch effect {
    case .performGetProfileNetworkRequest:
        return .networkFinishesLoadingProfile(
            .success([
                "name": "John Doe", "email": "john.doe@example.com"
            ])
        )
    case .performSaveProfileNetworkRequest(let email):
        return .networkFinishesSavingProfile(
            .success([
                "name": "John Doe", "email": email
            ])
        )
    }
}
```

### **Step 4: Use the `Effect`s Handler**

Now, we’ll return to the ViewModel and complete the TODOs.

```swift
final class MyViewModel: ObservableObject {
    @Published var loading: Bool = false
    @Published var email: String = ""
    let effectHandler: EffectHandler = fakeEffectHandler
    func handle(_ when: When) throws {
        switch when {
        case .systemShowsUserProfileView:
            loading = true
            // Trigger the effect and push the result back to the handler method
            Task { [weak self, effectHandler] in
                let profileResponseWhen = try await effectHandler(.performGetProfileNetworkRequest)
                try self?.handle(profileResponseWhen)
            }
        case .networkFinishesLoadingProfile(let profile),
             .networkFinishesSavingProfile(let profile):
            // Ensure completion occurs at an expected moment, otherwise throw an error
            guard loading else {
                throw InvalidStateError()
            }
            switch profile {
            case .success(let jsonProfile):
                if let email = jsonProfile["email"] {
                    self.email = email
                }
            case .failure:
                break
            }
            self.loading = false
        case .userTypesNewEmail(let newEmail):
            self.email = newEmail
        case .userTapsChangeEmailButton:
            loading = true
            Task { [weak self, email, effectHandler] in
                let profileResponseWhen = try await effectHandler(.performSaveProfileNetworkRequest(email: email))
                try self?.handle(profileResponseWhen)
            }
        }
    }
}
```

#### **Key Observations:**

1. **Triggering**: The effect is triggered asynchronously, and the completion is fed back into the handler method. This allows us to work consistently with a single entry point for business logic, while focusing testing efforts on the `When`s without needing to directly test the effects.
2. **Completion and State Check**: The state is checked upon effect completion. While this introduces some overhead, it ensures developers understand that **anything** can happen between the triggering and completion of an effect. If the completion occurs in an unexpected state, it’s treated as an exception.

### **Step 5: Write Some Tests**

Centralized effect handling allows you to disable them completely during testing. Below is an example test using a naive throwing handler to simulate the absence of effects.

```swift
func testUserStoryWithEffects() {
    TestRunner<MyViewModel>.GIVEN {
        let vm = MyViewModel()
        // Disable effects during testing
        vm.effectHandler = { _ in throw "No effects during unit tests" }
        return vm
    }
    .WHEN(.systemShowsUserProfileView)
    .THEN(\.loading, is: true)
    .THEN(\.email, is: "")
    .WHEN(.networkFinishesLoadingProfile(.success(["email": "john@example.com"])))
    .THEN(\.email, is: "john@example.com")
    .THEN(\.loading, is: false)
    .WHEN(.userTypesNewEmail(userEmail: "john.dohe@example.com"))
    .THEN(\.email, is: "john.dohe@example.com")
    .WHEN(.userTapsChangeEmailButton)
    .THEN(\.loading, is: true)
    .THEN(\.email, is: "john.dohe@example.com")
    .WHEN(.networkFinishesSavingProfile(.success(["email": "john.dohe@example.com"])))
    .THEN(\.email, is: "john.dohe@example.com")
    .THEN(\.loading, is: false)
    .runTests()
}
```

---

### **Benefits of This Approach (recap from Part 1)**

1. **Structured Flow of `When`s**: The single entry point structure ensures a well-defined flow of `When` events, making the code easier to follow, log, test, and maintain.
2. **Powerful Testing Capabilities**: Test your business logic in isolation by injecting mock `When`s and asserting the expected outcomes without real side effects. `Effect` completions are modeled as new `When` events that can be directly tested.
3. **Clean Separation of Business Logic and Side `Effect`s**: By modeling side effects separately and handling their results as `When` events, your business logic remains pure and testable.
4. **`Effect`s timeline awareness**: Responding to a effect completion in a completely new context (the handler method) enforces an awareness of asynchronicity on the developer, and checking for actual state feels necessary (which has allways been)
5. **Clean Declaration of Acceptance Criteria**: By modeling `state` and `when` separately, the expectations from the code are clear for everyone involved.

### **CONs of This Approach**

1. **Effects completion context handling complexity**: Inline concurrent code like async/await makes context handling simpler by keeping everything in one block, reducing the need for manual state management or tracking across separate events.
2. **Scaling complexity**: While a single entry point can leverage many features, it needs an important review when the app grows. Though scaling this approach can be done, the scaling architecture needs to be well designed and planned.
3. **Risk of Over-centralization**: If too much logic is placed in the single entry point, it can turn into a "God object," complicating maintenance.

---

### **Conclusion**

Implementing a single entry point for `When` handling in Swift iOS applications is a powerful way to manage business logic. It centralizes control, enhances testability, and separates the complexities of side effects, making your code cleaner, more maintainable, and easier to debug.

This approach simplifies how inputs and outputs are handled while also laying the groundwork for advanced features like automated testing, comprehensive logging, and structured error reporting. By embracing this architecture, you create a strong foundation for building scalable and reliable iOS applications.
