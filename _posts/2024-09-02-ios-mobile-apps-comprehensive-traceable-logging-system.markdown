---
layout: post
title:  "iOS Mobile Apps with Comprehensive and Traceable Logging System"
date:   2024-09-02 21:31:41 +0200
categories: swiftui
---

In the development of iOS mobile apps, one of the key aspects of ensuring robustness, maintainability, and scalability is the implementation of a well-thought-out logging system. A comprehensive logging solution not only aids in debugging but also enables traceability and reporting. With a modular design, developers can extend the logging system to integrate with third-party services, record user actions, and gather insights into app behavior. This article will guide you through building a logging system in Swift that achieves these goals.

### **Why You Need a Comprehensive Logging System**

While simple print statements (`print()`) are helpful during early stages of development, they don’t scale well for larger apps. Comprehensive logging offers several key benefits:

- **Debugging**: Log messages help identify issues in development, testing, and even production.
- **Traceability**: By logging critical events, you can trace the app's behavior, helping to debug complex issues.
- **Reporting**: Logs can be used to track key metrics such as user activity, performance bottlenecks, and errors.
- **Analytics**: A well-designed logging system can generate insights that help improve user experience and performance.

## **Building a Modular Logging System in Swift**

A modular logging system provides flexibility. It allows different modules to plug into the core logger 
for different functionalities like network reporting, analytics, or error handling. Let’s start by 
building a basic logger and then scale it into a modular system.

### **Step 1: The Core Logger**

Start by creating a `Logger` class that acts as the foundation of your logging system. The logger will handle basic logging operations and provide methods to log messages with different levels (info, debug, error, etc.).

```swift
enum LogLevel {
    case info
    case debug
    case warning
    case error
}

class Logger {
    static let shared = Logger()
    private init() {}
    func log(_ message: String, level: LogLevel = .info, file: String = #file, line: Int = #line) {
        let fileName = (file as NSString).lastPathComponent
        print("\(Datºe())\t\(level.emoji)[\(level)]\t\(fileName):\(line)\t\(message)")
    }
}
```

This simple `Logger` class has:
- **Log Levels**: `info`, `debug`, `warning`, and `error` to classify the severity of logs.
- **File and Line Information**: Captures the file name and line number for easier traceability.

**Usage Example:**

```swift
Logger.shared.log("App started successfully", level: .info)
Logger.shared.log("Fetching user data", level: .debug)
Logger.shared.log("Invalid response received", level: .error)
```

See the result in the debugger

![Image]({{ site.baseurl }}/assets/images/ios-mobile-apps-comprehensive-traceable-logging-system-print.png)


### **Step 2: Modular Logging**

A modular logging system allows you to extend logging to different subsystems, such as 
network logging, error reporting, or even analytics reporting. This can be achieved by 
using injection.

First, define a `LogHandler` injectable struct that each subsystem can implement.

```swift
struct LogMessage {
    let message: String
    let level: LogLevel
    let file: String
    let line: Int
}

struct LogHandler {
    let log: (LogMessage) -> Void
}
```

Then, modify the `Logger` class to allow multiple log handlers:

```swift
class Logger {
    static let shared = Logger()
    private var handlers: [LogHandler] = []
    func addHandler(_ handler: LogHandler) {
        handlers.append(handler)
    }
    func log(
        _ message: String,
        level: LogLevel = .info,
        file: String = #file,
        line: Int = #line
    ) {
        let logMessage = LogMessage(
            message: message,
            level: level,
            file: (file as NSString).lastPathComponent,
            line: line
        )
        // Pass the log message to all handlers
        for handler in handlers {
            handler.log(logMessage)
        }
    }
}
```

Now, the logger can handle multiple logging destinations by adding handlers dynamically.

### **Step 3: Adding Custom Log Handlers**

You can extend the logger by adding custom log handlers, such as one for network logging or analytics.

#### **Basic `print` Log Handler**

Our first log handler prints to a gdb logs to get the same result as in step 1.
 Here's an example implementation:

```swift
static var dummyPrintLogger = LogHandler { logMessage in
    let fileName = (logMessage.file as NSString).lastPathComponent
    print("\(Date())\t[\(logMessage.level)]\t\(fileName):\(logMessage.line)\t\(logMessage.message)")
}
```

**Usage Example:**

```swift
let logger = Logger()
logger.addHandler(dummyPrintLogger)
logger.log("App started successfully", level: .info)
logger.log("Fetching user data", level: .debug)
logger.log("Invalid response received", level: .error)
```

#### **OS Log Handler**

Apple provides a log tool, you may be already using it. In fact, we all should use it,
it provides many features, specially if abstraction is already in place.

```swift
import OSLog

extension LogLevel {
    // Map our levels to os_log types
    var structuredType: OSLogType {
        switch self {
        case .info: return .info
        case .debug: return .debug
        case .warning: return .error
        case .error: return .fault
        }
    }
}

static var structuredLogger: ComprehensiveModularLog.LogHandler {
    typealias Logger = os.Logger
    let structuredLogger = Logger(subsystem: "com.myorg.myapp", category: "general")
    return LogHandler { logMessage in
        _ = (logMessage.file as NSString).lastPathComponent
        structuredLogger.log(
            level: logMessage.level.structuredType,
            "\(logMessage.message)"
        )
    }
}
```

**Usage Example:**

```swift
let logger = Logger()
logger.addHandler(dummyPrintLogger)
logger.log("App started successfully", level: .info)
logger.log("Fetching user data", level: .debug)
logger.log("Invalid response received", level: .error)
```

See the result in the debugger:

![Image]({{ site.baseurl }}/assets/images/ios-mobile-apps-comprehensive-traceable-logging-system-structured.png)

Some of the features provided by Apple's system are visible in the debugger logs: timestamps, filters, categories, etc...

![Image]({{ site.baseurl }}/assets/images/ios-mobile-apps-comprehensive-traceable-logging-system-structured-expanded.png)


### **Other Log Handlers**

You can also log events for analytics purposes, connect to a network logging system,,,, 
Errors could be posted to error logging system for example to track any unexpected situation
happening on real world scenarios. Build whatever you want on top of the modular logging.

### **Use an existing solution**

As usual, someone did this before us. Apple for example. Take a look at this project for inspiration:

https://github.com/apple/swift-log

## **Conclusion**

Building a modular logging system in Swift allows for flexibility and scalability in your iOS apps.
 By creating a system that supports multiple log handlers, you enable logging for different purposes 
 such as debugging, traceability, network reporting, and (maybe) analytics tracking. As your app grows,
 so too can your logging system, adapting to new modules and external services without needing to 
 rework the core system.

In summary, a well-architected logging system:
- Improves debugging capabilities.
- Increases traceability of issues.
- Facilitates reporting.

With this foundation, your iOS app will be much easier to debug, monitor, and improve over time,
 leading to a more maintainable and reliable codebase.

**Stay tuned for more Swift tips and tricks!**