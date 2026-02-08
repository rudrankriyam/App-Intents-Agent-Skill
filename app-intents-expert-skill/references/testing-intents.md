# Testing App Intents

Strategies for unit testing intents, entities, queries, and integration testing with Siri and Shortcuts.

## Unit testing intents

App Intents are plain Swift structs, making them directly testable:

```swift
import Testing
import AppIntents
@testable import MyApp

@Suite
struct NavigateIntentTests {
    @Test
    func navigateToLandmarks() async throws {
        var intent = NavigateIntent()
        intent.section = .landmarks

        let result = try await intent.perform()
        // Verify navigation occurred
        #expect(Navigator.shared.currentSection == .landmarks)
    }

    @Test
    func navigateToMap() async throws {
        var intent = NavigateIntent()
        intent.section = .map

        let result = try await intent.perform()
        #expect(Navigator.shared.currentSection == .map)
    }
}
```

### Setting parameter values

Set `@Parameter` values directly on the intent struct:

```swift
var intent = OpenLandmarkIntent()
intent.target = LandmarkEntity(landmark: testLandmark)
```

## Testing with dependencies

Register test doubles before running intents:

```swift
@Suite
struct FindLandmarkTests {
    init() {
        // Register mock dependencies
        AppDependencyManager.shared.add { MockModelData() as ModelData }
    }

    @Test
    func findClosestLandmark() async throws {
        let intent = FindClosestLandmarkIntent()
        let result = try await intent.perform()
        // Verify the returned entity
    }
}
```

### Dependency injection pattern

Design your dependencies as protocols for easy mocking:

```swift
protocol ModelDataProviding {
    func findClosestLandmark() async throws -> Landmark
    func landmarks(for ids: [Int]) -> [Landmark]
}

class ModelData: ModelDataProviding { /* real implementation */ }
class MockModelData: ModelDataProviding { /* test implementation */ }
```

Register the mock in tests:

```swift
AppDependencyManager.shared.add { MockModelData() as ModelDataProviding }
```

## Testing entities and queries

### Testing entity creation

```swift
@Test
func entityCreation() {
    let landmark = Landmark(id: 1, name: "Niagara Falls", state: "New York")
    let entity = LandmarkEntity(landmark: landmark)

    #expect(entity.id == 1)
    #expect(entity.name == "Niagara Falls")
}
```

### Testing queries

```swift
@Test
func queryByIdentifiers() async throws {
    AppDependencyManager.shared.add { MockModelData() as ModelData }

    let query = LandmarkEntityQuery()
    let results = try await query.entities(for: [1, 2, 3])

    #expect(results.count == 3)
    #expect(results[0].id == 1)
}

@Test
func queryStringSearch() async throws {
    AppDependencyManager.shared.add { MockModelData() as ModelData }

    let query = LandmarkEntityQuery()
    let results = try await query.entities(matching: "Niagara")

    #expect(results.count == 1)
    #expect(results[0].name == "Niagara Falls")
}

@Test
func querySuggestedEntities() async throws {
    AppDependencyManager.shared.add { MockModelData() as ModelData }

    let query = LandmarkEntityQuery()
    let suggestions = try await query.suggestedEntities()

    #expect(!suggestions.isEmpty)
}
```

## Testing App Shortcuts

Verify your `AppShortcutsProvider` is correctly configured:

```swift
@Test
func shortcutsProviderHasExpectedShortcuts() {
    let shortcuts = MyAppShortcuts.appShortcuts
    #expect(shortcuts.count >= 2)
}
```

### Verifying phrases

While you can't directly test Siri phrase matching, you can verify the shortcuts are configured with phrases:

```swift
@Test
func shortcutPhrasesIncludeAppName() {
    // Verify at build time by inspecting the provider
    // Siri matching is tested manually on device
    let shortcuts = MyAppShortcuts.appShortcuts
    #expect(!shortcuts.isEmpty)
}
```

## Integration testing

### Testing in Shortcuts app

1. Build and run your app on a device or simulator
2. Open the Shortcuts app
3. Create a new shortcut with your intent
4. Verify parameters are editable and the intent runs correctly
5. Test with different parameter combinations

### Testing with Siri

1. Build and run on a physical device (Siri works best on device)
2. Invoke Siri and speak your App Shortcut phrases
3. Verify:
   - Siri recognizes the phrase
   - Parameters are resolved correctly
   - Dialog is spoken/displayed
   - View snippets render properly
   - The app foregrounds when expected

### Testing Spotlight integration

1. Donate entities to Spotlight
2. Open Spotlight and search for entity names
3. Verify entities appear in results
4. Tap results and verify navigation

## Common testing patterns

### Resetting state between tests

```swift
@Suite
struct IntentTests {
    init() {
        // Reset shared state
        AppDependencyManager.shared.add { FreshModelData() as ModelData }
    }
}
```

### Testing error cases

```swift
@Test
func performThrowsWhenLandmarkNotFound() async throws {
    AppDependencyManager.shared.add { EmptyModelData() as ModelData }

    let intent = FindClosestLandmarkIntent()
    await #expect(throws: AppIntentError.self) {
        try await intent.perform()
    }
}
```

### Testing return values

```swift
@Test
func intentReturnsCorrectEntity() async throws {
    var intent = FindClosestLandmarkIntent()
    let result = try await intent.perform()

    // Access the returned value through the result
    // The exact API depends on your return type composition
}
```

## What you can't unit test

Some aspects require manual testing on device:

- **Siri phrase recognition** — depends on speech recognition
- **Spotlight ranking** — depends on the indexing service
- **Interactive snippets rendering** — requires the system snippet host
- **App Shortcut parameter generation** — requires `updateAppShortcutParameters()`
- **Apple Intelligence integration** — requires device with Apple Intelligence enabled
- **Visual Intelligence image search** — requires camera/screenshot context

## Best practices

- **Test intents as plain structs.** They're just Swift — use standard testing patterns.
- **Mock dependencies with protocols.** Register test doubles in `AppDependencyManager`.
- **Test queries thoroughly.** Query correctness is critical — the system relies on them.
- **Test edge cases.** Empty results, invalid IDs, network failures.
- **Use device testing for Siri.** Simulator Siri support is limited.
- **Automate what you can.** Unit test the logic, manually test the system integration.
