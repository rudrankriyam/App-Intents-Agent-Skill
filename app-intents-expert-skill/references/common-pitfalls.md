# Common Pitfalls

Frequent errors, gotchas, and debugging techniques for App Intents development.

## Build-time errors

### "Expression is not a string literal"

**Cause:** `title`, `typeDisplayRepresentation`, or `caseDisplayRepresentations` is not a compile-time constant.

```swift
// BAD — computed property
static var title: LocalizedStringResource {
    "My Intent \(someValue)"  // Error
}

// GOOD — constant
static let title: LocalizedStringResource = "My Intent"
```

App Intents metadata extraction happens at **build time**. Titles, type representations, and case representations must be constant values that can be evaluated without running your code.

### "Type does not conform to AppEntity"

**Checklist:**
- [ ] Is the struct `Identifiable` with a stable `id`?
- [ ] Does it have `typeDisplayRepresentation`?
- [ ] Does it have `displayRepresentation`?
- [ ] Does it have `static let defaultQuery`?
- [ ] Is the query type correctly implementing `EntityQuery`?

### "AppIntentsPackage not found" or "Type not indexed"

When sharing types across targets (app + extension + package):

```swift
// Every target with App Intents types needs a package
struct MyPackage: AppIntentsPackage { }

// Parent targets must include child packages
struct AppTargetPackage: AppIntentsPackage {
    static let includedPackages: [any AppIntentsPackage.Type] = [
        MyPackage.self
    ]
}
```

### Intent not appearing in Shortcuts

**Checklist:**
- [ ] Is the intent in a target that's installed on the device?
- [ ] Does the intent have a valid `title`?
- [ ] If using App Shortcuts, is there an `AppShortcutsProvider`?
- [ ] Clean build folder (Cmd+Shift+K) and rebuild
- [ ] Delete the app from the device and reinstall

## Runtime errors

### "Entity not found" when resolving parameters

The entity's `entities(for:)` query method returned an empty array for the requested ID.

**Common causes:**
- Entity ID changed (IDs must be persistent and stable)
- Data was deleted but the system still holds a reference
- Query doesn't have access to the data store (missing `@Dependency`)

```swift
// Ensure entities(for:) always attempts to resolve
func entities(for identifiers: [LandmarkEntity.ID]) async throws -> [LandmarkEntity] {
    // Don't filter out missing IDs silently — the system expects results
    modelData.landmarks(for: identifiers).map(LandmarkEntity.init)
}
```

### Dependencies returning nil or wrong instance

Dependencies must be registered before any intent runs:

```swift
@main
struct MyApp: App {
    init() {
        // Register FIRST
        AppDependencyManager.shared.add { ModelData() }
    }
}
```

If you register too late (e.g., in a view's `onAppear`), intents triggered before that point will crash.

### Snippet intent crashes or shows blank

**Checklist:**
- [ ] All view-driving properties are marked `@Parameter`
- [ ] `perform()` returns quickly (no long network calls)
- [ ] View doesn't access unavailable state
- [ ] Entity parameters have valid queries that return fresh data

### "The operation couldn't be completed"

Generic error often caused by:
- Intent `perform()` throwing an unexpected error
- Network request timing out inside perform
- Missing required parameter that wasn't resolved

Wrap operations in do/catch for better diagnostics:

```swift
func perform() async throws -> some IntentResult {
    do {
        try await riskyOperation()
    } catch {
        // Log the actual error
        print("Intent failed: \(error)")
        throw error
    }
    return .result()
}
```

## Design gotchas

### Don't mutate state in SnippetIntent.perform()

Snippet intents are called **multiple times** during their lifecycle. Side effects will execute repeatedly:

```swift
// BAD — adds to favorites every time the snippet refreshes
struct MySnippet: SnippetIntent {
    func perform() async throws -> some IntentResult & ShowsSnippetView {
        store.addFavorite(item)  // Called on every refresh!
        return .result(view: MyView())
    }
}

// GOOD — only reads state
struct MySnippet: SnippetIntent {
    func perform() async throws -> some IntentResult & ShowsSnippetView {
        let isFavorite = store.isFavorite(item)
        return .result(view: MyView(isFavorite: isFavorite))
    }
}
```

### Don't catch requestConfirmation cancellation

When the user cancels a confirmation, let the error propagate:

```swift
// BAD
func perform() async throws -> some IntentResult {
    do {
        try await requestConfirmation(dialog: "Are you sure?")
    } catch {
        return .result(dialog: "Cancelled")  // Don't do this
    }
    // ...
}

// GOOD
func perform() async throws -> some IntentResult {
    try await requestConfirmation(dialog: "Are you sure?")
    // If we reach here, user confirmed
    // ...
}
```

### Entity IDs must not change

If you change an entity's ID scheme, all existing references break:

```swift
// BAD — ID changes if database row is recreated
var id: UUID { UUID() }  // New UUID every time!

// GOOD — stable, persistent ID
var id: Int { landmark.databaseID }
```

### @Parameter on optional vs required

Optional parameters don't prompt the user. Required parameters do:

```swift
@Parameter var name: String        // Required — system will ask for value
@Parameter var name: String?       // Optional — nil if not provided
@Parameter(default: "Default") var name: String  // Required but has a default
```

## Debugging techniques

### Print metadata extraction

Build and check the extracted metadata:

```bash
# In your app's derived data
find ~/Library/Developer/Xcode/DerivedData -name "*.appintentsmetadata" | head -5
```

### Check if intents are registered

In the Shortcuts app, search for your app. If intents don't appear:
1. Clean build folder
2. Delete the app
3. Rebuild and reinstall
4. Wait a few seconds for metadata extraction

### Test in Shortcuts first

Before testing with Siri, always verify your intents work correctly in the Shortcuts app. Shortcuts provides more detailed error information.

### Console logging

Filter Console.app for App Intents related logs:

```
subsystem:com.apple.appintents
```

## Performance gotchas

### Don't make entities expensive to create

Entities are created frequently (in queries, parameter resolution, etc.). Keep initialization lightweight:

```swift
// BAD — network call during init
struct LandmarkEntity: AppEntity {
    var id: Int
    var crowdStatus: String  // Set via network call in query

    // ...
}

// GOOD — use @DeferredProperty for expensive data
struct LandmarkEntity: AppEntity {
    var id: Int

    @DeferredProperty
    var crowdStatus: String {
        get async { await NetworkService.shared.fetchCrowdStatus(for: id) }
    }
}
```

### Don't return too many suggested entities

`suggestedEntities()` should return a focused list (10-20 items), not your entire database:

```swift
func suggestedEntities() async throws -> [LandmarkEntity] {
    // GOOD — focused list
    modelData.favoriteLandmarks.prefix(15).map(LandmarkEntity.init)
}
```

### Batch Spotlight donations

Don't donate entities one by one:

```swift
// BAD
for landmark in landmarks {
    try await CSSearchableIndex.default().indexAppEntities([LandmarkEntity(landmark: landmark)])
}

// GOOD
let entities = landmarks.map(LandmarkEntity.init)
try await CSSearchableIndex.default().indexAppEntities(entities)
```
