---
layout: post
title:  "Swift iOS App Single Entry Point for Action/When Events"
date:   2024-09-05 21:31:41 +0200
categories: swift action functional
---

Designing the business logic of a Swift iOS application around a single entry point can greatly simplify 
how actions are managed. This approach helps structure code for better testing, logging, crash reporting,
 and overall maintainability. By modeling side effects separately, you ensure a clean separation of concerns,
  enhancing the predictability and scalability of your app’s architecture.

### **Why Use a Single Entry Point for Actions?**

1. **Centralized Action/When Handling**: By funneling all app events through a single entry point,
 you gain a centralized place to manage business logic, making the app easier to understand and extend.
  
2. **Enhanced Testability**: A single entry point allows you to send mock events and verify state after 
sending, leading to robust and isolated testing scenarios.

3. **Logging and Analytics**: Centralizing events means you can easily log every event, enhancing your 
analytics, crash reporting, and other global capabilities.

4. **Separation of Concerns**: Clearly separates business logic from side effects (e.g., network requests,
 database writes), which can be modeled as independent entities that trigger further actions.

### **Defining the Single Entry Point Using an Enum**

Out approach to this solutios is to define a `When` enum that encapsulates all possible events in the app, or 
in your current app scope. Each case of the enum can have associated values to pass necessary data, allowing 
the business logic handler to react appropriately.

#### **Step 1: Define the `When` Enum**

Create an `When` enum that represents all possible events in your application/scope. Each case 
can have associated values that represent the data required for that `when`.

```swift
// Define the When enum with associated values
enum When: Codable {
    case userTypesNewEmail(userEmail: String)
    case userTapsChangeEmailButton
}
```

#### **Step 2: Define a Single Entry Point to Handle Whens**

Create a method (often in a ViewModel, Controller, or Service) that serves as the single entry point to handle these `when`s.

```swift
final class MyViewModel: ObservableObject {
    @Published var email: String = ""
    func handle(_ when: When) {
        switch when {
        case .userTypesNewEmail(let newEmail):
            self.email = newEmail
        case .userTapsChangeEmailButton:
            print("Saving data: \(email)")
        }
    }
}
```

#### **Step 3: Adding middlewares**

Middleware is the term used in some single entry point architectures to receive and/or
transform and forward the `when` events. Some really interesting features are easy to implement
on this approach, like

- Logs and Reporting: local/remote logs, analytics and crash reporting are easy to implement.

- Testing: testing becomes straightforward with event + state checking.

- Reproducibility: recording the received events to reproduce an ongoing error can be done.

- And much more... TODO


#### ***Example 3.1: Analytics and crash reporting middlewares**

```swift
// Create the necessary code to transform a when into an struct thaht your analytics platform can handle
struct AnalyticsProvider {

    struct AnalyticsEvent {
        let name: String
        let parameters: [String: Any]
    }

    static func buildAnalyticsEvent(when: AnalyticsWhen) throws -> AnalyticsEvent {
        let encoded = try JSONEncoder().encode(when)
        guard let jsonDict = try JSONSerialization.jsonObject(with: encoded, options: []) as? [String: Any],
                let first = jsonDict.keys.first else {
            throw "When \(when) is unexpectedly encoded as json "
                    + String(describing: String(data: encoded, encoding: .utf8))
        }
        return AnalyticsEvent(
            name: first,
            parameters: jsonDict[first] as? [String: Any] ?? [:]
        )
    }

    static func analyticsAndCrashReportMiddleware<T: AnalyticsWhen>(
        when: T,
        next: ((T) throws -> Void)
    ) rethrows {
        do {
            if let evt = try? buildAnalyticsEvent(when: when) {
                // Send to your Platform
                // Analytics.sendEvent(evt.name, evt.parameters)
            }
            return try next(when)
        } catch {
            // Send to your crash reporting Platform
            // CrashReporting.send(error)
            throw error
        }
    }
}

// Modify your ViewModel Call a middleware on every event
enum When: AnalyticsWhen {
    case userTypesNewEmail(userEmail: String)
    case userTapsChangeEmailButton
}

final class MyViewModel: ObservableObject {
    @Published var email: String = ""
    func handle(_ when: When) {
        AnalyticsProvider.analyticsAndCrashReportMiddleware(when: when) { when in
            switch when {
            case .userTypesNewEmail(let newEmail):
                self.email = newEmail
            case .userTapsChangeEmailButton:
                print("Saving data: \(email)")
            }
        }
    }
}
```

