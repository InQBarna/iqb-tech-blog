---
layout: post
title:  "Swift iOS App Single Entry Point for Action/When Events (Part 1)"
date:   2024-09-05 21:31:41 +0200
categories: swift action functional
---

Designing the business logic of a Swift iOS application around a single entry point can greatly simplify how actions are managed. This approach helps structure code for better testing, logging, reporting, and overall maintainability. By modeling side effects separately, you ensure a clean separation of concerns, enhancing the predictability and scalability of your app’s architecture.

## **Why Use a Single Entry Point for Actions?**

1. **Centralized Action Handling**: By funneling all app events through a single entry point, you gain a centralized place to manage business logic, making the app easier to understand and extend.

2. **Enhanced Testability**: A single entry point allows you to send mock events and verify the resulting state, leading to robust and isolated testing scenarios.

3. **Centralized Logging and Analytics**: Centralizing event handling means you can easily log every event, enhancing your analytics, crash reporting, and other global capabilities.

4. **Separation of Concerns**: This approach clearly separates business logic from side effects (e.g., network requests, database writes), which can be modeled as independent entities that trigger further actions.

---

## **Defining the Single Entry Point Using an Enum**

One way to implement this architecture is by defining a `When` enum that encapsulates all possible events in the app (or in your current app scope). Each case of the enum can have associated values to pass necessary data, allowing the business logic handler to react accordingly.

### **Step 1: Define the `When` Enum**

Create a `When` enum that represents all possible events in your application or scope. Each case can have associated values representing the data required for that specific `when`.

```swift
// Define the When enum with associated values
enum When: Codable {
    case userTypesNewEmail(userEmail: String)
    case userTapsChangeEmailButton
}
```

### **Step 2: Define a Single Entry Point to Handle Whens**

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

### **Step 3: Adding Middleware**

Middleware can be used in some single entry point architectures to receive, transform, and forward `when` events. This makes it easy to implement features such as logging, reporting, and testing at a centralized point.

#### **Example Step 3.1: Analytics and Crash Reporting Middleware**

Centralizing analytics and crash reporting helps improve app quality and user experience. By capturing real-time errors and events, you can quickly diagnose and fix crashes.

**Step 3.1.1**: Create the necessary code to capture a `when` event and send its associated parameters to your analytics platform.

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
            throw "When \(when) is unexpectedly encoded as json " +
                String(describing: String(data: encoded, encoding: .utf8))
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
                // Send to your analytics platform
                // Analytics.sendEvent(evt.name, evt.parameters)
            }
            return try next(when)
        } catch {
            // Send to your crash reporting platform
            // CrashReporting.send(error)
            throw error
        }
    }
}
```

**Step 3.1.2**: Modify your ViewModel to adopt the `AnalyticsWhen` protocol and call the middleware in the `when` handler.

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
        // Any thrown exception in business logic will be reported
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

#### **Example Step 3.2: Building a Simple Test Framework**

Testing is critical for ensuring quality and reliability. Here's an example of a small test framework integrated with XCTest for enhanced test logging and detection of unwanted retain cycles.

**Step 3.2.1**: Create a helper class to integrate testing with XCTest and detect retain cycles.

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

**Step 3.2.2**: Modify your ViewModel to adapt to the testing framework.

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

**Step 3.2.3**: Use the provided framework in your tests.

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

---

Ouch... this article is packed with valuable insights, but to ensure a clear and focused discussion, we’ve decided to split it into two parts, Effects will be added to the second part [Swift iOS App Single Entry Point for Action/When Events - Effects]({{ site.baseurl }}/swift/action/functional/2024/09/06/swift-ios-apps-single-entry-point-for-action-when-events-effects.html)

---

### **Benefits of This Approach**

1. **Structured Flow of `When`s**: The single entry point structure ensures a well-defined flow of `when`s, making the code easier to follow, log, test, and maintain.

2. **Powerful Testing Capabilities**: Test your business logic in isolation by injecting mock `when`s and asserting the expected outcomes.

3. **Clean Declaration of Acceptance Criteria**: By modeling `state` and `when` separately, the expectations from the code are clear for everyone involved.

### **Cons of Using a Single Entry Point for Actions**

1. **Scaling complexity**: While a single entry point can leverage many features, it needs an important review when the app grows. Though scaling this approach can be done, the scaling architecture needs to be well designed and planned.

2. **Risk of Over-centralization**: If too much logic is placed in the single entry point, it can turn into a "God object," complicating maintenance.

---

## **Conclusion**

Implementing a single entry point for `when` handling in Swift iOS applications provides a powerful way to manage business logic. It centralizes control, enhances testability, and makes your code cleaner, more maintainable, and easier to debug.

This approach simplifies how inputs and outputs are handled while laying the groundwork for advanced features like automated testing, comprehensive logging, and structured error reporting. Embracing this architecture builds a strong foundation for scalable, reliable iOS applications.