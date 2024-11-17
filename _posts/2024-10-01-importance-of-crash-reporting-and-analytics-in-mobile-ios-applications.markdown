---
layout: post
title:  "The Importance of Crash Reporting and Analytics in iOS Applications"
date:   2024-10-01 21:31:41 +0200
categories: swift analytics crash reporting
---

![Image]({{ site.baseurl }}/assets/images/importance-of-crash-reporting-and-analytics-in-mobile-ios-applications.png)

When building a mobile application, understanding user behavior and identifying unexpected crashes are essential for improving the user experience and ensuring product quality. Crash reporting and analytics tools allow developers to track and diagnose issues that occur in the wild, enabling faster bug resolution and better decision-making based on real user data.

In this article, we'll focus on how to integrate crash reporting and analytics into your iOS application while keeping your code flexible and maintainable. We'll also discuss best practices for abstracting third-party analytics libraries to maintain control over your codebase.

## 1. Why Crash Reporting and Analytics Matter

Crash reporting and analytics are critical for gaining insights into how your app performs in production. These tools help developers:

- **Identify bugs** that only occur under specific user conditions or environments.
- **Track user interactions** to understand how users are engaging with your app.
- **Improve app performance** by analyzing real-world usage patterns.

## 2. Abstracting Analytics and Crash Reporting Code

A common mistake in mobile development is tightly coupling your code to a third-party analytics or crash reporting library. This introduces dependencies that can make it difficult to switch tools in the future and can also make testing more challenging.

To avoid these problems, it's important to create an abstraction layer that sits between your app and the third-party service. This layer provides flexibility, allowing you to swap out the underlying tool without affecting the rest of your code. It also simplifies testing by letting you mock analytics events in your unit and integration tests.

Here’s an example of how you can abstract your analytics code using a value structure with closure-based member variables:

```swift
struct Analytics {
    
    enum Value: Equatable {
        case int(Int)
        case string(String)
        case bool(Bool)
    }

    enum Param: Equatable {
        case userID(String)
        case event(String, parameters: [String: Value]? = nil)
        case error(EquatableError)
    }

    enum Dimension: Equatable {
        case global(String, Value?)
        case clearGlobals
        case user(String, String?)
        case screen(name: String, class: String? = nil)
    }
    
    let send: (Param) -> Void
    let enable: (Bool) -> Void
    let set: (Dimension) -> Void
}
```

### Explanation:
- **Param** represents events, errors, or user-specific data being tracked.
- **Dimension** captures contextual information like user details, screen info, or global data. These dimensions are added to subsequent events.
- By using closures (`send`, `enable`, and `set`), you allow the analytics logic to be injected and tested independently of any third-party services.

This design provides the flexibility to mock analytics and test your app’s behavior without needing to interact with the actual third-party service.

See a test example:

```swift
func testLoginAnalytics() {
    var sent: [Analytics.Param] = []
    var dimensions: [Analytics.Dimension] = []
    let sut = LoginViewModel(
        analytics: Analytics(
            send: { sent.append($0) },
            enable: { _ in XCTFail("No expected analytics enabling in this test") },
            set: { dimensions.append($0) }
        )
    )
    sut.when(LoginViewModel.When.didAppear)
    XCTAssertEqual(sent, [.event("enteringLogin")])
    XCTAssertEqual(dimensions, [.screen(name: "login")])
    sent.removeAll()
    dimensions.removeAll()
    sut.when(LoginViewModel.When.didLoginSuccessfully(
        LoginViewModel.Profile(
            userName: "userName",
            userId: "00001"
        )
    ))
    XCTAssertEqual(sent, [.userID("00001"), .event("login")])
}
```

## 3. Handle Errors Gracefully

An essential part of crash reporting is understanding what led to a crash. Your error-handling strategy should ensure that key crash data is captured without disrupting the user experience. By centralizing error handling and logging in your app, you can gain insights into unexpected behaviors and system failures. It’s also important to ensure that critical errors are reported as soon as they occur, even if the app is still in a recoverable state.

```swift
struct LoginViewModel {
    let analytics: Analytics
    struct Profile {
        let userName: String
        let userId: String
    }
    enum When {
        case didAppear
        case didLoginSuccessfully(Profile)
    }
    func when(_ when: When) {
        // Enclose your throwing business logic to send non-fatal errors
        do {
            switch when {
            case .didAppear:
                analytics.set(.screen(name: "login"))
                analytics.send(.event("enteringLogin"))
                try someThrowingMethod()
            case .didLoginSuccessfully(let profile):
                analytics.send(.userID(profile.userId))
                analytics.send(.event("login"))
            }
        } catch {
            // Send the error
            //  (we have some equatable helper for testing injection)
            analytics.send(.error(error.toEquatableError()))
        }
    }
}
```

## 4. Log Critical Events Sparingly

While it's important to capture user interactions and error data, logging everything can overwhelm your crash reporting and analytics tools. Overloading your system with logs can lead to performance issues and make it harder to pinpoint the root cause of critical issues.

A best practice is to focus on logging the most important events and interactions, such as:

- **Key user actions** that impact the app's state (e.g., starting a purchase or submitting a form).
- **Critical failures** that might prevent a user from continuing their workflow.

This ensures that your logs are useful and not cluttered with unnecessary data, making it easier to trace the origin of a crash or bug.

Most of third party analytics service provide built-in methods and events for key-interactions, use those events as starting point for your analytics.

## 5. Tracking Events and Contextual Information

In mobile applications, it's often necessary to log both the events a user performs and the context in which those events occur. For instance, you might need to capture the user ID, session data, or screen information when tracking an event.

To make this process smoother, your analytics abstraction should account for both events and dimensions. Here's an example of how you might set user-specific or screen-specific dimensions when logging an event:

```swift
func trackUserProfileViewed(analytics: Analytics, userID: String) {
    analytics.set(.user("userID", userID))
    analytics.set(.screen(name: "UserProfile"))
    analytics.send(.event("UserProfileViewed"))
}
```

This example demonstrates how to easily manage contextual information alongside the event. By separating the concerns of **events** and **dimensions**, you ensure that your analytics layer is flexible and easy to manage.

## Conclusion

Setting up crash reporting and analytics is essential for understanding both the technical and behavioral aspects of your app. By abstracting the analytics layer and following best practices such as structured error handling, logging sparingly, and tracking contextual data, you can maintain a clean, testable, and flexible codebase.

These strategies ensure that your app is not only reliable in production but also easy to adapt as your tools or requirements evolve.