---
layout: post
title:  "Automated Testing for Mobile App Developers: Trade-offs and Best Practices"
date:   2024-09-27 21:31:41 +0200
categories: swift automated testing
---

![Image]({{ site.baseurl }}/assets/images/automated-testing-for-mobile-apps-tradeoffs-best-practices.png)

Automated testing is crucial in delivering high-quality mobile apps, but it comes with trade-offs, especially regarding external dependencies. In this article, we'll explore the journey from unit tests to end-to-end tests, highlighting the impact of dependencies at each stage. We’ll also include examples of integration tests—both with and without dependencies—so you can see how to handle these scenarios effectively.

## The Importance and Challenge of Managing External Dependencies

A key aspect of automated testing is how to handle external dependencies. These dependencies might include network services, third-party APIs, or device-specific functionality that is often beyond your control. When these dependencies impact your tests, they can make your tests less reliable, harder to maintain, and slower to run.

We've talked about external dependencies at [Injection of External/Environmental Dependencies and Boundaries in Swift iOS Development]({{ site.baseurl }}swift/dependency/injection/2024/09/04/injection-of-external-environmental-dependencies-and-boundaries-in-swift.html). Let’s recap some key points from the previous post:

### Levels of Automated Testing

Automated tests can be categorized into three main types: unit tests, integration tests, and end-to-end (E2E) tests. Each of these has different trade-offs in terms of reliability, speed, and complexity, especially when dealing with external dependencies.

## 1. Unit Testing: Fast, Isolated, and Simple

Unit tests focus on testing individual components or functions in isolation. They don't rely on external dependencies, making them fast, reliable, and easy to maintain. Let’s look at an example using Swift’s XCTest.

### Unit Test Example: Viewing a User Profile

```swift
import XCTest

struct UserService {
    let fetchName: () async throws -> String
}
class ProfileViewModel {
    let userService: UserService
    var loading: Bool = true
    var userName: String?
    func fetchProfile() { /* ... */ }
}
    
final class AutomatedTestingMobileAppsTests: XCTestCase {
    func testFetchUserProfileUnit() {
        let sut = ProfileViewModel(
            userService: UserService(
                fetchName: {
                    try await Task.sleep(nanoseconds: 200_000_000)
                    return "John Doe"
                }
            )
        )
        sut.fetchProfile()
        XCTAssertEqual(sut.loading, true)
        sleep(2) // Naive and flacky await, for the example simplicity
        XCTAssertEqual(sut.userName, "John Doe")
        XCTAssertEqual(sut.loading, false)
    }
}
```

In this example, the test is entirely self-contained. It checks that the user's email is displayed correctly in the profile without relying on network requests or any other external data sources, that have been abstracted an injected by UserService.

### Trade-off in Unit Testing

- **Pros**: Fast and reliable, with no external dependencies. Best for testing business logic.
- **Cons**: Limited in scope as it doesn’t test how different components interact with each other.

## 2. Integration Testing: Connecting the Dots

Integration tests check how different parts of the app work together. They often involve more complex scenarios than unit tests and might or might not involve external dependencies.

### Integration Test Example Without Dependencies

This example connects two components fully under developer control: the `UserProfileViewModel` and an `UserService` object.

```swift
class APIClient {
    let token: String
    let fetchJSON: (URLRequest) async throws -> Data
}
extension UserService {
    init(apiClient: APIClient) {
        struct ProfileJSON: Decodable {
            let name: String
        }
        self.init {
            var request = URLRequest(url: URL(string: "http://example.com")!)
            request.setValue("bearer \(apiClient.token)", forHTTPHeaderField: "Authorization")
            let jsonData = try await apiClient.fetchJSON(request)
            return try JSONDecoder().decode(ProfileJSON.self, from: jsonData).name
        }
    }
}

extension AutomatedTestingMobileAppsTests {
    func testFetchUserProfileIntegrationNoDependencies() {
        let sut1 = UserService(
            apiClient: APIClient(token: "myToken") { _ in
                """
                { "name": "John Doe" }
                """
                .data(using: .utf8)!
            }
        )
        let sut2 = ProfileViewModel(userService: sut1)
        sut2.fetchProfile()
        XCTAssertEqual(sut2.loading, true)
        sleep(2) // Naive and flacky await, for the example simplicity
        XCTAssertEqual(sut2.userName, "John Doe")
        XCTAssertEqual(sut2.loading, false)
    }
}
```

