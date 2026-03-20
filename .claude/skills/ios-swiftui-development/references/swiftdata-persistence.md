# SwiftData Persistence Patterns

## @Model Macro

```swift
import SwiftData

@Model
class Item {
    var name: String
    var timestamp: Date
    var isCompleted: Bool

    @Attribute(.unique) var slug: String

    // Relationships
    @Relationship(deleteRule: .cascade)
    var tags: [Tag] = []

    @Relationship(inverse: \Category.items)
    var category: Category?

    // Exclude from persistence
    @Transient var isSelected = false

    init(name: String, slug: String) {
        self.name = name
        self.slug = slug
        self.timestamp = .now
        self.isCompleted = false
    }
}

@Model
class Tag {
    var name: String
    var items: [Item] = []

    init(name: String) {
        self.name = name
    }
}

@Model
class Category {
    var name: String
    var items: [Item] = []

    init(name: String) {
        self.name = name
    }
}
```

### Property Attributes

| Attribute | Purpose |
|-----------|---------|
| `@Attribute(.unique)` | Unique constraint (upsert on conflict) |
| `@Attribute(.spotlight)` | Index for Spotlight search |
| `@Attribute(.externalStorage)` | Store large data (images) externally |
| `@Attribute(.transformable(by:))` | Custom value transformer |
| `@Relationship(deleteRule:)` | `.cascade`, `.nullify`, `.deny`, `.noAction` |
| `@Transient` | Exclude from persistence |

## ModelContainer & ModelContext Setup

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: [Item.self, Tag.self, Category.self])
    }
}

// Custom configuration
@main
struct MyApp: App {
    let container: ModelContainer

    init() {
        let schema = Schema([Item.self, Tag.self, Category.self])
        let config = ModelConfiguration(
            "MyStore",
            schema: schema,
            isStoredInMemoryOnly: false,
            cloudKitDatabase: .automatic  // Enable CloudKit sync
        )
        container = try! ModelContainer(for: schema, configurations: [config])
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(container)
    }
}
```

## @Query in Views

```swift
struct ItemListView: View {
    // Basic query
    @Query var items: [Item]

    // Sorted
    @Query(sort: \Item.timestamp, order: .reverse)
    var recentItems: [Item]

    // Filtered + sorted
    @Query(
        filter: #Predicate<Item> { !$0.isCompleted },
        sort: [SortDescriptor(\Item.name)]
    )
    var activeItems: [Item]

    // Dynamic query with init
    @Query var filteredItems: [Item]

    init(category: Category) {
        let categoryName = category.name
        _filteredItems = Query(
            filter: #Predicate<Item> { item in
                item.category?.name == categoryName
            },
            sort: \Item.timestamp
        )
    }

    var body: some View {
        List(activeItems) { item in
            ItemRow(item: item)
        }
    }
}
```

## #Predicate Macro

```swift
// Simple comparison
#Predicate<Item> { $0.isCompleted == true }

// String matching
#Predicate<Item> { $0.name.localizedStandardContains("search") }

// Compound predicates
#Predicate<Item> { item in
    !item.isCompleted && item.name.count > 0
}

// Date comparison
let cutoff = Calendar.current.date(byAdding: .day, value: -7, to: .now)!
#Predicate<Item> { $0.timestamp > cutoff }

// Relationship traversal
#Predicate<Item> { $0.category?.name == "Work" }

// Build predicates dynamically
func itemPredicate(searchText: String, showCompleted: Bool) -> Predicate<Item> {
    if searchText.isEmpty {
        return #Predicate<Item> { item in
            showCompleted || !item.isCompleted
        }
    }
    return #Predicate<Item> { item in
        (showCompleted || !item.isCompleted) &&
        item.name.localizedStandardContains(searchText)
    }
}
```

## CRUD Operations

```swift
@MainActor @Observable
class ItemViewModel {
    private let modelContext: ModelContext

    init(modelContext: ModelContext) {
        self.modelContext = modelContext
    }

    // Create
    func addItem(name: String, slug: String) {
        let item = Item(name: name, slug: slug)
        modelContext.insert(item)
        try? modelContext.save()  // Explicit save (auto-save is default)
    }

