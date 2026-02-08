# Migration from SiriKit

Moving from legacy SiriKit (`INIntent`) to the App Intents framework.

## When to migrate

- **New intents**: Always use App Intents. SiriKit is not receiving new features.
- **Existing SiriKit intents**: Migrate when you update to iOS 26+ or when you need new capabilities (Spotlight, Apple Intelligence, interactive snippets).
- **Custom SiriKit domains**: Migrate to App Intents. Custom intents defined in `.intentdefinition` files should be rewritten as `AppIntent` structs.
- **System SiriKit domains** (messaging, payments, etc.): Check Apple's documentation — some system domains still use SiriKit, but Apple is progressively moving them to App Intents.

## Key differences

| Feature | SiriKit (INIntent) | App Intents |
|---|---|---|
| Definition | `.intentdefinition` file (Xcode editor) | Swift structs in code |
| Code generation | Xcode generates classes | No code generation — you write the structs |
| Parameters | Properties on generated classes | `@Parameter` property wrapper |
| Resolution | `resolve` methods on handler | `EntityQuery` and parameter resolution |
| Confirmation | `confirm` method on handler | `requestConfirmation()` in `perform()` |
| Execution | `handle` method on handler | `perform()` method on intent |
| Extension | Requires Intents Extension | Can run in app or extension |
| Shortcuts | Requires `INShortcut` donation | `AppShortcutsProvider` with phrases |
| Siri dialog | `INIntentResponse` with templates | `IntentDialog` returned from `perform()` |
| Spotlight | Not integrated | `IndexedEntity` + donation |
| Apple Intelligence | Limited | Full integration via `@AppIntent` macro |

## Migration steps

### Step 1: Identify your SiriKit intents

Review your `.intentdefinition` file and Intents Extension. List each intent and its parameters.

### Step 2: Create equivalent AppIntent structs

For each SiriKit intent, create a corresponding `AppIntent`:

**Before (SiriKit):**
```swift
// Generated from .intentdefinition
class OrderFoodIntent: INIntent {
    @NSManaged var restaurant: INObject?
    @NSManaged var menuItem: INObject?
    @NSManaged var quantity: NSNumber?
}

class OrderFoodIntentHandler: NSObject, OrderFoodIntentHandling {
    func resolveRestaurant(for intent: OrderFoodIntent) async -> INObjectResolutionResult { /* ... */ }
    func confirm(intent: OrderFoodIntent) async -> OrderFoodIntentResponse { /* ... */ }
    func handle(intent: OrderFoodIntent) async -> OrderFoodIntentResponse { /* ... */ }
}
```

**After (App Intents):**
```swift
struct OrderFoodIntent: AppIntent {
    static let title: LocalizedStringResource = "Order Food"

    @Parameter(title: "Restaurant")
    var restaurant: RestaurantEntity

    @Parameter(title: "Menu Item")
    var menuItem: MenuItemEntity

    @Parameter(title: "Quantity", default: 1)
    var quantity: Int

    static var parameterSummary: some ParameterSummary {
        Summary("Order \(\.$quantity) \(\.$menuItem) from \(\.$restaurant)")
    }

    func perform() async throws -> some IntentResult & ProvidesDialog {
        try await requestConfirmation(
            dialog: "Order \(quantity) \(menuItem.name) from \(restaurant.name)?"
        )
        try await OrderService.shared.place(
            restaurant: restaurant.id,
            item: menuItem.id,
            quantity: quantity
        )
        return .result(dialog: "Order placed!")
    }
}
```

### Step 3: Convert INObject types to AppEntity

Replace SiriKit's `INObject` with proper `AppEntity` types:

```swift
struct RestaurantEntity: AppEntity {
    var id: String
    @ComputedProperty var name: String { restaurant.name }
    let restaurant: Restaurant

    static let typeDisplayRepresentation = TypeDisplayRepresentation(name: "Restaurant")
    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(name)")
    }
    static let defaultQuery = RestaurantEntityQuery()
}
```

### Step 4: Convert resolution to queries

SiriKit's `resolve` methods become `EntityQuery` implementations:

**Before:**
```swift
func resolveRestaurant(for intent: OrderFoodIntent) async -> INObjectResolutionResult {
    if let restaurant = intent.restaurant {
        return .success(with: restaurant)
    }
    return .needsValue()
}
```

**After:**
```swift
struct RestaurantEntityQuery: EntityStringQuery {
    func entities(for identifiers: [String]) async throws -> [RestaurantEntity] {
        RestaurantStore.shared.restaurants(for: identifiers).map(RestaurantEntity.init)
    }

    func entities(matching string: String) async throws -> [RestaurantEntity] {
        RestaurantStore.shared.search(string).map(RestaurantEntity.init)
    }

    func suggestedEntities() async throws -> [RestaurantEntity] {
        RestaurantStore.shared.recentRestaurants.map(RestaurantEntity.init)
    }
}
```

### Step 5: Replace INShortcut donations with AppShortcutsProvider

**Before:**
```swift
let intent = OrderFoodIntent()
intent.restaurant = INObject(identifier: "pizza-place", display: "Pizza Place")
let shortcut = INShortcut(intent: intent)
INVoiceShortcutCenter.shared.setShortcutSuggestions([shortcut])
```

**After:**
```swift
struct MyAppShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: OrderFoodIntent(),
            phrases: [
                "Order from \(\.$restaurant) in \(.applicationName)",
                "Order food in \(.applicationName)"
            ],
            shortTitle: "Order Food",
            systemImageName: "fork.knife"
        )
    }
}
```

### Step 6: Remove Intents Extension (if possible)

If all your intents are migrated to App Intents and can run in-process, you can remove your Intents Extension. App Intents can run directly in your app process.

If you need background execution (e.g., for Shortcuts automations), keep an App Intents Extension instead.

## Coexistence

During migration, SiriKit and App Intents can coexist in the same app. You don't have to migrate everything at once:

- Keep existing SiriKit intents that work and aren't being changed
- Build new intents with App Intents
- Migrate SiriKit intents one at a time as you update features

## Key benefits after migration

- **No code generation.** Intents are pure Swift structs you control.
- **Spotlight integration.** Entities appear in Spotlight search.
- **Apple Intelligence.** Intents can use assistant schemas.
- **Interactive snippets.** Rich, interactive views in Siri and Shortcuts.
- **Swift Packages.** Share intent code across targets without frameworks.
- **Better type safety.** `AppEntity` and `AppEnum` vs untyped `INObject`.
