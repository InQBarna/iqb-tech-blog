---
layout: post
title:  "Swift iOS App Single Entry Point for Action/When Events"
date:   2024-09-05 21:31:41 +0200
categories: swift action functional
---

Designing the business logic of a Swift iOS application around a single entry point can greatly simplify 
how actions are managed. This approach helps structure code for better testing, logging, reporting,
 and overall maintainability. By modeling side effects separately, you ensure a clean separation of concerns,
 enhancing the predictability and scalability of your app’s architecture.

### **Why Use a Single Entry Point for Actions?**

1. **Centralized Action/When Handling**: By funneling all app events through a single entry point,
 you gain a centralized place to manage business logic, making the app easier to understand and extend.
  
2. **Enhanced Testability**: A single entry point allows you to send mock events and verify state after 
sending, leading to robust and isolated testing scenarios.

3. **Centralized logging, analytics and more**: Centralizing events means you can easily log every event, enhancing your 
analytics, crash reporting, and other global capabilities

4. **Separation of Concerns**: Clearly separates business logic from side effects (e.g., network requests,
 database writes), which can be modeled as independent entities that trigger further actions.

### **Defining the Single Entry Point Using an Enum**

Our approach to this solution is to define a `When` enum that encapsulates all possible events in the app (or 
in your current app scope). Each case of the enum can have associated values to pass necessary data, allowing 
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
with this approach, like

- Logs and Reporting: local/remote logs, analytics and crash reporting are easy to implement.

- Testing: testing becomes straightforward with the basic steps: send an event, then check the
correct state.

- Reproducibility: recording the received events to reproduce an ongoing error can be implemented
in the event handler.

- And much more... some features are easier to implement in a global middleware than in separate
methods.


#### **Example Step 3.1: Analytics and crash reporting middleware**

Integrating analytics and crash reporting in a mobile app is essential for improving user experience 
and app quality. Crash reporting captures real-time errors, enabling detection, quick diagnosis and 
fixes, which reduces app crashes and enhances stability. Sending analytics and crash reporting is
easy on a centralized place, please see the following example:

**Step 3.1.1**: Create the necessary code to capture a `when`event and send its associated parameters 
to your analytics platform

```swift
struct AnalyticsProvider {

    struct AnalyticsEvent {
        let name: String
        let parameters: [String: Any]
    }

    static func buildAnalyticsEvent(
        when: AnalyticsWhen
    ) throws -> AnalyticsEvent {
        let encoded = try JSONEncoder().encode(when)
        let json = try JSONSerialization.jsonObject(with: encoded, options: [])
        guard let jsonDict = json as? [String: Any],
              let enumCaseName = jsonDict.keys.first else {
            throw "When \(when) is unexpectedly encoded as json "
                + String(describing: String(data: encoded, encoding: .utf8))
        }
        return AnalyticsEvent(
            name: enumCaseName,
            parameters: jsonDict[enumCaseName] as? [String: Any] ?? [:]
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
```

**Step 3.1.2**: Modify your ViewModel to adopt necessary `AnalyticsWhen` protocol and call the 
analytics and crash report middleware in the `when` handler.

```swift
enum When: AnalyticsWhen {
    case userTypesNewEmail(userEmail: String)
    case userTapsChangeEmailButton
}

final class MyViewModel: ObservableObject {
    @Published var email: String = ""
    @Published var savedEmail: String?
    func handle(_ when: When) {
        // Every event will be sent to analytics
        // Every thrown exception in business logic will be reported
        AnalyticsProvider.analyticsAndCrashReportMiddleware(when: when) {
            switch $0 {
            case .userTypesNewEmail(let newEmail):
                self.email = newEmail
            case .userTapsChangeEmailButton:
                self.savedEmail = self.email
            }
        }
    }
}
```

#### **Example Step 3.2: Building a simple test framework**

Adding testing in your app ensures quality, reliability, and clear understanding of
business logic. Next example will define a small testing framework for an app enforcing: 
clear readibility, integration with xctest logging framework and checking unwanted 
retain cycles. More features can be added to the single entry point methods.

**Step 3.2.1**: Create a helper class to be used in your project's tests to enforce
xctest integration and retain cycles detection.

