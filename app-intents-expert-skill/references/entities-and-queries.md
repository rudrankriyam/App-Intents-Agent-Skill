# Entities and Queries

Building `AppEntity`, `AppEnum`, and query types for dynamic data resolution across Siri, Shortcuts, Spotlight, and Apple Intelligence.

## AppEnum — fixed set of values

Use `AppEnum` for types with a constant, known set of values:

```swift
enum NavigationOption: String, AppEnum {
    case landmarks
    case map
    case collections

    static let typeDisplayRepresentation: TypeDisplayRepresentation = "Navigation Option"

    static let caseDisplayRepresentations: [NavigationOption: DisplayRepresentation] = [
        .landmarks: DisplayRepresentation(
            title: "Landmarks",
            image: .init(systemName: "building.columns")
        ),
        .map: DisplayRepresentation(
            title: "Map",
            image: .init(systemName: "map")
        ),
        .collections: DisplayRepresentation(
            title: "Collections",
            image: .init(systemName: "book.closed")
        )
    ]
}
```

Requirements:
- Must have a `String` raw value
- `typeDisplayRepresentation` — describes the type as a whole
- `caseDisplayRepresentations` — describes each case
- All representations must be **compile-time constants**

## AppEntity — dynamic values

Use `AppEntity` for values that are dynamic and resolved at runtime:

```swift
struct LandmarkEntity: AppEntity {
    var id: Int { landmark.id }

    @ComputedProperty
    var name: String { landmark.name }

    @ComputedProperty
    var description: String { landmark.description }

    let landmark: Landmark

    static let typeDisplayRepresentation = TypeDisplayRepresentation(name: "Landmark")

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(name)")
    }

    static let defaultQuery = LandmarkEntityQuery()
}
```

Requirements:
- Must be `Identifiable` with a **persistent, stable** identifier
- The system must be able to look up entities by their ID at any time
- `typeDisplayRepresentation` — compile-time constant describing the type
- `displayRepresentation` — instance property describing each entity
- `defaultQuery` — the query the system uses to resolve entities

## Property types

### @Property — stored on the entity

Standard properties exposed to Shortcuts and the system:

```swift
@Property(title: "Name")
var name: String
```

### @ComputedProperty (iOS 26+) — derived from source of truth

Avoids storing duplicate values. Reads from the actual data model:

```swift
@ComputedProperty
var name: String { landmark.name }

@ComputedProperty
var defaultPlace: String { UserDefaults.standard.string(forKey: "place") ?? "" }
```

Prefer `@ComputedProperty` over `@Property` when the value can be derived from another source.

### @DeferredProperty (iOS 26+) — expensive, loaded on demand

For values that require network calls or heavy computation. The async getter is only called when the system explicitly requests the value:

```swift
@DeferredProperty
var crowdStatus: String {
    get async {
        await networkService.fetchCrowdStatus(for: id)
    }
}
```

### Decision guide

| Property type | When to use | Overhead |
|---|---|---|
| `@Property` | Value is stored directly on the entity struct | Low |
| `@ComputedProperty` | Value derived from another source (model, UserDefaults) | Low |
| `@DeferredProperty` | Value requires async work (network, heavy I/O) | Lowest (lazy) |

## Queries

Queries tell the system how to find and resolve entities.

### EntityQuery (base)

The minimum — lookup entities by their IDs:

```swift
struct LandmarkEntityQuery: EntityQuery {
    @Dependency
    var modelData: ModelData

    func entities(for identifiers: [LandmarkEntity.ID]) async throws -> [LandmarkEntity] {
        modelData
            .landmarks(for: identifiers)
            .map(LandmarkEntity.init)
    }
}
```

**Every query must implement `entities(for:)`** — the system uses this to resolve entity references.

### Suggested entities

Provide default entities shown when a parameter needs a value:

