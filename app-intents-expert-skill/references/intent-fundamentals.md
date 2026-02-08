# Intent Fundamentals

Core patterns for building App Intents: the `AppIntent` protocol, parameters, perform methods, results, dialog, and supported modes.

## Creating a basic intent

Every intent is a struct conforming to `AppIntent`:

```swift
import AppIntents

struct OpenFavoritesIntent: AppIntent {
    static let title: LocalizedStringResource = "Open Favorites"

    func perform() async throws -> some IntentResult {
        return .result()
    }
}
```

Requirements:
- `title` must be a **compile-time constant** `LocalizedStringResource` (no computed properties or function calls)
- `perform()` is `async throws` and returns `some IntentResult`

## Parameters

Use `@Parameter` to declare inputs. Parameters can be required (default) or optional:

```swift
struct NavigateIntent: AppIntent {
    static let title: LocalizedStringResource = "Navigate to Section"

    @Parameter(title: "Section", requestValueDialog: "Which section?")
    var section: NavigationOption

    @Parameter(title: "Animated", default: true)
    var animated: Bool

    func perform() async throws -> some IntentResult {
        Navigator.shared.navigate(to: section, animated: animated)
        return .result()
    }
}
```

Supported parameter types:
- Swift primitives: `String`, `Int`, `Double`, `Bool`, `Date`, `URL`
- `AppEnum` — fixed set of values known at compile time
- `AppEntity` — dynamic values resolved at runtime via queries
- `IntentFile` — file inputs
- Arrays of any of the above

### Parameter Summary

A human-readable sentence describing the intent and its parameters. **Required for Spotlight actions on Mac** (iOS 26+):

```swift
static var parameterSummary: some ParameterSummary {
    Summary("Navigate to \(\.$section)")
}
```

Conditional summaries:

```swift
static var parameterSummary: some ParameterSummary {
    When(\.$useCustomDate, .equalTo, true) {
        Summary("Log \(\.$item) on \(\.$date)")
    } otherwise: {
        Summary("Log \(\.$item)")
    }
}
```

## Return types

Compose return types to declare what your intent produces:

```swift
// Returns nothing
func perform() async throws -> some IntentResult

// Returns a value
func perform() async throws -> some ReturnsValue<MyEntity>

// Returns a value with spoken dialog
func perform() async throws -> some ReturnsValue<MyEntity> & ProvidesDialog

// Returns a value with dialog and a view snippet
func perform() async throws -> some ReturnsValue<MyEntity> & ProvidesDialog & ShowsSnippetView

// Shows an interactive snippet intent (iOS 26+)
func perform() async throws -> some ReturnsValue<MyEntity> & ShowsSnippetIntent & ProvidesDialog
```

### Dialog

Dialog is text that Siri can speak aloud:

```swift
return .result(
    value: landmark,
    dialog: IntentDialog(
        full: "The closest landmark is \(landmark.name).",
        supporting: "\(landmark.name) is located in \(landmark.continent)."
    )
)
```

### View snippets

Attach a SwiftUI view to display alongside the result:

```swift
return .result(
    value: landmark,
    dialog: "The closest landmark is \(landmark.name)",
    view: LandmarkCardView(landmark: landmark)
)
```

## Supported modes (foreground control)

Control whether the intent foregrounds the app:

```swift
struct GetStatusIntent: AppIntent {
    static let title: LocalizedStringResource = "Get Status"

    // Never foregrounds
    static let supportedModes: IntentModes = .background

    // Always foregrounds before perform runs
    static let supportedModes: IntentModes = .foreground

    // Both — intent decides at runtime
    static let supportedModes: IntentModes = [.background, .foreground]

    func perform() async throws -> some IntentResult {
        // Check current mode
        if currentMode == .foreground {
            // Navigate in the app
        } else {
            // Return dialog only
        }
        return .result()
    }
}
```

### Dynamic and deferred foreground modes

```swift
// Dynamic: intent decides whether to foreground
static let supportedModes: IntentModes = [.background, .dynamic]

// Deferred: intent will foreground eventually, but not immediately
static let supportedModes: IntentModes = [.background, .deferred]
```

To foreground from a dynamic/deferred intent:

```swift
func perform() async throws -> some IntentResult {
    let canForeground = systemContext.canContinueInForeground

    if canForeground {
        try await continueInForeground(alwaysConfirm: false)
        // Now in foreground — navigate
    } else {
        // Return dialog-only result
    }
    return .result()
}
```

## Dependencies

Inject shared objects into intents using `@Dependency`:

```swift
struct FindLandmarkIntent: AppIntent {
    static let title: LocalizedStringResource = "Find Landmark"

    @Dependency
    var modelData: ModelData

    func perform() async throws -> some ReturnsValue<LandmarkEntity> {
        let landmark = try await modelData.findClosestLandmark()
        return .result(value: landmark)
    }
}
```

Register dependencies early in the app lifecycle:

```swift
@main
struct MyApp: App {
    init() {
        AppDependencyManager.shared.add { ModelData() }
    }
}
```

## Open Intent

Use `OpenIntent` for intents that navigate to content in your app:

```swift
struct OpenLandmarkIntent: OpenIntent {
    static let title: LocalizedStringResource = "Open Landmark"

    @Parameter(title: "Landmark", requestValueDialog: "Which landmark?")
    var target: LandmarkEntity
}
```

`OpenIntent` automatically foregrounds the app — no need to set `supportedModes`. Combine with `TargetContentProvidingIntent` for SwiftUI navigation (see [intent-driven-architecture.md](intent-driven-architecture.md)).

## Undoable Intent (iOS 26+)

Support system undo gestures:

```swift
struct DeleteCollectionIntent: AppIntent, UndoableIntent {
    static let title: LocalizedStringResource = "Delete Collection"

    @Parameter var collection: CollectionEntity

    func perform() async throws -> some IntentResult {
        let data = collection.backup()
        undoManager?.registerUndo(withTarget: store) { store in
            store.restore(data)
        }
        undoManager?.setActionName("Delete \(collection.name)")
        store.delete(collection)
        return .result()
    }
}
```

## Multiple choice (iOS 26+)

Present options for the user to pick from:

```swift
func perform() async throws -> some IntentResult {
    let delete = IntentOption(title: "Delete", style: .destructive)
    let archive = IntentOption(title: "Archive")
    let cancel = IntentOption(title: "Cancel", style: .cancel)

    let choice = try await requestChoice(
        between: [delete, archive, cancel],
        dialog: "What would you like to do?"
    )

    switch choice {
    case delete: store.delete(item)
    case archive: store.archive(item)
    default: break
    }
    return .result()
}
```

## Requesting values at runtime

If a parameter is not provided, request it:

```swift
func perform() async throws -> some IntentResult {
    let count = try await $ticketCount.requestValue("How many tickets?")
    // use count
    return .result()
}
```

## Key rules

- **Titles must be constant.** `static let title` cannot call functions or use computed properties.
- **Perform is the only place for side effects.** Do not mutate state in initializers.
- **Return types are declarative.** The system inspects the return type signature at build time.
- **Register dependencies early.** `AppDependencyManager.shared.add { }` in `App.init()`.
- **Use `@MainActor` on perform** when the intent needs to interact with UI-related code.
