# App Intents & Widgets

## App Intents

### Basic AppIntent

```swift
import AppIntents

struct AddItemIntent: AppIntent {
    static var title: LocalizedStringResource = "Add Item"
    static var description: IntentDescription = "Creates a new item in the app"

    @Parameter(title: "Name")
    var name: String

    @Parameter(title: "Category", default: "General")
    var category: String

    static var parameterSummary: some ParameterSummary {
        Summary("Add \(\.$name) to \(\.$category)")
    }

    func perform() async throws -> some IntentResult & ReturnsValue<String> {
        let service = ItemService()
        let item = try await service.create(name: name, category: category)
        return .result(value: item.name)
    }
}
```

### AppEntity

```swift
struct ItemEntity: AppEntity {
    static var typeDisplayRepresentation = TypeDisplayRepresentation(name: "Item")

    static var defaultQuery = ItemQuery()

    var id: UUID
    var name: String
    var category: String

    var displayRepresentation: DisplayRepresentation {
        DisplayRepresentation(title: "\(name)", subtitle: "\(category)")
    }
}

struct ItemQuery: EntityQuery {
    func entities(for identifiers: [UUID]) async throws -> [ItemEntity] {
        let service = ItemService()
        return try await service.fetch(ids: identifiers).map { item in
            ItemEntity(id: item.id, name: item.name, category: item.category)
        }
    }

    func suggestedEntities() async throws -> [ItemEntity] {
        let service = ItemService()
        return try await service.fetchRecent().map { item in
            ItemEntity(id: item.id, name: item.name, category: item.category)
        }
    }
}
```

### AppShortcutsProvider (Siri / Shortcuts)

```swift
struct AppShortcuts: AppShortcutsProvider {
    static var appShortcuts: [AppShortcut] {
        AppShortcut(
            intent: AddItemIntent(),
            phrases: [
                "Add \(\.$name) to \(.applicationName)",
                "Create item in \(.applicationName)"
            ],
            shortTitle: "Add Item",
            systemImageName: "plus.circle"
        )
    }
}
```

## WidgetKit

### TimelineProvider

```swift
import WidgetKit
import SwiftUI

struct ItemEntry: TimelineEntry {
    let date: Date
    let items: [Item]
}

struct ItemTimelineProvider: TimelineProvider {
    func placeholder(in context: Context) -> ItemEntry {
        ItemEntry(date: .now, items: [.sample])
    }

    func getSnapshot(in context: Context, completion: @escaping (ItemEntry) -> Void) {
        let entry = ItemEntry(date: .now, items: [.sample])
        completion(entry)
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<ItemEntry>) -> Void) {
        Task {
            let items = try? await ItemService().fetchRecent()
            let entry = ItemEntry(date: .now, items: items ?? [])
            let nextUpdate = Calendar.current.date(byAdding: .hour, value: 1, to: .now)!
            let timeline = Timeline(entries: [entry], policy: .after(nextUpdate))
            completion(timeline)
        }
    }
}
```

### Widget Definition

```swift
struct ItemWidget: Widget {
    let kind = "ItemWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: ItemTimelineProvider()) { entry in
            ItemWidgetView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("Recent Items")
        .description("Shows your most recent items.")
        .supportedFamilies([.systemSmall, .systemMedium, .systemLarge])
    }
}

struct ItemWidgetView: View {
    let entry: ItemEntry

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text("Recent Items")
                .font(.headline)
            ForEach(entry.items.prefix(3)) { item in
                Label(item.name, systemImage: "circle")
                    .font(.caption)
            }
        }
    }
}
```

### Configurable Widget (AppIntentTimelineProvider)

```swift
struct ConfigurableItemWidget: Widget {
    var body: some WidgetConfiguration {
        AppIntentConfiguration(
            kind: "ConfigurableItemWidget",
            intent: SelectCategoryIntent.self,
            provider: ConfigurableProvider()
        ) { entry in
            ItemWidgetView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("Items by Category")
        .description("Shows items from a chosen category.")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}

struct SelectCategoryIntent: WidgetConfigurationIntent {
    static var title: LocalizedStringResource = "Select Category"

    @Parameter(title: "Category")
    var category: CategoryEntity?
}

struct ConfigurableProvider: AppIntentTimelineProvider {
    func placeholder(in context: Context) -> ItemEntry {
        ItemEntry(date: .now, items: [.sample])
    }

    func snapshot(for configuration: SelectCategoryIntent, in context: Context) async -> ItemEntry {
        ItemEntry(date: .now, items: [.sample])
    }

    func timeline(for configuration: SelectCategoryIntent, in context: Context) async -> Timeline<ItemEntry> {
        let categoryName = configuration.category?.name
        let items = try? await ItemService().fetch(category: categoryName)
        let entry = ItemEntry(date: .now, items: items ?? [])
        let nextUpdate = Calendar.current.date(byAdding: .hour, value: 1, to: .now)!
        return Timeline(entries: [entry], policy: .after(nextUpdate))
    }
}
```

## Live Activities

### ActivityAttributes

