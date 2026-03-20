# MVVM with @Observable Patterns

## @Observable ViewModel Pattern

```swift
import Foundation
import Observation

@MainActor @Observable
class ItemListViewModel {
    // MARK: - Published State
    var items: [Item] = []
    var isLoading = false
    var errorMessage: String?
    var searchText = ""

    // MARK: - Computed Properties
    var filteredItems: [Item] {
        if searchText.isEmpty { return items }
        return items.filter { $0.name.localizedCaseInsensitiveContains(searchText) }
    }

    var isEmpty: Bool { items.isEmpty && !isLoading }

    // MARK: - Dependencies
    private let repository: ItemRepositoryProtocol

    init(repository: ItemRepositoryProtocol = ItemRepository()) {
        self.repository = repository
    }

    // MARK: - Actions
    func loadItems() async {
        isLoading = true
        defer { isLoading = false }
        do {
            items = try await repository.fetchAll()
            errorMessage = nil
        } catch {
            errorMessage = error.localizedDescription
        }
    }

    func deleteItem(_ item: Item) async {
        do {
            try await repository.delete(id: item.id)
            items.removeAll { $0.id == item.id }
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

## Data Flow

```
View (SwiftUI)
  |-- reads --> ViewModel (@Observable)
  |-- calls --> ViewModel.action()
                    |-- calls --> Service (protocol)
                    |                |-- async --> Network / DB / System
                    |                |<-- returns data or throws
                    |<-- updates state (triggers view refresh)
  |<-- SwiftUI re-renders on state change
```

## Property Wrapper Decision Tree

| Question | Answer | Use |
|----------|--------|-----|
| Is this view-local transient state? | Yes | `@State` |
| Does a parent own this value and child needs read/write? | Yes | `@Binding` |
| Do I need to mutate an @Observable object's properties in a child? | Yes | `@Bindable` |
| Is this shared app-wide (DI, settings, theme)? | Yes | `@Environment` |
| Does this view create and own the ViewModel? | Yes | `@State var vm = ViewModel()` |
| Does this view receive a ViewModel from parent? | Yes | `@Bindable var vm: ViewModel` or `let vm: ViewModel` |

### Examples
```swift
// View OWNS the ViewModel
struct ItemListScreen: View {
    @State private var viewModel = ItemListViewModel()

    var body: some View {
        ItemListContent(viewModel: viewModel)
    }
}

// View RECEIVES the ViewModel (needs to bind to its properties)
struct ItemListContent: View {
    @Bindable var viewModel: ItemListViewModel

    var body: some View {
        List(viewModel.filteredItems) { item in
            Text(item.name)
        }
        .searchable(text: $viewModel.searchText)
    }
}

// View-local state only
struct ExpandableCard: View {
    let title: String
    @State private var isExpanded = false

    var body: some View {
        DisclosureGroup(title, isExpanded: $isExpanded) {
            Text("Details here")
        }
    }
}
```

## ViewModel Composition

Break large ViewModels into focused ones:

```swift
// Instead of one massive ViewModel:
@MainActor @Observable
class ProfileViewModel {
    var user: User?
    var posts: [Post] = []
    var followers: [User] = []
    // 20+ properties, 500+ lines...
}

// Compose smaller, focused ViewModels:
@MainActor @Observable
class ProfileHeaderViewModel {
    var user: User?
    private let userService: UserServiceProtocol

    func loadUser(id: UUID) async { ... }
}

@MainActor @Observable
class PostsViewModel {
    var posts: [Post] = []
    var isLoading = false
    private let postService: PostServiceProtocol

    func loadPosts(for userId: UUID) async { ... }
}

// Parent view composes them
struct ProfileScreen: View {
    @State private var headerVM = ProfileHeaderViewModel()
    @State private var postsVM = PostsViewModel()
    let userId: UUID

    var body: some View {
        ScrollView {
            ProfileHeader(viewModel: headerVM)
            PostsList(viewModel: postsVM)
        }
        .task {
            async let _ = headerVM.loadUser(id: userId)
            async let _ = postsVM.loadPosts(for: userId)
        }
    }
}
```

## Dependency Injection via Environment

```swift
// Define environment key
struct ItemRepositoryKey: EnvironmentKey {
    static let defaultValue: ItemRepositoryProtocol = ItemRepository()
}

extension EnvironmentValues {
    var itemRepository: ItemRepositoryProtocol {
        get { self[ItemRepositoryKey.self] }
        set { self[ItemRepositoryKey.self] = newValue }
    }
}

// Inject at app root
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.itemRepository, ItemRepository())
        }
    }
}

// Read in ViewModel or View
struct ItemListScreen: View {
    @Environment(\.itemRepository) private var repository

    var body: some View {
        ItemListView(viewModel: ItemListViewModel(repository: repository))
    }
}
```

## Navigation with MVVM

```swift
@MainActor @Observable
class AppRouter {
    var path = NavigationPath()

    func navigate(to destination: AppDestination) {
        path.append(destination)
    }

    func popToRoot() {
        path = NavigationPath()
    }
}

enum AppDestination: Hashable {
    case itemDetail(Item)
    case settings
    case profile(userId: UUID)
}

struct RootView: View {
    @State private var router = AppRouter()

    var body: some View {
        NavigationStack(path: $router.path) {
            HomeView()
                .navigationDestination(for: AppDestination.self) { dest in
                    switch dest {
                    case .itemDetail(let item): ItemDetailView(item: item)
                    case .settings: SettingsView()
                    case .profile(let id): ProfileView(userId: id)
                    }
                }
        }
        .environment(router)
    }
}
```

## Testing @Observable ViewModels

```swift
import Testing

struct ItemListViewModelTests {
    @Test("loads items from repository")
    func loadItems() async {
        let mock = MockItemRepository(items: [Item(id: UUID(), name: "Test")])
        let vm = ItemListViewModel(repository: mock)

        await vm.loadItems()

        #expect(vm.items.count == 1)
        #expect(vm.isLoading == false)
        #expect(vm.errorMessage == nil)
    }

    @Test("filters items by search text")
    func filterItems() async {
        let items = [
            Item(id: UUID(), name: "Apple"),
            Item(id: UUID(), name: "Banana")
        ]
        let vm = ItemListViewModel(repository: MockItemRepository(items: items))
        await vm.loadItems()

        vm.searchText = "app"

        #expect(vm.filteredItems.count == 1)
        #expect(vm.filteredItems.first?.name == "Apple")
    }

    @Test("handles repository error")
    func errorHandling() async {
        let mock = MockItemRepository(error: TestError.networkFailure)
        let vm = ItemListViewModel(repository: mock)

        await vm.loadItems()

        #expect(vm.items.isEmpty)
        #expect(vm.errorMessage != nil)
    }
}
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Business logic in Views | Untestable, violates MVVM | Move to ViewModel |
| Massive ViewModel (500+ lines) | Hard to maintain/test | Compose smaller VMs |
| ViewModel imports SwiftUI | Couples logic to UI framework | Use `import Foundation` |
| Direct service calls in Views | No abstraction, hard to test | ViewModel calls service |
| Skipping protocol for services | Can't mock in tests | Always define protocol |
| Using ObservableObject in new code | Legacy, more boilerplate | Use @Observable |
| @Published + @ObservableObject | Unnecessary property wrappers | @Observable handles it |
| Force-unwrapping ViewModel state | Crash risk | Use optionals + guard |
