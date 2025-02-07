---
layout: post
title:  "Managing and Maintaining Localized Mobile Applications"
date:   2024-10-15 21:31:41 +0200
categories: swift dependecy injection
---

![Image]({{ site.baseurl }}/assets/images/managing-maintaning-localized-mobile-projects.png)

Localization is an essential part of mobile app development, ensuring that your app resonates with users worldwide. However, managing and maintaining localization requires thoughtful planning, efficient workflows, and automation to reduce manual effort and potential errors.

This article outlines how to develop with localization in mind, manage common team workflows, and integrate essential scripts into your development and CI/CD pipeline.

---

## 1. Developing with Localization in Mind

A well-structured localization process begins with embedding localization into your app’s development from the start. Apple’s **string catalog** format provides a modern, organized approach to managing localization keys.

### Example: Using String Catalogs

The string catalog format (`.xcstrings`) simplifies the process of defining and managing localized strings. Here’s how you can implement it:

#### Defining Strings in a Catalog
```swift
import SwiftUI

struct ContentView: View {
    var body: some View {
        Text("welcome_message") // Automatically resolves from the string catalog
    }
}
````

```json
// Example Strings.xcstrings entry
{
  "sourceLanguage" : "es",
  "strings" : {
    "welcome_message" : {
      "extractionState" : "manual",
      "localizations" : {
        "en" : {
          "stringUnit" : {
            "state" : "translated",
            "value" : "Bienvenido"
          }
        },
        "es" : {
          "stringUnit" : {
            "state" : "translated",
            "value" : "Welcome"
          }
        }
      }
    },
    // ...
}
```

**Advantages of String Catalogs:**
- **Organized Structure**: Groups strings with associated metadata (e.g., comments for translators).
- **Built-in Pluralization**: Handles language-specific plural rules seamlessly.
- **All in one place**: Available languages, translations and plurals are stored in a single file.

---

### 1.1 The Importance of Using a Translation Platform

Using a dedicated platform like **Lokalise**, **Phrase**, or **Crowdin** ensures that your translations are consistent, accessible, and easy to manage. These platforms provide features like:
- **Centralized Storage**: One source of truth for all translations for all platforms involved in the product.
- **Synchronization** between source code and localizations. Adding new localized content to the source code automatically adds them to the string catalog.
- **Automation**: Direct integration with your project for pulling, pushing, and linting translations.

For developers, it's strongly recommended to choose a platform with an api. So pulling and pushing new transalations can be automated.

---

## 2. Typical Team Workflows and Their Management

Localization workflows often involve multiple stakeholders, including product teams, developers, and translators. Here’s how to manage these workflows effectively:

---

### 2.1 Project Specification: Defining Translations and Keys

During the project specification phase:
- **Define all translatable strings** while designing the user interface and experience.
- **Define keys/ids for every translation**, although Apple's examples work with the base localization as the id/key of the localied content, most of the platforms work better with keys/ids. Using keys/ids instead of base localization strings works better with multi-platform projects. Try using a consistent key/id naming such as `{screen}_{element}_{action}`.
- **Centralize translations** in the localization platform to ensure consistency across all stakeholders.
- Use a **pull script** to keep the local project in sync with the platform.

#### Example Pull Script
```swift
// PullLocalization.swift
func downloadLocalizableFile(
    localizableUrl: String,
    localizablePath: String
) throws {
    print("Downloading Localizations from \(localizableUrl) to \(localizablePath)")
    guard let url = URL(string: localizableUrl) else {
        print("Invalid URL")
        return
    }
    let data = try Data(contentsOf: url)
    try data.write(to: URL(fileURLWithPath: localizablePath))
    print("Downloaded Localizations from \(localizableUrl), saved to \(localizablePath)")
}

// Main process
try downloadLocalizableFile(
    localizableUrl: "https://localise.biz/api/export/all.xcstrings?key=API_KEY",
    localizablePath: "Resources/Localizations/Localizable.xcstrings"
)
```

Usage

```bash
swift scripts/PullLocalization.swift .
```

---

### 2.2 New Scenarios from Developers During Development

Developers often encounter edge cases requiring additional localized strings, such as error messages or new UI elements. These keys should be:
- **Use previous naming convention** to keep keys/ids consistent.
- **Add to the local string catalog**. This happens automatically when project is compiled.
- **Pushed to the translation platform** using a **push script** to keep the platform up to date.

#### Example Push Script

See how we explode the `xcodebuild -exportLocalizations` command. It searches for all localization source code in the project and creates a string catalog with all keys used in the source code.

```bash
#! /bin/sh

# Export keys in source code to string catalog
xcodebuild -exportLocalizations -workspace WORKSPACE.xcworkspace -localizationPath Resources/Localizations/exported -exportLanguage es

# Upload string catalog to platform with a "crear only new translations, ignore existing"
curl --data-binary @Resources/Localizations/exported/es.xcloc/Localized\ Contents/es.xliff 'https://localise.biz/api/import/xlf?ignore-existing=true&locale=es&key=API_KEY'

# Remove exported content
rm -Rf Resources/Localizations/exported
```

---

### 2.3 New Features or Localizations from Product Stakeholders

When product stakeholders introduce new features or languages:
1. Use the **pull script** to fetch updated translations.
2. Run a **lint script** to detect potential issues such as:
   - Missing translations for newly added languages.
   - Unused keys that are no longer in the source code.
   - Keys referenced in the source code but missing in the remote platform.

#### Example Lint Script

Please see an excerpt of the scripts we use to insert in the projects for linting

```swift
// LocalizationReport.swift
struct TranslationError {
    let languageCode: String
    let key: String
    enum Type {
       case emptyTranslation
       case contentEqualToKey
    }
    let description: String
}

class XliffValidator {
    func validateXliffFiles(at paths: [String]) -> [TranslationError] {
        // Search all keys and check which ones have errors: missing translations or
        //  translated content is equal to key
    }
    func printReportToSTDOut(from errors: [TranslationError]) {
        // Print a report of found errors to stdout
    }
    /* ...  we use swift xml parser to read xliff files  ... */
}

func getXliffFileNames() throws -> [String] {
    let folderPath = "Resources/Localizations/exported/"
    let xclocDirectories = (try FileManager.default.contentsOfDirectory(atPath: folderPath))
        .filter { $0.hasSuffix(".xcloc") }
    return  xclocDirectories.map { xclocDirectory in
        let xclocPath = folderPath + xclocDirectory + "/Localized Contents/"
        let xliffFiles = try FileManager.default.contentsOfDirectory(atPath: xclocPath)
        return xliffFiles.filter { $0.hasSuffix(".xliff") }
            .map { xclocPath + $0 }
    }
}

func main() throws {
    let validator = XliffValidator()
    let xliffPaths = try getXliffFileNames()
    let errors = validator.validateXliffFiles(at: xliffPaths)
    validator.printReportToSTDOut(from: errors)
    exit(errors.count > 0 = -1 : 0)
}

do {
    main()
} catch {
    print("Linting error: \(error.localizedDescription)")
    exit(-1)
}
```

This script assumes an xliff (xccatalog) file exists with all source code's localized content. So there's a previous step before running it

```bash
#! /bin/sh

# Export keys in source code to string catalog
xcodebuild -exportLocalizations -workspace WORKSPACE.xcworkspace -localizationPath Resources/Localizations/exported -exportLanguage es -exportLanguage ca -exportLanguage en

# Run report / link script
swift scripts/LocalizationReport.swift .

# Remove exported
rm -Rf Resources/Localizations/exported
```

---

## 3. Integrating Pull, Push, and Lint Scripts

To streamline localization management:
1. **Pull Script**: Keeps your local project up to date with the latest translations.
2. **Push Script**: Ensures new keys are added to the translation platform.
3. **Lint Script**: Validates localization consistency by detecting missing or unused keys.

### Automating with CI/CD

Integrate these scripts into your CI/CD pipeline to ensure localization remains consistent throughout development. For example:
- Execute the **lint script** as part of pull request validation build.
- Run a localized content update before a release build to fetch the latest translations. For example **pull_localizations**, **lint_localizations** and the commit the changes to the release branch if lint success for an up-to-date localizations releas.
- ... choose your preferred automated workflow to ensure quality and work agreements in your teams ...

---

## Conclusion

Managing and maintaining localized mobile applications requires a combination of thoughtful planning, effective workflows, and automation. By developing with localization in mind, centralizing translations in a platform, and utilizing scripts to handle pulling, pushing, and linting, you can streamline the process and deliver a polished experience for users worldwide.

Stay tuned for more insights into automating localization and integrating it into your development lifecycle!