```swift
import ActivityKit

struct DeliveryAttributes: ActivityAttributes {
    public struct ContentState: Codable, Hashable {
        var status: String
        var estimatedArrival: Date
        var driverName: String
    }

    var orderNumber: String
    var restaurantName: String
}
```

### Live Activity View

```swift
struct DeliveryLiveActivity: Widget {
    var body: some WidgetConfiguration {
        ActivityConfiguration(for: DeliveryAttributes.self) { context in
            // Lock Screen / Banner
            VStack {
                HStack {
                    Text(context.attributes.restaurantName)
                        .font(.headline)
                    Spacer()
                    Text(context.state.status)
                        .foregroundStyle(.secondary)
                }
                ProgressView(value: 0.6)
                Text("ETA: \(context.state.estimatedArrival, style: .relative)")
                    .font(.caption)
            }
            .padding()
        } dynamicIsland: { context in
            DynamicIsland {
                // Expanded
                DynamicIslandExpandedRegion(.leading) {
                    Image(systemName: "bag.fill")
                }
                DynamicIslandExpandedRegion(.trailing) {
                    Text(context.state.estimatedArrival, style: .timer)
                }
                DynamicIslandExpandedRegion(.bottom) {
                    Text(context.state.status)
                }
            } compactLeading: {
                Image(systemName: "bag.fill")
            } compactTrailing: {
                Text(context.state.estimatedArrival, style: .timer)
            } minimal: {
                Image(systemName: "bag.fill")
            }
        }
    }
}
```

### Starting / Updating / Ending

```swift
// Start
func startDeliveryActivity(order: Order) throws -> Activity<DeliveryAttributes> {
    let attributes = DeliveryAttributes(
        orderNumber: order.number,
        restaurantName: order.restaurant
    )
    let state = DeliveryAttributes.ContentState(
        status: "Preparing",
        estimatedArrival: order.eta,
        driverName: ""
    )
    return try Activity.request(
        attributes: attributes,
        content: .init(state: state, staleDate: nil)
    )
}

// Update
func updateActivity(_ activity: Activity<DeliveryAttributes>, status: String, eta: Date) async {
    let state = DeliveryAttributes.ContentState(
        status: status,
        estimatedArrival: eta,
        driverName: "Alex"
    )
    await activity.update(.init(state: state, staleDate: nil))
}

// End
func endActivity(_ activity: Activity<DeliveryAttributes>) async {
    let finalState = DeliveryAttributes.ContentState(
        status: "Delivered",
        estimatedArrival: .now,
        driverName: "Alex"
    )
    await activity.end(.init(state: finalState, staleDate: nil), dismissalPolicy: .after(.now + 3600))
}
```

## Control Center Widgets (iOS 26)

```swift
struct ToggleItemControl: ControlWidget {
    var body: some ControlWidgetConfiguration {
        StaticControlConfiguration(kind: "ToggleItem") {
            ControlWidgetToggle(
                "Quick Add",
                isOn: QuickAddManager.shared.isEnabled,
                action: ToggleQuickAddIntent()
            ) { isOn in
                Label(isOn ? "On" : "Off", systemImage: isOn ? "plus.circle.fill" : "plus.circle")
            }
        }
        .displayName("Quick Add")
    }
}

struct ToggleQuickAddIntent: SetValueIntent {
    static var title: LocalizedStringResource = "Toggle Quick Add"

    @Parameter(title: "Enabled")
    var value: Bool

    func perform() async throws -> some IntentResult {
        QuickAddManager.shared.isEnabled = value
        return .result()
    }
}
```

## Spotlight Integration

```swift
import CoreSpotlight

// Index via CSSearchableItem
func indexItem(_ item: Item) {
    let attributes = CSSearchableItemAttributeSet(contentType: .content)
    attributes.title = item.name
    attributes.contentDescription = item.description
    attributes.keywords = item.tags.map(\.name)

    let searchableItem = CSSearchableItem(
        uniqueIdentifier: item.id.uuidString,
        domainIdentifier: "com.example.items",
        attributeSet: attributes
    )

    CSSearchableIndex.default().indexSearchableItems([searchableItem])
}

// Handle Spotlight tap
struct ContentView: View {
    var body: some View {
        NavigationStack { ... }
            .onContinueUserActivity(CSSearchableItemActionType) { activity in
                guard let id = activity.userInfo?[CSSearchableItemActivityIdentifier] as? String,
                      let uuid = UUID(uuidString: id) else { return }
                // Navigate to item
            }
    }
}

// Or use App Intents for automatic indexing
struct ViewItemIntent: AppIntent {
    static var title: LocalizedStringResource = "View Item"

    @Parameter(title: "Item")
    var item: ItemEntity

    static var openAppWhenRun = true

    func perform() async throws -> some IntentResult {
        // Navigate to item
        return .result()
    }
}
```

## Widget Bundle

```swift
@main
struct AppWidgets: WidgetBundle {
    var body: some Widget {
        ItemWidget()
        ConfigurableItemWidget()
        DeliveryLiveActivity()
        #if os(iOS)
        ToggleItemControl()
        #endif
    }
}
```
