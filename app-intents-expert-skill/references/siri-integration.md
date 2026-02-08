# Siri Integration

Making your App Intents work with Siri, including phrases, dialog, SiriTipView, and the Gemini-powered capabilities in iOS 26.4+.

## How Siri discovers your intents

Siri finds your intents through two mechanisms:

1. **App Shortcuts** — intents registered in `AppShortcutsProvider` with phrases (see [shortcuts-provider.md](shortcuts-provider.md))
2. **Assistant schemas** — intents annotated with `@AppIntent` macro for Apple Intelligence domains (see [apple-intelligence.md](apple-intelligence.md))

Siri does NOT automatically discover all `AppIntent` conforming types. You must explicitly surface them through one of these mechanisms.

## Siri phrases

Phrases are defined in `AppShortcut` and must include `\(.applicationName)`:

```swift
AppShortcut(
    intent: FindClosestLandmarkIntent(),
    phrases: [
        "Find closest landmark in \(.applicationName)",
        "What's near me in \(.applicationName)",
        "Nearest landmark in \(.applicationName)"
    ],
    shortTitle: "Closest Landmark",
    systemImageName: "location"
)
```

### Parameterized phrases

Include at most one parameter per phrase. The system creates a shortcut for each suggested entity value:

```swift
AppShortcut(
    intent: OpenLandmarkIntent(),
    phrases: [
        "Open \(\.$target) in \(.applicationName)",
        "Show \(\.$target) in \(.applicationName)"
    ],
    shortTitle: "Open Landmark",
    systemImageName: "mappin"
)
```

For this to work, the entity's query must implement `suggestedEntities()`, and you must call `updateAppShortcutParameters()` when those entities change.

## Dialog

Dialog is how your intent communicates results through Siri's voice:

### Simple dialog

```swift
return .result(dialog: "Done! Your collection has been created.")
```

### Structured dialog (iOS 26+)

Provide both full and supporting dialog:

```swift
return .result(
    dialog: IntentDialog(
        full: "The closest landmark is \(name).",
        supporting: "\(name) is located in \(continent)."
    )
)
```

- `full` — the primary spoken/displayed text
- `supporting` — additional context shown visually

### Request value dialog

Customize the prompt when asking for parameter values:

```swift
@Parameter(title: "Landmark", requestValueDialog: "Which landmark would you like to open?")
var target: LandmarkEntity
```

## SiriTipView

Display a tip in your app's UI that teaches users the Siri phrase:

```swift
import AppIntents

struct LandmarkDetailView: View {
    var body: some View {
        VStack {
            // Your content
            SiriTipView(intent: OpenLandmarkIntent())
        }
    }
}
```

`SiriTipView` automatically shows the registered phrase for the intent. It respects the user's Siri settings and only appears when relevant.

### ShortcutsLink

Link to the Shortcuts app to show all your app's shortcuts:

```swift
ShortcutsLink()
```

This opens the Shortcuts app filtered to your app's available shortcuts.

## Confirmation flows

Ask the user to confirm before performing destructive or significant actions:

```swift
func perform() async throws -> some IntentResult {
    try await requestConfirmation(
        actionName: .delete,
        dialog: "Are you sure you want to delete \(collection.name)?"
    )
    // User confirmed — proceed
    store.delete(collection)
    return .result(dialog: "Deleted \(collection.name).")
}
```

### Confirmation with snippet view

```swift
try await requestConfirmation(
    actionName: .send,
    dialog: "Send this order?",
    view: OrderSummaryView(order: order)
)
```

## Siri and foreground behavior

When Siri runs your intent:
- By default, the app stays in the background — Siri shows dialog and snippets
- Set `supportedModes: .foreground` to open the app before performing
- Use dynamic/deferred modes when the decision depends on runtime state

For voice-only contexts (AirPods, CarPlay), Siri reads the dialog aloud. Always provide meaningful dialog even if you also show a view.

## Best practices

- **Keep phrases natural.** Write them as things a person would actually say.
- **Provide 2-5 phrase variations.** More variations increase the chance Siri matches the user's speech.
- **Always include dialog.** Even if your intent opens the app, Siri needs dialog for voice-only contexts.
- **Use `SiriTipView` strategically.** Place it where users are likely to want to repeat the action with Siri.
- **Test with Siri.** Run your phrases through Siri on device to verify recognition and behavior.
- **Localize phrases.** Phrases are `LocalizedStringResource` — provide translations for each supported locale.
