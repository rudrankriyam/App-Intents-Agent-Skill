# Spotlight Indexing

Making your entities searchable in Spotlight with `IndexedEntity`, property indexing keys, and entity donation.

## IndexedEntity

Conform your entity to `IndexedEntity` to make it searchable in Spotlight:

```swift
struct LandmarkEntity: IndexedEntity {
    var id: Int { landmark.id }

    @Property(indexingKey: \.displayName)
    var name: String

    @Property(indexingKey: \.contentDescription)
    var description: String

    let landmark: Landmark

    static let typeDisplayRepresentation = TypeDisplayRepresentation(name: "Landmark")

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(name)")
    }

    static let defaultQuery = LandmarkEntityQuery()
}
```

### Indexing keys (iOS 26+)

The `indexingKey` parameter on `@Property` maps entity properties to Core Spotlight attribute keys. Common keys:

| Indexing Key | Purpose |
|---|---|
| `\.displayName` | Primary searchable name |
| `\.contentDescription` | Description text for search and display |
| `\.keywords` | Additional search keywords |
| `\.thumbnailData` | Thumbnail image data |

Custom indexing keys are also supported for domain-specific search filtering. For example, adding a continent property with a custom key allows Spotlight to filter landmarks by typing "Asia."

## Donating entities to Spotlight

After conforming to `IndexedEntity`, donate entities so Spotlight can index them:

```swift
// Donate a single entity
try await CSSearchableIndex.default().indexAppEntities([landmarkEntity])

// Donate multiple entities
let entities = landmarks.map(LandmarkEntity.init)
try await CSSearchableIndex.default().indexAppEntities(entities)

// Remove entities
try await CSSearchableIndex.default().deleteAppEntities(
    ofType: LandmarkEntity.self,
    identifiedBy: [removedLandmark.id]
)
```

### When to donate

- On app launch (for initial indexing)
- When new content is created or imported
- After a sync operation
- When content is updated (re-donate with same ID to update)

## Handling Spotlight taps

When a user taps a Spotlight result, the system looks for a matching `OpenIntent`:

```swift
struct OpenLandmarkIntent: OpenIntent, TargetContentProvidingIntent {
    static let title: LocalizedStringResource = "Open Landmark"

    @Parameter(title: "Landmark", requestValueDialog: "Which landmark?")
    var target: LandmarkEntity
}
```

**You must implement an `OpenIntent` for your entity type.** Without it, tapping a Spotlight result will just foreground the app without navigation.

### SwiftUI navigation from Spotlight

Use `onAppIntentExecution` to handle the navigation in your view:

```swift
struct LandmarksNavigationStack: View {
    @State var path: [Landmark] = []

    var body: some View {
        NavigationStack(path: $path) {
            LandmarkListView()
        }
        .onAppIntentExecution(OpenLandmarkIntent.self) { intent in
            path.append(intent.target.landmark)
        }
    }
}
```

## Spotlight actions on Mac (iOS 26+)

New in iOS 26, intents with a complete `ParameterSummary` can run directly from Spotlight on Mac:

```swift
struct NavigateIntent: AppIntent {
    static let title: LocalizedStringResource = "Navigate to Section"

    static var parameterSummary: some ParameterSummary {
        Summary("Navigate to \(\.$section)")
    }

    @Parameter(title: "Section")
    var section: NavigationOption

    // ...
}
```

Requirements:
- `ParameterSummary` must include **all required parameters**
- Parameters must be resolvable from Spotlight's context (enum values, suggested entities)

## Prioritizing entities with on-screen context

Associate entities with visible content so Spotlight prioritizes them in suggestions:

```swift
struct LandmarkDetailView: View {
    let landmark: LandmarkEntity

    var body: some View {
        ScrollView { /* content */ }
            .userActivity("com.myapp.landmark") { activity in
                activity.appEntityIdentifier = EntityIdentifier(landmark)
            }
    }
}
```

This tells the system which entities are currently relevant, improving Spotlight suggestions and Apple Intelligence context.

## Auto-generated Find actions

When you adopt `IndexedEntity` with property indexing keys, the Shortcuts app **automatically generates Find and Filter actions** for your entities. Users can build shortcuts that search and filter your entities by the indexed properties without you writing any additional code.

## Best practices

- **Donate frequently.** Spotlight rankings improve with fresh data.
- **Use meaningful indexing keys.** Map properties to the right Core Spotlight attributes for better search relevance.
- **Always implement OpenIntent.** Without it, Spotlight results are dead ends.
- **Keep entity IDs stable.** Spotlight stores references by ID — changing IDs creates orphaned entries.
- **Re-donate on update.** Donating an entity with an existing ID updates its Spotlight entry.
- **Batch donations.** Use `indexAppEntities([...])` with arrays for better performance.
