# SwiftUI Component Patterns

## View Composition

### @ViewBuilder
```swift
struct ConditionalContent<Content: View>: View {
    let isEmpty: Bool
    @ViewBuilder let content: () -> Content

    var body: some View {
        if isEmpty {
            ContentUnavailableView("No Items", systemImage: "tray")
        } else {
            content()
        }
    }
}
```

### ViewModifier
```swift
struct CardModifier: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(.regularMaterial)
            .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

extension View {
    func cardStyle() -> some View {
        modifier(CardModifier())
    }
}

// Usage
Text("Hello").cardStyle()
```

## Lists & Grids

### List
```swift
List {
    ForEach(items) { item in
        ItemRow(item: item)
    }
    .onDelete { indexSet in
        viewModel.deleteItems(at: indexSet)
    }
    .onMove { from, to in
        viewModel.moveItems(from: from, to: to)
    }
}
.listStyle(.insetGrouped)
.refreshable { await viewModel.refresh() }
```

### LazyVGrid
```swift
let columns = [GridItem(.adaptive(minimum: 150), spacing: 16)]

ScrollView {
    LazyVGrid(columns: columns, spacing: 16) {
        ForEach(items) { item in
            ItemCard(item: item)
        }
    }
    .padding()
}
```

### LazyHStack with Sections
```swift
ScrollView(.horizontal) {
    LazyHStack(spacing: 12) {
        ForEach(categories) { category in
            CategoryCard(category: category)
                .containerRelativeFrame(.horizontal, count: 3, spacing: 12)
        }
    }
    .scrollTargetLayout()
}
.scrollTargetBehavior(.viewAligned)
```

## Forms & Input

```swift
Form {
    Section("Profile") {
        TextField("Name", text: $viewModel.name)
            .textContentType(.name)

        TextField("Email", text: $viewModel.email)
            .textContentType(.emailAddress)
            .keyboardType(.emailAddress)
            .autocapitalization(.none)
    }

    Section("Preferences") {
        Picker("Theme", selection: $viewModel.theme) {
            ForEach(Theme.allCases, id: \.self) { theme in
                Text(theme.displayName).tag(theme)
            }
        }

        Toggle("Notifications", isOn: $viewModel.notificationsEnabled)

        DatePicker("Birthday", selection: $viewModel.birthday, displayedComponents: .date)
    }

    Section {
        Button("Save") { Task { await viewModel.save() } }
            .disabled(!viewModel.isValid)
    }
}
```

## Navigation

### NavigationStack with Type-Safe Routing
```swift
NavigationStack(path: $router.path) {
    List(items) { item in
        NavigationLink(value: item) {
            ItemRow(item: item)
        }
    }
    .navigationTitle("Items")
    .navigationDestination(for: Item.self) { item in
        ItemDetailView(item: item)
    }
}
```

### TabView
```swift
TabView(selection: $selectedTab) {
    Tab("Home", systemImage: "house", value: .home) {
        HomeView()
    }
    Tab("Search", systemImage: "magnifyingglass", value: .search) {
        SearchView()
    }
    Tab("Profile", systemImage: "person", value: .profile) {
        ProfileView()
    }
}
```

### Sheets & Alerts
```swift
.sheet(item: $viewModel.selectedItem) { item in
    ItemEditView(item: item)
        .presentationDetents([.medium, .large])
        .presentationDragIndicator(.visible)
}

.alert("Delete Item?", isPresented: $viewModel.showDeleteAlert) {
    Button("Delete", role: .destructive) {
        Task { await viewModel.confirmDelete() }
    }
    Button("Cancel", role: .cancel) { }
} message: {
    Text("This action cannot be undone.")
}
```

## Async Loading

### .task Pattern
```swift
struct UserProfileView: View {
    @Bindable var viewModel: ProfileViewModel

    var body: some View {
        Group {
            switch (viewModel.isLoading, viewModel.user, viewModel.errorMessage) {
            case (true, _, _):
                ProgressView("Loading...")
            case (_, _, let error?):
                ContentUnavailableView {
                    Label("Error", systemImage: "exclamationmark.triangle")
                } description: {
                    Text(error)
                } actions: {
                    Button("Retry") { Task { await viewModel.load() } }
                }
            case (_, let user?, _):
                ProfileContent(user: user)
            default:
                ContentUnavailableView("No Profile", systemImage: "person.slash")
            }
        }
        .task { await viewModel.load() }
    }
}
```

### AsyncImage
```swift
AsyncImage(url: user.avatarURL) { phase in
    switch phase {
    case .success(let image):
        image.resizable().scaledToFill()
    case .failure:
        Image(systemName: "person.circle.fill")
            .foregroundStyle(.secondary)
    case .empty:
        ProgressView()
    @unknown default:
        EmptyView()
    }
}
.frame(width: 60, height: 60)
.clipShape(Circle())
```

## Animation

### withAnimation
```swift
Button("Toggle") {
    withAnimation(.spring(duration: 0.3)) {
        isExpanded.toggle()
    }
}
```

### matchedGeometryEffect
```swift
@Namespace private var animation

// In source view
Image(item.image)
    .matchedGeometryEffect(id: item.id, in: animation)

// In destination view
Image(item.image)
    .matchedGeometryEffect(id: item.id, in: animation)
```

### Keyframe Animation
```swift
KeyframeAnimator(initialValue: AnimationValues(), trigger: triggerCount) { values in
    Image(systemName: "heart.fill")
        .scaleEffect(values.scale)
        .rotationEffect(values.rotation)
} keyframes: { _ in
    KeyframeTrack(\.scale) {
        SpringKeyframe(1.5, duration: 0.2)
        SpringKeyframe(1.0, duration: 0.2)
    }
    KeyframeTrack(\.rotation) {
        LinearKeyframe(.degrees(15), duration: 0.1)
        LinearKeyframe(.degrees(-15), duration: 0.1)
        LinearKeyframe(.zero, duration: 0.1)
    }
}
```

## ScrollView

### ScrollPosition
```swift
@State private var scrollPosition = ScrollPosition()

ScrollView {
    LazyVStack {
        ForEach(messages) { message in
            MessageBubble(message: message)
        }
    }
}
.scrollPosition($scrollPosition)
.onChange(of: messages.count) {
    scrollPosition.scrollTo(edge: .bottom)
}
```

### Scroll Target Behavior
```swift
ScrollView(.horizontal) {
    LazyHStack(spacing: 0) {
        ForEach(pages) { page in
            PageView(page: page)
                .containerRelativeFrame(.horizontal)
        }
    }
    .scrollTargetLayout()
}
.scrollTargetBehavior(.paging)
```

## Search

```swift
.searchable(text: $viewModel.searchText, prompt: "Search items")
.searchSuggestions {
    ForEach(viewModel.suggestions) { suggestion in
        Text(suggestion.name)
            .searchCompletion(suggestion.name)
    }
}
.searchScopes($viewModel.searchScope) {
    Text("All").tag(SearchScope.all)
    Text("Recent").tag(SearchScope.recent)
    Text("Favorites").tag(SearchScope.favorites)
}
```