##### **Example 3.2: Building a simple test framework**

Another example: adding a middleware to send crashes to your crash reporting plaftorm. This example shows
you how middlewares should be declared as controlling next middleware call

```swift
// TODO
```

### **Handling Effects Separately**

Effects (or side effects), are external interactions like API requests, database updates, or system notifications.
In this event/`when` based approach, these asynchronous effects should be modeled separately as 2 different synchronous 
events to maintain clean business logic: One for effect triggering and another one to handle effect completion

Instead of directly managing side effects inside the when handler, you should model their completion or failure as
additional cases in the `When` enum. For instance:

```swift
// Define additional when cases for handling effect completions or failures
enum When {
    case systemShowsUserProfileView
    case networkFinishesLoadingProfile(Result<[String: String], Error>)
    case userTypesNewEmail(userEmail: String)
    case userTapsChangeEmailButton
    case networkFinishesSavingProfile(Result<[String: String], Error>)
}
```

#### **Step 4: Triggering `When`s from Effects**

When an effect completes, trigger a new `when`. For example, after loading data from a network request, an `when` is triggered to handle the response.

```swift
// Build an effects handler to handle async tasks from outside
enum Effect {
    case performGetProfileNetworkRequest
    case performSaveProfileNetworkRequest(email: String)
}
typealias EffectHandler = (Effect) async throws -> When
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
        // Simulate network delay and response
        try await Task.sleep(nanoseconds: 1_000_000_000)
        return .networkFinishesSavingProfile(
            .success([
                "name": "John Doe", "email": email
            ])
        )
    }
}

// Implement the business logic
final class MyViewModel: ObservableObject {
    @Published var loading: Bool = false
    @Published var email: String = ""
    // Declared as var so it can be injected. Inject a throwing handler for testing, so test don't trigger effects
    let effectHandler: EffectHandler = fakeEffectHandler 
    func handle(_ when: When) {
        switch when {
        case .systemShowsUserProfileView:
            loading = true
            Task { [weak self, effectHandler] in
                let profileResponseWhen = try await effectHandler(.performGetProfileNetworkRequest)
                self?.handle(profileResponseWhen)
            }
        case .networkFinishesLoadingProfile(let profile),
                .networkFinishesSavingProfile(let profile):
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
                self?.handle(profileResponseWhen)
            }
        }
    }
}
```

### **Benefits of This Approach**

1. **Structured Flow of `When`s**: The single entry point structure ensures a well-defined flow of `when`s,
 making the code easier to follow, log, test and maintain.

2. **Powerful Testing Capabilities**: Test your business logic in isolation by injecting mock `when`s and 
asserting the expected outcomes without dealing with real side effects. Effects are controlledly completed by
new modelled `when` completion events in the test code.

3. **Clean Separation of Business Logic and Side Effects**: By modeling side effects separately and handling their results 
as `when`s, your business logic remains pure and testable.
   
### **Conclusion**

Implementing a single entry point for `when` handling in Swift iOS applications provides a powerful way to manage business
logic. It centralizes control, enhances testability, and separates the complexities of side effects, making your code cleaner,
more maintainable, and easier to debug. 

This approach not only simplifies how inputs and outputs are handled but also lays the groundwork for advanced features 
like automated testing, comprehensive logging, and structured error reporting. By embracing this architecture, you set 
 strong foundation for building robust, scalable, and reliable iOS applications.
