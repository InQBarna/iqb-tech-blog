---
layout: post
title:  "Injection of External/Environmental Dependencies and Boundaries in Swift iOS Development"
date:   2024-09-04 21:31:41 +0200
categories: swift dependency injection
---

![Image]({{ site.baseurl }}/assets/images/injection-of-external-environmental-dependencies-and-boundaries-in-swift.png)

When developing iOS applications with Swift, it’s crucial to understand the boundaries between your business logic and external dependencies, such as system libraries and third-party services. By properly injecting these dependencies, you can enhance your code’s flexibility, testability, and resilience. This article will guide you through the concept of dependency injection, the importance of separating external dependencies, and best practices for setting up boundaries in your Swift code.

### **Why Inject External Dependencies?**

External dependencies include system libraries, environmental factors, and third-party services that your app relies on, such as:

- **System Locale and Date Formatting**: Handling dates, times, and locales based on user settings.
- **Network Responses**: Making API calls and processing responses.
- **File System Access**: Reading from or writing to files.
- **User Defaults or Keychain**: Storing and retrieving user-specific data.

These dependencies are essential but can be unpredictable, especially during testing. Injecting these dependencies rather than hardcoding them into your business logic provides several advantages:

1. **Mocking and Stubbing**: Easily mock or stub dependencies during testing, simulating different scenarios without relying on actual system behavior or network conditions.
2. **Decoupling Business Logic**: By decoupling business logic from third-party libraries and system dependencies, you keep your core application logic clean and focused.
3. **Increased Testability**: Dependencies can be swapped out for mock implementations in unit tests, ensuring tests are isolated and reliable.
4. **Improved Flexibility and Scalability**: Your code becomes more adaptable to changes, such as switching to a different library or adjusting behavior for testing.

### **Identifying Boundaries: Business Logic vs. External Dependencies**

A critical aspect of managing dependencies is identifying the boundaries between your business logic and external dependencies.

- **Business Logic**: The core algorithms, rules, and data manipulations specific to your application. This is the code you write and control.
- **External Dependencies**: Code and libraries that interact with the outside world, such as date formatting, networking, or accessing hardware features. These are often pre-tested and maintained by their providers.

**Boundary Principle**: Set up boundaries at the interface where your business logic meets external dependencies. The goal is to inject dependencies rather than letting them bleed into your business logic.

### **Examples of Dependency Injection and Decoupling**

Let’s explore a few common examples of dependency injection in Swift iOS development.

#### **1. Injecting Date Providers**

Directly using `Date()` and `DateFormatter()` inside your business logic couples your code to system behavior, making testing difficult. Instead, inject a date provider.

**Provider Definition:**

Define a struct to abstract the date functionality:

```swift
struct DateProvider {
    var currentDate: () -> Date
}

// The Providers enum is used as a namespace
enum Providers {}
extension Providers {
    static var defaultDateProvider = DateProvider(
        currentDate: Date.init
    )
}
```

*You may use a protocol for this, but we prefer using value types whenever possible.*

**Using the DateProvider in Business Logic:**

Inject the `DateProvider` into your class instead of directly using `Date()`:

```swift
struct EventScheduler {
    private let dateProvider: DateProvider
    func scheduleEvent() -> String {
        let currentDate = dateProvider.currentDate()
        // Your business logic here using currentDate
        return "Event scheduled at \(currentDate)"
    }
}
```

**Testing with Mocked Date Providers:**

Inject a mock provider during testing to simulate specific date scenarios.

```swift
extension Providers {
    static var mockDateProviderFirstJan1970 = DateProvider(
        currentDate: {
            // Return a fixed date for testing
            return Date(timeIntervalSince1970: 0)
        }
    )
}

// Unit Test Example
class DependenciesAndBoundariesTests: XCTestCase {
    func testInjectionDate() {
        let scheduler = EventScheduler(dateProvider: Providers.mockDateProviderFirstJan1970)
        XCTAssertEqual(scheduler.scheduleEvent(), "Event scheduled at 1970-01-01 00:00:00 +0000")
    }
}
```

#### **2. Injecting Network Clients**

Networking is another area where tightly coupling your code with libraries like `URLSession` can make testing and maintenance difficult. Inject a network client instead.

**Provider Definition:**

Define a provider for networking:

```swift
struct NetworkProvider {
    let fetchData: (URLRequest) async throws -> Data
}
extension Providers {
    static var defaultNetworkProvider = NetworkProvider(
        fetchData: {
            try await URLSession.shared.data(for: $0).0
        }
    )
}
```

Let's make the provider more versatile. We'll ensure these lines are tested as part of our business logic:

```swift
extension NetworkProvider {
    func fetchData<T: Decodable>(from: URLRequest) async throws -> T {
        try await JSONDecoder().decode(T.self, from: fetchData(from))
    }
}
```

**Injecting the Network Client in Business Logic:**

Inject the `NetworkClient` instead of directly using `URLSession`:

```swift
struct DataFetcher {
    private let networkClient: NetworkProvider
    init(networkClient: NetworkProvider) {
        self.networkClient = networkClient
    }
    
    struct DTO: Codable {
        let message: String
    }
    func fetchRemoteData(from url: URLRequest) async throws -> DTO {
        try await networkClient.fetchData(from: url) as DTO
    }
}
```

**Testing with Mocked Network Clients:**

For testing, use a mock implementation of `NetworkClient` to simulate network responses.

```swift
extension DependenciesAndBoundariesTests {
    func testInjectionNetwork() async throws {
        let fetcher = DataFetcher(
            networkClient: NetworkProvider(
                fetchData: { _ in
                    // Return a fixed string for testing
                    return "{\"message\": \"Hello world\"}".data(using: .utf8)!
                }
            )
        )
        let dummyURL = URLRequest(url: URL(string: "about:blank")!)
        let resultString = try await fetcher.fetchRemoteData(from: dummyURL)
        XCTAssertEqual(resultString.message, "Hello world")
    }
}
```

### **Best Practices for Setting Up Dependency Injection Boundaries**

1. **Use Providers to Define Boundaries**: Providers serve as boundaries that separate your business logic from external dependencies, allowing you to inject different implementations based on the context.
   
2. **Inject Dependencies at the Highest Level**: Dependencies should be injected at the topmost level where they are used, such as during the initialization of a view model or a service class.

3. **Minimize Direct Access**: Avoid directly accessing system or third-party APIs within your business logic. Instead, encapsulate them behind providers or service layers.

4. **Design for Testability**: Always design your classes and services with testing in mind. Ask yourself: Can this code be tested in isolation without relying on actual system behavior?

5. **Leverage Dependency Injection Frameworks**: Consider using dependency injection frameworks like Resolver or Swinject to manage the injection and lifecycle of dependencies in your app.

## **Conclusion**

By properly injecting external dependencies and defining clear boundaries between your business logic and third-party libraries, you can create a more maintainable, testable, and flexible codebase. Decoupling these dependencies allows you to focus on building robust business logic while safely leveraging external functionality.