# Apple Intelligence Integration

Integrating with Apple Intelligence through assistant schemas, the `@AppIntent` macro, on-screen entities, and Visual Intelligence image search.

## Assistant schemas and the @AppIntent macro

Apple Intelligence uses **assistant schemas** to understand your app's intents in the context of specific domains. The `@AppIntent` macro (replacing `AssistantIntent` in iOS 26+) annotates your intent with a schema:

```swift
@AppIntent(.photos, .search)
struct SearchPhotosIntent: AppIntent {
    static let title: LocalizedStringResource = "Search Photos"

    var searchCriteria: PhotoSearchCriteria

    func perform() async throws -> some IntentResult {
        // Search photos using criteria
    }
}
```

### Available domains

Apple Intelligence supports predefined schemas across domains including:

- **Photos** — search, edit, organize
- **Mail** — compose, search, organize
- **Messages** — send, search
- **Books** — open, search
- **Browsers** — open tabs, search
- **Spreadsheets** — create, open
- **Word processors** — create, open
- **Presentations** — create, open
- **File management** — open, search
- **Journaling** — create entries

Check `AppIntentDomains` in Apple's documentation for the full list of available domains and their schemas.

### Semantic content search (iOS 26+)

The `@AppIntent` macro also supports Visual Intelligence schemas:

```swift
@AppIntent(.semanticContentSearch)
struct SearchByImageIntent: AppIntent {
    static let title: LocalizedStringResource = "Search by Image"

    var semanticContent: SemanticContentDescriptor

    func perform() async throws -> some IntentResult {
        // Process the semantic content and navigate to search
    }
}
```

The macro automatically marks the schema-required properties as intent parameters.

## Visual Intelligence / Image Search (iOS 26+)

Allow users to search your app's content by pointing their camera at objects or selecting screenshots:

### Step 1: Implement an IntentValueQuery

```swift
struct LandmarkImageQuery: IntentValueQuery {
    @Dependency var modelData: ModelData

    func values(for descriptor: SemanticContentDescriptor) async throws -> [LandmarkEntity] {
        // Convert descriptor pixels to a recognizable format
        guard let cgImage = descriptor.cgImage else { return [] }

        // Search your data model using the image
        let matches = try await modelData.searchLandmarks(matching: cgImage)
        return matches.map(LandmarkEntity.init)
    }
}
```

### Step 2: Implement an OpenIntent

Required for tapping on search results:

```swift
struct OpenLandmarkIntent: OpenIntent {
    static let title: LocalizedStringResource = "Open Landmark"

    @Parameter(title: "Landmark")
    var target: LandmarkEntity
}
```

### Step 3: Support mixed result types with UnionValue

If your image search can return different entity types:

```swift
enum SearchResult: UnionValue {
    case landmark(LandmarkEntity)
    case collection(CollectionEntity)
}

struct ImageSearchQuery: IntentValueQuery {
    func values(for descriptor: SemanticContentDescriptor) async throws -> [SearchResult] {
        let landmarks = try await searchLandmarks(descriptor)
        let collections = try await searchCollections(descriptor)
        return landmarks.map { .landmark($0) } + collections.map { .collection($0) }
    }
}
```

Implement `OpenIntent` for each entity type in the union.

### Image search best practices

- Return a few pages of results to increase the chance of a match
- Use your servers for larger dataset searches, but don't take too long
- Provide a "more results" option that opens your app's full search view

## On-screen entities

Associate App Entities with visible content so Apple Intelligence (and ChatGPT) can understand what the user is looking at:

### Step 1: Attach entity to a view

```swift
struct LandmarkDetailView: View {
    let landmark: LandmarkEntity

    var body: some View {
        ScrollView { /* content */ }
            .userActivity("com.myapp.viewLandmark") { activity in
                activity.appEntityIdentifier = EntityIdentifier(landmark)
            }
    }
}
```

### Step 2: Make entity Transferable

ChatGPT needs data it can process. Conform your entity to `Transferable` with a supported representation:

```swift
extension LandmarkEntity: Transferable {
    static var transferRepresentation: some TransferRepresentation {
        // PDF for rich content
        DataRepresentation(exportedContentType: .pdf) {
            try $0.generatePDF()
        }

        // Plain text fallback
        DataRepresentation(exportedContentType: .plainText) {
            $0.textDescription.data(using: .utf8) ?? Data()
        }
    }
}
```

Supported types for ChatGPT integration:
- PDF (`.pdf`)
- Plain text (`.plainText`)
- Rich text (`.rtf`)

### How it works

When a user asks Siri about something on screen (e.g., "Is this place near the ocean?"):
1. Siri identifies the on-screen entity via `userActivity`
2. Offers to send a screenshot or the full entity content to ChatGPT
3. ChatGPT processes the content and responds

## PredictableIntent

Help the system learn user patterns and suggest intents proactively:

```swift
struct OpenLandmarkIntent: AppIntent, PredictableIntent {
    static let title: LocalizedStringResource = "Open Landmark"

    @Parameter var target: LandmarkEntity

    static var predictionConfiguration: some IntentPredictionConfiguration {
        IntentPrediction(parameters: (\.$target, .value(niagara))) {
            DisplayRepresentation(title: "Open Niagara Falls")
        }
        IntentPrediction(parameters: (\.$target, .value(eiffel))) {
            DisplayRepresentation(title: "Open Eiffel Tower")
        }
    }
}
```

The system uses these predictions alongside user behavior to surface relevant suggestions.

## Key rules

- **`OpenIntent` is required** for image search results to be tappable.
- **`@AppIntent` macro replaces `AssistantIntent`** in iOS 26+ — use the macro for new code.
- **On-screen entities need `Transferable`** to share content with ChatGPT.
- **Don't block image search queries.** Return results quickly, even if incomplete.
- **Use `userActivity` on detail views** to associate entities with visible content.
