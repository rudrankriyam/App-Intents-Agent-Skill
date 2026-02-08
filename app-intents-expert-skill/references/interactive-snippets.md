# Interactive Snippets (iOS 26+)

Building interactive App Intents snippets with buttons, toggles, confirmation flows, and live state updates.

## Overview

Interactive snippets let your intents display SwiftUI views that users can interact with — directly in Siri, Spotlight, and the Shortcuts app. Users can tap buttons, toggle switches, and trigger additional intents without leaving the snippet.

## SnippetIntent

A `SnippetIntent` renders a view based on its parameters and the current app state:

```swift
struct LandmarkSnippetIntent: SnippetIntent {
    static let title: LocalizedStringResource = "Landmark Snippet"

    @Parameter
    var landmark: LandmarkEntity

    @Dependency
    var modelData: ModelData

    func perform() async throws -> some IntentResult & ShowsSnippetView {
        let isFavorite = await modelData.isFavorite(landmark)
        return .result(
            view: LandmarkView(landmark: landmark, isFavorite: isFavorite)
        )
    }
}
```

### Requirements

- All view-driving variables **must be marked as `@Parameter`** — the system populates them automatically
- **Do NOT mutate app state** in the snippet intent's `perform()` — it's called multiple times during the snippet lifecycle
- Render views quickly — slow snippets feel unresponsive
- The snippet view uses the same SwiftUI as interactive widgets

## Returning a snippet from a parent intent

Use `ShowsSnippetIntent` in the return type:

```swift
struct ClosestLandmarkIntent: AppIntent {
    static let title: LocalizedStringResource = "Find Closest Landmark"

    @Dependency var modelData: ModelData

    func perform() async throws -> some ReturnsValue<LandmarkEntity> & ShowsSnippetIntent & ProvidesDialog {
        let landmark = await findClosestLandmark()
        return .result(
            value: landmark,
            dialog: IntentDialog(
                full: "The closest landmark is \(landmark.name).",
                supporting: "\(landmark.name) is located in \(landmark.continent)."
            ),
            snippetIntent: LandmarkSnippetIntent(landmark: landmark)
        )
    }
}
```

The system uses the `snippetIntent` parameter to manage the snippet's lifecycle. The parameter values you provide (here, `landmark`) are stored and used to repopulate the snippet intent each time it refreshes.

## Interactive buttons and toggles

Associate buttons and toggles with App Intents inside your snippet view:

```swift
struct LandmarkView: View {
    let landmark: LandmarkEntity
    let isFavorite: Bool

    var body: some View {
        HStack {
            Text(landmark.name)

            Button(intent: UpdateFavoritesIntent(
                landmark: landmark,
                isFavorite: !isFavorite
            )) {
                Image(systemName: isFavorite ? "heart.fill" : "heart")
            }

            Button(intent: FindTicketsIntent(landmark: landmark)) {
                Text("Find Tickets")
            }
        }
    }
}
```

- `Button(intent:)` — runs the associated intent when tapped
- `Toggle(isOn:intent:)` — runs the intent when toggled

These are the same APIs used for interactive widgets.

## Snippet update cycle

When a button or toggle is tapped:

1. The system runs the associated intent (e.g., `UpdateFavoritesIntent`)
2. Waits for it to complete
3. Re-populates the snippet intent's parameters with the original values
4. App Entity parameters are **re-fetched from their queries** (getting latest state)
5. Runs the snippet intent's `perform()` again
6. The updated view replaces the previous one with animation

This cycle continues until the snippet is dismissed. Use `contentTransition` modifiers to customize animations between states.

## Confirmation snippets

Present an interactive configuration before proceeding:

```swift
struct FindTicketsIntent: AppIntent {
    static let title: LocalizedStringResource = "Find Tickets"

    @Parameter var landmark: LandmarkEntity

    func perform() async throws -> some IntentResult & ShowsSnippetIntent {
        let searchRequest = await searchEngine.createRequest(for: landmark)

        // Present interactive configuration
        try await requestConfirmation(
            actionName: .search,
            snippetIntent: TicketRequestSnippetIntent(searchRequest: searchRequest)
        )

        // User tapped the action button — proceed with search
        let results = try await searchEngine.search(searchRequest)
        return .result(
            snippetIntent: TicketResultsSnippetIntent(results: results)
        )
    }
}
```

If the user cancels the confirmation, `requestConfirmation` throws an error. **Do not catch this error** — let it terminate the perform method.

## Snippet chaining

A button inside a result snippet can trigger an intent that presents its own snippet, replacing the current one:

```
Result Snippet → Button tap → New Intent → Confirmation Snippet or Result Snippet
```

This only works when the original snippet is showing a **result** (not a confirmation).

## Reloading snippets

Force a snippet to refresh when app state changes:

```swift
// Reload a specific snippet intent
await LandmarkSnippetIntent.reload()
```

Useful when background state changes should be reflected in a visible snippet.

## Key design rules

- **Model entities, not values.** Use `AppEntity` for parameters that need fresh data on refresh. Primitive values are only set once and never re-fetched.
- **Don't store mutable state on snippet intents.** The system recreates them each cycle.
- **Keep perform fast.** Slow rendering makes the snippet feel broken.
- **The system won't terminate your app** while a snippet is visible — feel free to hold state in memory.
- **Dark mode and dynamic type** — the system may re-run your snippet intent to adapt to device changes.
- **Use `contentTransition`** to animate changes between snippet updates.

## Complete example

```swift
// The snippet intent
struct OrderSnippetIntent: SnippetIntent {
    static let title: LocalizedStringResource = "Order Snippet"

    @Parameter var order: OrderEntity
    @Dependency var store: OrderStore

    func perform() async throws -> some IntentResult & ShowsSnippetView {
        let status = await store.status(for: order)
        return .result(
            view: OrderStatusView(order: order, status: status)
        )
    }
}

// The view with interactive buttons
struct OrderStatusView: View {
    let order: OrderEntity
    let status: OrderStatus

    var body: some View {
        VStack {
            Text(order.name)
            Text(status.description)

            if status == .ready {
                Button(intent: ConfirmPickupIntent(order: order)) {
                    Text("Confirm Pickup")
                }
            }
        }
        .contentTransition(.numericText())
    }
}
```