```swift
struct LandmarkEntityQuery: EntityQuery {
    @Dependency var modelData: ModelData

    func entities(for identifiers: [LandmarkEntity.ID]) async throws -> [LandmarkEntity] {
        modelData.landmarks(for: identifiers).map(LandmarkEntity.init)
    }

    func suggestedEntities() async throws -> [LandmarkEntity] {
        modelData.favoriteLandmarks.map(LandmarkEntity.init)
    }
}
```

Suggested entities are also used to generate parameterized App Shortcuts.

### EnumerableEntityQuery

When all entities can fit in memory:

```swift
struct ColorEntityQuery: EnumerableEntityQuery {
    func allEntities() async throws -> [ColorEntity] {
        ColorEntity.allColors
    }

    func entities(for identifiers: [ColorEntity.ID]) async throws -> [ColorEntity] {
        allEntities().filter { identifiers.contains($0.id) }
    }
}
```

The framework can derive more complex query behaviors from `allEntities()`.

### EntityStringQuery

Search entities by a string:

```swift
struct LandmarkEntityQuery: EntityStringQuery {
    @Dependency var modelData: ModelData

    func entities(for identifiers: [LandmarkEntity.ID]) async throws -> [LandmarkEntity] {
        modelData.landmarks(for: identifiers).map(LandmarkEntity.init)
    }

    func entities(matching string: String) async throws -> [LandmarkEntity] {
        modelData.landmarks
            .filter { $0.name.localizedCaseInsensitiveContains(string) ||
                      $0.description.localizedCaseInsensitiveContains(string) }
            .map(LandmarkEntity.init)
    }
}
```

### EntityPropertyQuery

Filter entities by specific properties. Enables the "Find and Filter" action in Shortcuts:

```swift
struct LandmarkEntityQuery: EntityPropertyQuery {
    static let properties = QueryProperties {
        Property(\LandmarkEntity.$state) {
            EqualToComparator { $0 }
            ContainsComparator { $0 }
        }
        Property(\LandmarkEntity.$isFeatured) {
            EqualToComparator { $0 }
        }
    }

    static let sortingOptions = SortingOptions {
        SortableBy(\LandmarkEntity.$name)
    }

    func entities(
        matching comparators: [LandmarkEntityQuery.Comparator],
        mode: ComparatorMode,
        sortedBy: [Sort<LandmarkEntity>],
        limit: Int?
    ) async throws -> [LandmarkEntity] {
        // Apply comparators and sorting to your data source
    }

    func entities(for identifiers: [LandmarkEntity.ID]) async throws -> [LandmarkEntity] {
        // ID-based lookup
    }
}
```

## Transferable conformance

Make entities shareable with other apps and usable in more Shortcuts actions:

```swift
extension LandmarkEntity: Transferable {
    static var transferRepresentation: some TransferRepresentation {
        DataRepresentation(exportedContentType: .image) {
            return try $0.imageRepresentationData
        }
    }
}
```

This enables:
- Using entity images as inputs to photo-related Shortcuts actions
- Sharing entity data between apps
- Providing content to ChatGPT via on-screen entity (PDF, plain text, rich text)

## UnionValue — mixed entity types

Return different entity types from a single query:

```swift
enum SearchResult: UnionValue {
    case landmark(LandmarkEntity)
    case collection(CollectionEntity)
}
```

Useful for Visual Intelligence image search where results may include multiple entity types.

## Key rules

- **Entity IDs must be persistent.** The system stores references to entities by ID and resolves them later.
- **Always implement `entities(for:)`.** This is how the system looks up entities by their stored IDs.
- **Use `@Dependency` for data access.** Don't create new data stores inside queries.
- **Display representations on types must be constant.** `typeDisplayRepresentation` is read at build time.
- **Display representations on instances can be dynamic.** `displayRepresentation` is computed at runtime.
- **Call `updateAppShortcutParameters()`** after suggested entities change to regenerate parameterized App Shortcuts.