This test ensures the interaction between the `UserProfileViewModel` and `UserService` happens as expected, without involving network calls or third-party APIs, which are abstracted behind `APIClient`.

### Trade-offs in Integration Testing Without dependencies

- **Pros**: Validates that different components work together correctly.
- **Cons**: More maintenance effort than unit, since any change on affected modules need a change in the test.

### Integration Test Example With Dependencies

Now, let's consider an integration test that relies on a network request.

```swift
extension APIClient {
    static func buildProductionAPIClient(token: String) -> APIClient {
        APIClient(token: token) { request in
            try await URLSession.shared.data(for: request).0
        }
    }
}

extension AutomatedTestingMobileAppsTests {
    func testFetchUserProfileIntegration() {
        // Third party library to add a network stub
        stub({ _ in true}) { _ in
            .init(jsonObject: ["name": "John Doe"], 
                  statusCode: 200, 
                  headers: ["content-type": "application/json"])
        }
        let sut1 = UserService(
            // Real apiclient making network requests
            apiClient: APIClient.buildProductionAPIClient(token: "myToken")
        )
        let sut2 = ProfileViewModel(userService: sut1)
        sut2.fetchProfile()
        XCTAssertEqual(sut2.loading, true)
        sleep(2) // Naive and flacky await, for the example simplicity
        XCTAssertEqual(sut2.userName, "John Doe")
        XCTAssertEqual(sut2.loading, false)
    }
}
```

Here, the test includes a network service, making it more prone to changes in the external system's behavior or availability. Using a network stubs library for example helps mitigate some of these issues, but it is still affected by dependencies that may break or out of the scope of the test.

### Trade-offs in Integration Testing

- **Pros**: Validates that different components work together correctly.
- **Cons**: Tests with dependencies like network services can be slower and more brittle.

## 3. End-to-End Testing: Complete User Experience

End-to-end tests validate the entire app's functionality by simulating real user interactions. These tests are the most comprehensive but are also the harder to maintain and the most affected by external dependencies.

### End-to-End Test Example Using XCTest and XCUI Test

```swift
import XCTest

final class UserProfileUITests: XCTestCase {
    func testUserProfileViewing() {
        let app = XCUIApplication()
        app.launch()

        app.buttons["View Profile"].tap()

        let emailLabel = app.staticTexts["EmailLabel"]
        XCTAssertTrue(emailLabel.exists, "Email label should be visible on the profile screen")
        XCTAssertEqual(emailLabel.label, "john.doe@example.com", "Email should be displayed correctly in the UI")
    }
}
```

This E2E test verifies that the user can successfully view their profile in the app. Since it involves the full app stack and potentially real network interactions, it can be more susceptible to flaky tests due to network instability or service downtimes. Furthermore, before starting this test, we need to get sure the account of the user in test is available in our system, so usually a pre-production environment with mocks or aumatically filled-in information is necessary.

### Trade-offs in End-to-End Testing

- **Pros**: Provides the highest level of confidence by testing the app's behavior as a whole.
- **Cons**: Slowest and most brittle due to dependencies on network, UI state, and other systems outside the developer's control: environment accounts, network reachability, etc.... Hard to maintain the test due to changes in the environment: device language or time zone, etc...

## Summary


