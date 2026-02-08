# Intent-Driven Architecture

Structuring your app around App Intents to share logic between your UI, Siri, Shortcuts, and Spotlight.

## Core principle

App Intents should not duplicate your app's business logic. Instead, they act as a thin bridge between system integrations and your existing code. The intent describes **what** to do; your data layer handles **how**.

## Separating concerns

### Bad: Business logic in the intent

```swift
// Don't do this — logic is trapped in the intent
struct AddToFavoritesIntent: AppIntent {
    static let title: LocalizedStringResource = "Add to Favorites"
    @Parameter var landmark: LandmarkEntity

    func perform() async throws -> some IntentResult {
        let defaults = UserDefaults.standard
        var favorites = defaults.array(forKey: "favorites") as? [Int] ?? []
        favorites.append(landmark.id)
        defaults.set(favorites, forKey: "favorites")
        return .result()
    }
}
```

### Good: Intent delegates to shared logic

```swift
// Intent is a thin wrapper
struct AddToFavoritesIntent: AppIntent {
    static let title: LocalizedStringResource = "Add to Favorites"
    @Parameter var landmark: LandmarkEntity
    @Dependency var store: FavoritesStore

    func perform() async throws -> some IntentResult {
        try await store.addFavorite(landmark.id)
        return .result(dialog: "Added \(landmark.name) to favorites.")
    }
}

// Same store is used by SwiftUI views
struct LandmarkDetailView: View {
    @Environment(FavoritesStore.self) var store
    let landmark: Landmark

    var body: some View {
        Button("Favorite") {
            Task { try await store.addFavorite(landmark.id) }
        }
    }
}
```

## Entity as bridge type

Create entities as lightweight bridges to your data model, not as replacement models:

```swift
struct LandmarkEntity: AppEntity {
    // Bridge to the real model
    let landmark: Landmark

    var id: Int { landmark.id }

    @ComputedProperty
    var name: String { landmark.name }

    @ComputedProperty
    var state: String { landmark.state }

    static let defaultQuery = LandmarkEntityQuery()
    // ...
}
```

The entity wraps your model and exposes properties through `@ComputedProperty`. No data duplication.

## Navigation with TargetContentProvidingIntent

Avoid putting navigation logic in intents. Use `TargetContentProvidingIntent` with `onAppIntentExecution`:

```swift
// Intent — no navigation code, no perform method needed
struct OpenLandmarkIntent: OpenIntent, TargetContentProvidingIntent {
    static let title: LocalizedStringResource = "Open Landmark"

    @Parameter(title: "Landmark", requestValueDialog: "Which landmark?")
    var target: LandmarkEntity
}

// View handles its own navigation
struct LandmarksNavigationStack: View {
    @State var path: [Landmark] = []

    var body: some View {
        NavigationStack(path: $path) {
            LandmarkListView()
                .navigationDestination(for: Landmark.self) { landmark in
                    LandmarkDetailView(landmark: landmark)
                }
        }
        .onAppIntentExecution(OpenLandmarkIntent.self) { intent in
            path.append(intent.target.landmark)
        }
    }
}
```

Benefits:
- No `@Dependency` needed for navigation
- No `@MainActor` annotation on the intent
- Navigation logic stays in SwiftUI where it belongs
- The intent doesn't need a `perform()` method at all

### Scene control with handlesExternalEvents

Control which scene handles an intent:

```swift
// Intent has a content identifier
struct OpenLandmarkIntent: OpenIntent, TargetContentProvidingIntent {
    var contentIdentifier: String { persistentIdentifier }
    // ...
}

// Scene declares which intents it handles
WindowGroup {
    LandmarkBrowserView()
}
.handlesExternalEvents(matching: ["OpenLandmarkIntent"])

// Or conditionally on views
LandmarkBrowserView()
    .handlesExternalEvents(matching: isEditing ? [] : ["OpenLandmarkIntent"])
```

The `contentIdentifier` defaults to the intent's `persistentIdentifier` (typically the struct name).

### UIKit scene handling

For UIKit apps, use `UISceneAppIntent` or `AppIntentSceneDelegate`:

```swift
struct OpenLandmarkIntent: OpenIntent, UISceneAppIntent {
    @Parameter var target: LandmarkEntity

    func perform() async throws -> some IntentResult {
        // Access scene via self.scene
    }
}

// Or delegate-based
class SceneDelegate: UIResponder, UIWindowSceneDelegate, AppIntentSceneDelegate {
    func handle(_ intent: OpenLandmarkIntent) async {
        // Navigate to landmark
    }
}
```

## Dependency injection

Register shared dependencies once, use everywhere:

```swift
@main
struct MyApp: App {
    init() {
        // Register all dependencies
        AppDependencyManager.shared.add { ModelData() }
        AppDependencyManager.shared.add { FavoritesStore() }
        AppDependencyManager.shared.add { SearchEngine() }
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(ModelData.shared)
                .environment(FavoritesStore.shared)
        }
    }
}
```

Intents access dependencies with `@Dependency`:

```swift
struct FindLandmarkIntent: AppIntent {
    @Dependency var modelData: ModelData
    @Dependency var favorites: FavoritesStore
}
```

Register dependencies as early as possible — ideally in `App.init()`.

## Sharing types across targets

When App Intents code lives in multiple targets (app + extension + package):

### Step 1: Move shared types to a Swift Package

```swift
// In your package
public struct LandmarkEntity: AppEntity { /* ... */ }
public struct LandmarkEntityQuery: EntityQuery { /* ... */ }
```

### Step 2: Register each target with AppIntentsPackage

```swift
// In the Swift Package containing the entity
struct LandmarkIntentsPackage: AppIntentsPackage { }

// In your app target
struct AppIntentsPackage: AppIntentsPackage {
    static let includedPackages: [any AppIntentsPackage.Type] = [
        LandmarkIntentsPackage.self
    ]
}

// In your extension target
struct ExtensionIntentsPackage: AppIntentsPackage {
    static let includedPackages: [any AppIntentsPackage.Type] = [
        LandmarkIntentsPackage.self
    ]
}
```

This ensures the App Intents runtime has proper access to all types across targets.

## Architecture checklist

- [ ] Business logic lives in shared services, not in intents
- [ ] Entities are bridges to your data model, using `@ComputedProperty`
- [ ] Navigation is handled by `onAppIntentExecution`, not intent `perform()`
- [ ] Dependencies are registered in `App.init()` via `AppDependencyManager`
- [ ] Shared types are in Swift Packages with `AppIntentsPackage` registration
- [ ] Queries use `@Dependency` to access data, not direct instantiation