```swift
final class TestRunner<T: SingleEntryPoint> {

    let sut: () -> T
    private var steps: [(T) -> Void] = []
    private init(_ builder: @escaping () -> T) {
        self.sut = builder
    }

    static func GIVEN<TT: SingleEntryPoint>(
        file: StaticString = #filePath,
        line: UInt = #line,
        builder: @escaping () -> TT
    ) -> TestRunner<TT> {
        return TestRunner<TT>(builder)
    }

    func WHEN(
        _ when: T.When,
        file: StaticString = #filePath,
        line: UInt = #line
    ) -> Self {
        self.steps.append({ sut in
            XCTContext.runActivity(named: "WHEN: \(try! buildTestLogEventName(when))") { _ in
                sut.handle(when)
            }
        })
        return self
    }

    func THEN<P: Equatable>(
        file: StaticString = #filePath,
        line: UInt = #line,
        _ keyPath: KeyPath<T, P>,
        is expectedValue: P
    ) -> Self {
        steps.append({ sut in
            XCTContext.runActivity(named: "THEN: \(keyPath.lastPathComponent) is \(expectedValue)") { _ in
                XCTAssertEqual(sut[keyPath: keyPath], expectedValue, file: file, line: line)
            }
        })
        return self
    }

    func runTests() {
        weak var releasableSut: T?
        autoreleasepool {
            let sut: T = XCTContext.runActivity(named: "GIVEN: \(T.self)") { _ in return self.sut() }
            releasableSut = sut
            for step in steps {
                step(sut)
            }
        }
        XCTAssert(releasableSut == nil)
    }
}
```

**Step 3.2.2**: Make small changes to the VM to adapt to the testing framework protocols

```swift
final class MyViewModel: ObservableObject, SingleEntryPoint {
    
    enum When: Codable {
        case userTypesNewEmail(userEmail: String)
        case userTapsChangeEmailButton
    }

    @Published var email: String = ""
    @Published var savedEmail: String?
    func handle(_ when: When) {
        switch when {
        case .userTypesNewEmail(let newEmail):
            self.email = newEmail
        case .userTapsChangeEmailButton:
            self.savedEmail = self.email
        }
    }
}
```

**Step 3.2.3**: Use the providede framework baseline in your tests.

```swift
class SingleEntryPointTests: XCTestCase {
    func testUserStory() {
        TestRunner<MyViewModel>.GIVEN {
            MyViewModel()
        }
        .WHEN(.userTypesNewEmail(userEmail: "john.dohe@example.com"))
        .THEN(\.email, is: "john.dohe@example.com")
        .WHEN(.userTapsChangeEmailButton)
        .THEN(\.savedEmail, is: "john.dohe@example.com")
        .runTests()
    }
}
```

In case you've never needed XCTest loggin integration for xcode or jenkins, please see the result:

![Image]({{ site.baseurl }}/assets/images/swift-ios-apps-single-entry-poing-xctest-integration.png)

### **Handling Effects Separately**

Ouch... this article is packed with valuable insights, but to ensure a clear and focused discussion,
we’ve decided to split it into two parts, Effects will be added to the second part. Stay tuned for it,
where we’ll dive even deeper into the topic!


### **Benefits of This Approach**

1. **Structured Flow of `When`s**: The single entry point structure ensures a well-defined flow of `when`s,
 making the code easier to follow, log, test and maintain.

2. **Powerful Testing Capabilities**: Test your business logic in isolation by injecting mock `when`s and 
asserting the expected outcomes.

3. **Clean Declaration of Acceptance Criterai and scelarios**: By modeling `state` and `when` separatedly,
expectation of the source code is clear to everyone involved.
   
### **Conclusion**

Implementing a single entry point for `when` handling in Swift iOS applications provides a powerful way to 
manage business logic. It centralizes control, enhances testability, making your code cleaner, more maintainable,
and easier to debug. 

This approach not only simplifies how inputs and outputs are handled but also lays the groundwork for advanced 
features  like automated testing, comprehensive logging, and structured error reporting. By embracing this 
architecture, you set strong foundation for building robust, scalable, and reliable iOS applications.
