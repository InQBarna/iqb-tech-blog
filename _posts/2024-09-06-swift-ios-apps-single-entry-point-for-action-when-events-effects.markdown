---
layout: post
title:  "Swift iOS App Single Entry Point for Action/When Events: Effects"
date:   2024-09-06 21:31:41 +0200
categories: swift action functional
---

This is the second part of the article [Swift iOS App Single Entry Point for Action/When Events]({{ site.baseurl }}swift/action/functional/2024/09/05/swift-ios-apps-single-entry-point-for-action-when-events.html). Let's go for some recap:

### **Why Use a Single Entry Point for Actions?**

1. **Centralized Action/`When` Handling**: By funneling all app events through a single entry point,
 you gain a centralized place to manage business logic, making the app easier to understand and extend. 
  
2. **Enhanced Testability**: A single entry point allows you to send mock events and verify state after 
sending, leading to robust and isolated testing scenarios.

3. **Centralized logging, analytics and more**: Centralizing events means you can easily log every event, enhancing your 
analytics, crash reporting, and other global capabilities

4. **Separation of Concerns**: Clearly separates business logic from side effects (e.g., network requests,
 database writes), which can be modeled as independent entities that trigger further actions.

### **How to build single synchronous entry point for `Effect`s**

In the previous post we learned how to define the `When` enum and the handler method. Let's now
consider how to design an asynchronous action keeping in mind the features we liked in the previous 
post: synchronicity and enhanced testability via event + state check.

#### **Step 1: Define the `When` Enum**

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

See how for most of asynchronous tasks we have 2 synchronous `When` cases: one for the effect 
triggering and another one for the effect completion. This may seem artificial, but cleanly 
states that **ANYTHING** can happen while the `Effect` is running, forcing the developer to 
consider the effect completion to complete at **any unexpected moment in time**

#### **Step 2: Define a Single Entry Point to Handle `When`s**

As a baseline, let's first handle synchrhnous code: normal `When`s and `Effect`s completions.
And mark TODOs for `Effect`s triggering:

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
                /// TODO: Handle error. Out of the scope of the example
                break
            }
            self.loading = false
        case .userTypesNewEmail(let newEmail):
            self.email = newEmail
        case .userTapsChangeEmailButton:
            loading = true
            /// TODO: Trigger WRITE effect
        }
    }
}
```

#### **Step 3: Build the `Effect`s handler**

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

#### **Step 4: Use the `Effect`s handler**

Let's grab the previous ViewModel and complete the TODOs.

 ```swift
final class MyViewModel: ObservableObject {
    @Published var loading: Bool = false
    @Published var email: String = ""
    let effectHandler: EffectHandler = fakeEffectHandler
    func handle(_ when: When) throws {
        switch when {
        case .systemShowsUserProfileView:
            loading = true
            // Trigger the effect ant push the result back to the handler method
            Task { [weak self, effectHandler] in
                let profileResponseWhen = try await effectHandler(.performGetProfileNetworkRequest)
                try self?.handle(profileResponseWhen)
            }
        case .networkFinishesLoadingProfile(let profile),
             .networkFinishesSavingProfile(let profile):
            // Check completion occurs in an expected moment, otherwise fail
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

Please note the following in the previous example

**1- Triggering:** The effect is triggered asynchrnously and completion is feeded back into
the handler method. Using this approach, we can still consistently work with a single entry
point for our business logic. Since most of the time `Effect`s are out of our business logic 
(they perform network, system and other operations from third party libraries), we can 
focus on our `When` testing and keep all of our tested business login inside our `When`s 
handler

**2- Completion and state check:** Check current state on handler completion. This may be an
overhead, but some developers fail to understand ANYTHING can happen during trigger and 
completion of an effect. If `Effect` completion occurs on an unexpected situation, this should
 be thrown as an exception and captured on your logging/reporting system. 

#### **Step 5: Write some tests. Will be step 1 in the following examples 😀**

Centralized `Effect`s running allows you to disable them completely for testing. In the following
example a naive throwing handler is implemented for the tests and `Effect`s completion
`When` events can be inline defined and tested

```swift
func testUserStoryWithEffects() {
    TestRunner<MyViewModel>.GIVEN {
        let vm = MyViewModel()
        // We don't want effects to be triggered during these tests
        //  we may build a more elegant solution, but it's out of the scope
        //  of this example
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

### **Benefits of This Approach**

1. **Structured Flow of `When`s**: The single entry point structure ensures a well-defined flow of `when`s,
 making the code easier to follow, log, test and maintain.

2. **Powerful Testing Capabilities**: Test your business logic in isolation by injecting mock `when`s and 
asserting the expected outcomes without dealing with real side effects. `Effect`s are controlledly completed by
new modelled `when` completion events in the test code.

3. **Clean Separation of Business Logic and Side `Effect`s**: By modeling side `Effect`s separately and handling their results 
as `when`s, your business logic remains pure and testable.
   
### **Conclusion**

Implementing a single entry point for `when` handling in Swift iOS applications provides a powerful way to manage business
logic. It centralizes control, enhances testability, and separates the complexities of side effects, making your code cleaner,
more maintainable, and easier to debug. 

This approach not only simplifies how inputs and outputs are handled but also lays the groundwork for advanced features 
like automated testing, comprehensive logging, and structured error reporting. By embracing this architecture, you set 
 strong foundation for building robust, scalable, and reliable iOS applications.
