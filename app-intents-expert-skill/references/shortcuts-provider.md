# App Shortcuts Provider

Creating `AppShortcutsProvider` to make your intents discoverable in Siri, Spotlight, Shortcuts app, Action Button, and Apple Pencil.

## AppShortcutsProvider

Your app should define a single provider containing all your App Shortcuts:

```swift
import AppIntents

struct MyAppShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: NavigateIntent(),
            phrases: [
                "Navigate in \(.applicationName)",
                "Navigate to \(\.$section) in \(.applicationName)"
            ],
            shortTitle: "Navigate",
            systemImageName: "arrowshape.forward"
        )

        AppShortcut(
            intent: FindClosestLandmarkIntent(),
            phrases: [
                "Find closest landmark in \(.applicationName)"
            ],
            shortTitle: "Closest Landmark",
            systemImageName: "location"
        )

        AppShortcut(
            intent: OpenLandmarkIntent(),
            phrases: [
                "Open \(\.$target) in \(.applicationName)",
                "Open landmark in \(.applicationName)"
            ],
            shortTitle: "Open Landmark",
            systemImageName: "mappin"
        )
    }
}
```

## AppShortcut structure

Each `AppShortcut` requires:

| Parameter | Type | Description |
|---|---|---|
| `intent` | `AppIntent` | An instance of the intent to run |
| `phrases` | `[LocalizedStringResource]` | Siri phrases — each must include `\(.applicationName)` |
| `shortTitle` | `LocalizedStringResource` | Shown in Shortcuts app and Spotlight |
| `systemImageName` | `String` | SF Symbol for the shortcut's icon |

## Phrase rules

1. **Every phrase must include `\(.applicationName)`** — the system substitutes your app's name
2. **At most one `@Parameter` per phrase** — e.g., `\(\.$target)`
3. **Parameterized phrases create one shortcut per suggested entity value**
4. **Mix parameterized and non-parameterized phrases** — provide fallbacks without parameters
5. **Phrases are localized** — provide translations for all supported locales

### Non-parameterized phrase

Creates a single App Shortcut:

```swift
"Find closest landmark in \(.applicationName)"
```

### Parameterized phrase with AppEnum

Creates one shortcut per enum case:

```swift
"Navigate to \(\.$section) in \(.applicationName)"
// Creates: "Navigate to Landmarks in MyApp", "Navigate to Map in MyApp", etc.
```

### Parameterized phrase with AppEntity

Creates one shortcut per suggested entity:

```swift
"Open \(\.$target) in \(.applicationName)"
// Creates: "Open Niagara Falls in MyApp", "Open Eiffel Tower in MyApp", etc.
```

Requires `suggestedEntities()` on the entity's query.

## Updating parameterized shortcuts

When suggested entities change (e.g., user adds a new favorite), update the shortcuts:

```swift
// Call this whenever suggested entities change
Task {
    try await MyAppShortcuts.updateAppShortcutParameters()
}
```

Common places to call this:
- After the user adds/removes a favorite
- On app launch when data has changed
- After a sync operation completes

## Where App Shortcuts appear

Once registered, your shortcuts automatically appear in:

- **Spotlight** — shown when searching, and can run actions directly on Mac (iOS 26+)
- **Siri** — invoked by speaking or typing phrases
- **Shortcuts app** — listed in your app's section without user setup
- **Action Button** — configurable on iPhone 15 Pro+ and iPhone 16+
- **Apple Pencil Pro** — configurable squeeze action

## Spotlight integration (iOS 26+)

New in iOS 26, Shortcuts can run directly from Spotlight on Mac. For this to work:

1. Implement a `ParameterSummary` that includes **all required parameters**
2. The intent's parameters must be resolvable from Spotlight's context

```swift
struct NavigateIntent: AppIntent {
    static let title: LocalizedStringResource = "Navigate to Section"

    static var parameterSummary: some ParameterSummary {
        Summary("Navigate to \(\.$section)")
    }

    @Parameter(title: "Section")
    var section: NavigationOption

    func perform() async throws -> some IntentResult {
        // ...
    }
}
```

## Best practices

- **Define one `AppShortcutsProvider` per app.** The system expects a single provider.
- **Start with 2-3 shortcuts.** Focus on your app's most important actions.
- **Use descriptive `shortTitle` values.** These appear in the Shortcuts app and Spotlight.
- **Choose recognizable SF Symbols.** The icon helps users identify shortcuts visually.
- **Call `updateAppShortcutParameters()`** after any change to suggested entities or favorites.
- **Test in Shortcuts app.** Verify your shortcuts appear correctly and run as expected.