| **Pros / Cons** | **Unit**  | **Integ.** | **E2E** |
| Maintenance effort | ✅ | 🟡 | ❌ |
| Autonomy (managing dependencies and/or third parties) | ✅ |  ✅️ | ❌ |
| Flackyness | ✅ |  ✅️ | ❌ |
| Execution speed | ✅ |  ✅️ | ❌ |
| Mapping test to real use case or Acceptance Criteria | 🟡 | ✅ | ✅ |
| Deploy confidence | ❌ | 🟡 | ✅ |


## A Working Approach: Focusing on Integration Tests for Maximum Control

At Inqbarna, we rely heavily on integration tests for components within our control. These tests give us the best balance between maintenance effort and confidence in our app's functionality. We still build a test pyramid with a few E2E tests, some Integration tests and many unit, buy we emphasize and enforce the importance of integration tests **relying on mocked dependencies**.

| **Testing target** | **Unit**  | **Integration** | **E2E** |
|-------------------------------------------------|-------------|-------------------|------------|
| Corner Cases                                | ✅✅✅  | ✅           | ❌    |
| Main Features                               | ✅  | ✅✅✅        | ❌    |
| Key Eventual Features (purchase, invite, register) | ✅  | ✅✅✅        | ✅     |

BTW: ✅✅✅ stands for `Many`, ✅ stands for `Some` and ❌ for `None`

### Integration Test Strategy

1. **Control What You Can**: For components developed in-house, use integration tests without external dependencies to ensure they interact as expected.
2. **Mock External Services**: For features relying on third-party services, mock those dependencies to limit flakiness and focus on how the app handles data.
3. **Declare Tests As Use Cases Or Acceptance Criteria**: Tests and source code can be named and declared using human expressions near to Acceptance Criteria sentences to make them understandable and easy to maintain. Ubiquotous language and Acceptance As Code are nice trends to focus on when writing your tests.

### Example: Integration Test with Mocking Network Dependency

In a previous article [Swift iOS App Single Entry Point for Action/When Events - Effects]({{ site.baseurl }}/swift/action/functional/2024/09/06/swift-ios-apps-single-entry-point-for-action-when-events-effects.html) we built a small testing framework that follows the above mentioned strategy, please see the following example with uses a pretty similar strategy:

```swift
protocol ProfileViewModelAcceptance {
    associatedtype SUT
    var sut: SUT { get }
    static func GIVENAnAppLaunch() -> Self
    static func GIVENAnAppLaunchWithNoConnection() -> Self
    func WHENUserNavigatesToProfile() -> Self
    func WHENNetworkFinishesWithResponse(_: Data) async -> Self
}

extension AutomatedTestingMobileAppsTests {
    func testFetchUserProfileIntegrationAcceptanceAsCode() {
        ProfileViewModelTester.GIVENAnAppLaunch()
            .WHENUserNavigatesToProfile()
            .THEN(\.loading, is: true)
            .WHENNetworkFinishesWithResponse("""
                { "name": "John Doe" }
                """.data(using: .utf8)!
            )
            .THEN(\.loading, is: false)
            .THEN(\.userName, is: "John Doe")
    }
}
```

The example enables mocking service responses without the need to abstract them, since the ProfileViewModelTester already provides asynchronous control abstraction and network responses can be mocked declaratively as in WHENNetworkFinishesWithResponse. This network abstraction provides control over the data returned and helps simulate various scenarios without depending on real-world network behavior. AND the test declaration is human friendly and easy to understand.

The real implementation has been removed from the article for the sake of simplicity and the length of the article. However, it's been real-world tested and tests passed.

## Conclusion

Automated testing for mobile apps involves balancing trade-offs between speed, reliability, and complexity. Unit tests provide the fastest feedback with minimal dependencies, while integration and end-to-end tests offer a more holistic view but at a higher cost of maintenance.

By focusing on integration tests where you have the most control and using mocks for external services, you can achieve a solid testing strategy that provides high value with a relatively lower maintenance effort.