    // Read (via FetchDescriptor)
    func fetchItems() throws -> [Item] {
        let descriptor = FetchDescriptor<Item>(
            predicate: #Predicate { !$0.isCompleted },
            sortBy: [SortDescriptor(\Item.timestamp, order: .reverse)]
        )
        return try modelContext.fetch(descriptor)
    }

    // Read with pagination
    func fetchPage(offset: Int, limit: Int) throws -> [Item] {
        var descriptor = FetchDescriptor<Item>(
            sortBy: [SortDescriptor(\Item.timestamp, order: .reverse)]
        )
        descriptor.fetchOffset = offset
        descriptor.fetchLimit = limit
        return try modelContext.fetch(descriptor)
    }

    // Update (just modify properties -- SwiftData tracks changes)
    func toggleCompleted(_ item: Item) {
        item.isCompleted.toggle()
        // Auto-saved on next run loop
    }

    // Delete
    func deleteItem(_ item: Item) {
        modelContext.delete(item)
    }

    // Batch delete
    func deleteCompleted() throws {
        try modelContext.delete(model: Item.self, where: #Predicate { $0.isCompleted })
    }
}
```

## Migration

```swift
// Version 1
enum SchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)
    static var models: [any PersistentModel.Type] { [Item.self] }

    @Model class Item {
        var name: String
        var timestamp: Date
        init(name: String) {
            self.name = name
            self.timestamp = .now
        }
    }
}

// Version 2 -- added isCompleted
enum SchemaV2: VersionedSchema {
    static var versionIdentifier = Schema.Version(2, 0, 0)
    static var models: [any PersistentModel.Type] { [Item.self] }

    @Model class Item {
        var name: String
        var timestamp: Date
        var isCompleted: Bool
        init(name: String) {
            self.name = name
            self.timestamp = .now
            self.isCompleted = false
        }
    }
}

// Migration plan
enum ItemMigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] {
        [SchemaV1.self, SchemaV2.self]
    }

    static var stages: [MigrationStage] {
        [migrateV1toV2]
    }

    static let migrateV1toV2 = MigrationStage.lightweight(
        fromVersion: SchemaV1.self,
        toVersion: SchemaV2.self
    )
}

// Use in container
let container = try ModelContainer(
    for: SchemaV2.Item.self,
    migrationPlan: ItemMigrationPlan.self
)
```

## Testing with In-Memory Containers

```swift
import Testing
import SwiftData

struct ItemViewModelTests {
    @Test("adds item to store")
    func addItem() throws {
        let config = ModelConfiguration(isStoredInMemoryOnly: true)
        let container = try ModelContainer(for: Item.self, configurations: config)
        let context = container.mainContext

        let vm = ItemViewModel(modelContext: context)
        vm.addItem(name: "Test", slug: "test")

        let items = try context.fetch(FetchDescriptor<Item>())
        #expect(items.count == 1)
        #expect(items.first?.name == "Test")
    }

    @Test("deletes completed items")
    func deleteCompleted() throws {
        let config = ModelConfiguration(isStoredInMemoryOnly: true)
        let container = try ModelContainer(for: Item.self, configurations: config)
        let context = container.mainContext

        let item1 = Item(name: "Done", slug: "done")
        item1.isCompleted = true
        let item2 = Item(name: "Active", slug: "active")

        context.insert(item1)
        context.insert(item2)

        let vm = ItemViewModel(modelContext: context)
        try vm.deleteCompleted()

        let items = try context.fetch(FetchDescriptor<Item>())
        #expect(items.count == 1)
        #expect(items.first?.name == "Active")
    }
}
```

## When to Use What

| Need | Use |
|------|-----|
| Structured app data, relationships, queries | **SwiftData** |
| Small key-value preferences | **UserDefaults** |
| Sensitive credentials, tokens | **Keychain** (via Security framework) |
| Files (documents, exports) | **FileManager** + app sandbox |
| Temporary cache | **URLCache** or **NSCache** |
| Sync across devices | **SwiftData + CloudKit** or **NSUbiquitousKeyValueStore** |
| Scene-specific state restoration | **@SceneStorage** |
| App-wide lightweight state | **@AppStorage** (wraps UserDefaults) |
