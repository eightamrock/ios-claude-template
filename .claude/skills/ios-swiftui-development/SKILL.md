---
name: ios-swiftui-development
description: Use when building Apple platform apps with SwiftUI, implementing MVVM architecture with @Observable, designing UI per Apple HIG, writing Swift with modern concurrency, or targeting iOS 26+ with liquid glass and native components
---

# Apple Platform SwiftUI Development

## Overview

Build iOS 26+, macOS 26+, watchOS 12+, and visionOS 2+ apps using SwiftUI with MVVM architecture (@Observable), modern Swift concurrency (async/await, actors), and Apple Human Interface Guidelines compliance. Prefer native components, SPM-only dependencies, and Apple frameworks over third-party alternatives.

## When to Use

- Building new apps or features with SwiftUI on any Apple platform
- Implementing MVVM architecture with @Observable
- Designing UI that needs HIG compliance
- Working with iOS 26 liquid glass and native components
- Writing Swift with modern concurrency patterns
- Setting up SwiftUI navigation, forms, or data flow
- Creating or refactoring ViewModels
- Working with SwiftData persistence
- Implementing App Intents, widgets, or Live Activities
- Building networking layers with URLSession
- Multi-platform development (iOS, macOS, watchOS, visionOS)

## When NOT to Use

- UIKit-only projects (no SwiftUI)
- Backend Swift (Vapor, server-side)
- Objective-C codebases
- Cross-platform frameworks (Flutter, React Native)

## Architecture: MVVM with @Observable

### Structure
```
Models/       -- Pure data types (struct, enum). Codable, Identifiable. No UI imports.
ViewModels/   -- @Observable @MainActor classes. All business logic. Import Foundation only.
Views/        -- Thin SwiftUI structs. Layout only. Bind to ViewModels.
Services/     -- Protocol-defined. Networking, persistence, system APIs. Injected into VMs.
```

### Rules
1. ViewModels NEVER import SwiftUI
2. Views contain ZERO business logic
3. Services always behind protocols (for testability)
4. Models are value types (structs/enums)
5. Use `@Observable` (not `ObservableObject`)

### Quick Pattern
```swift
// Model
struct Item: Identifiable, Codable {
    let id: UUID
    var name: String
}

// ViewModel
@MainActor @Observable
class ItemListViewModel {
    var items: [Item] = []
    var isLoading = false
    private let service: ItemServiceProtocol

    init(service: ItemServiceProtocol = ItemService()) {
        self.service = service
    }

    func load() async {
        isLoading = true
        defer { isLoading = false }
        items = (try? await service.fetchAll()) ?? []
    }
}

// View
struct ItemListView: View {
    @Bindable var viewModel: ItemListViewModel

    var body: some View {
        List(viewModel.items) { item in
            Text(item.name)
        }
        .task { await viewModel.load() }
    }
}
```

**Deep dive:** See `references/mvvm-observable-patterns.md` for composition, DI via Environment, navigation patterns, testing, and anti-patterns.

## SwiftUI Data Flow Decision Table

| Scenario | Property Wrapper |
|----------|-----------------|
| View-local transient state | `@State` |
| Parent passes value, child needs read/write | `@Binding` |
| Child needs to mutate @Observable object properties | `@Bindable` |
| Shared data through view hierarchy | `@Environment` |
| View creates and owns its ViewModel | `@State var vm = ViewModel()` |
| View receives ViewModel from parent | `@Bindable var vm` or `let vm` |

### Lifecycle
- `.task { }` for async work on appear (auto-cancels on disappear)
- `.task(id: value)` to re-trigger when value changes
- Never use `onAppear` for async work
- `#Preview` macro for all views

### Navigation
- `NavigationStack` + `NavigationPath` for programmatic navigation
- `.navigationDestination(for:)` for type-safe routing
- `TabView` for top-level (max 5 tabs)
- `sheet()` for modal tasks, `fullScreenCover()` for immersive

**Deep dive:** See `references/swiftui-component-patterns.md` for lists, grids, forms, search, animation, scroll views, and async loading patterns.

## iOS 26 / Liquid Glass

### Automatic (no code needed)
TabView, NavigationStack bars, toolbars, system alerts, sheets, popovers -- all get glass automatically.

### Manual
```swift
.glassEffect(.regular)           // Custom floating elements
GlassEffectContainer { ... }     // Group glass elements
MeshGradient(width:height:points:colors:)  // Rich backgrounds behind glass
```

### Key Rules
- Do NOT override system glass on built-in components
- Only apply `.glassEffect` to floating/overlay elements
- Never use glass on content backgrounds, cards, or list rows
- Test both light and dark mode
- Respect `accessibilityReduceTransparency`

**Deep dive:** See `references/liquid-glass-native-components.md` for mesh gradients, native components catalog, dark mode, and performance notes.

## Code Quality Standards

### Naming

| Element | Convention | Example |
|---------|-----------|---------|
| Types | UpperCamelCase | `UserProfile` |
| Functions/properties | lowerCamelCase | `fetchUser()`, `isLoading` |
| Enum cases | lowerCamelCase | `.networkError` |
| Booleans | is/has/should prefix | `isValid`, `hasContent` |
| Protocols (capability) | -able/-ible | `Loadable` |
| Protocols (role) | Noun | `DataSource` |

### Concurrency Quick Reference

| Need | Use |
|------|-----|
| Async operation | `async/await` |
| UI-updating code | `@MainActor` |
| Shared mutable state | `actor` |
| Parallel independent work | `async let` or `TaskGroup` |
| Cross-isolation types | `Sendable` conformance |

### General Rules
- `guard` for early exits
- Never force-unwrap (`!`)
- Prefer value types over reference types
- Use access control intentionally (`private` by default)
- One primary type per file

**Deep dive:** See `references/swift-code-quality.md` for protocol-oriented design, error handling, concurrency patterns, optionals, and memory management.

## HIG Compliance (Top 10 Critical Rules)

1. **44pt minimum touch targets** for all interactive elements
2. **Dynamic Type** for all text -- never hardcode font sizes
3. **Dark Mode** support -- use system colors, test both modes
4. **SF Symbols** for icons (match weight to text, correct rendering mode)
5. **Safe areas** -- never clip content under system UI
6. **Accessibility labels** on all interactive elements and meaningful images
7. **Content states** -- every data view handles loading, empty, error, loaded
8. **System components** over custom (DatePicker, PhotosPicker, ShareLink, etc.)
9. **Haptic feedback** for meaningful interactions (not every tap)
10. **Contrast ratio** 4.5:1 for text, never convey meaning by color alone

**Full checklist:** See `references/hig-compliance-checklist.md` for comprehensive layout, typography, color, navigation, interaction, accessibility, and platform convention checks.

## Xcode MCP Tools

When Xcode is running, prefer MCP tools over CLI equivalents:

| Task | Tool |
|------|------|
| Build project | `mcp__xcode__BuildProject` |
| Run all tests | `mcp__xcode__RunAllTests` |
| Run specific tests | `mcp__xcode__RunSomeTests` |
| Get test list | `mcp__xcode__GetTestList` |
| Validate UI changes | `mcp__xcode__RenderPreview` |
| Check build errors | `mcp__xcode__GetBuildLog` + `XcodeListNavigatorIssues` |
| File diagnostics | `mcp__xcode__XcodeRefreshCodeIssuesInFile` |
| Search docs | `mcp__xcode__DocumentationSearch` |
| Read/write files | `mcp__xcode__XcodeRead` / `XcodeWrite` / `XcodeUpdate` |
| Search project files | `mcp__xcode__XcodeGrep` / `XcodeGlob` |
| File operations | `mcp__xcode__XcodeLS` / `XcodeMV` / `XcodeRM` / `XcodeMakeDir` |

Use `RenderPreview` after UI changes to validate visually. Fall back to `xcodebuild` for CI/headless builds.

## LSP Enforcement

The `swift-lsp` plugin MUST be used:
- Run diagnostics after every file edit
- Fix all errors and warnings before moving on
- Use go-to-definition to understand types before modifying
- Use find-references before renaming or removing symbols
- Resolve all warnings before considering work complete

## Testing (Swift Testing Framework)

```swift
import Testing

struct ItemViewModelTests {
    @Test("loads items successfully")
    func loadItems() async {
        let mock = MockService(items: [.sample])
        let vm = ItemListViewModel(service: mock)
        await vm.load()
        #expect(vm.items.count == 1)
        #expect(!vm.isLoading)
    }
}
```

### Rules
- Use Swift Testing (`@Test`, `#expect`) not XCTest for new tests
- Test ViewModels thoroughly (all public methods, state transitions)
- Protocol mocks for all service dependencies
- Test error paths, not just happy paths
- Use `#Preview` for visual validation of views

## Build Commands

```bash
# iOS
xcodebuild -project "Project.xcodeproj" -scheme "Project" -sdk iphonesimulator build

# macOS
xcodebuild -project "Project.xcodeproj" -scheme "Project" -destination 'platform=macOS' build

# watchOS
xcodebuild -project "Project.xcodeproj" -scheme "ProjectWatch" \
  -destination 'platform=watchOS Simulator,name=Apple Watch Series 10 (46mm)' build

# visionOS
xcodebuild -project "Project.xcodeproj" -scheme "Project" \
  -destination 'platform=visionOS Simulator,name=Apple Vision Pro' build

# Test (iOS)
xcodebuild -project "Project.xcodeproj" -scheme "Project" -sdk iphonesimulator \
  -destination 'platform=iOS Simulator,name=iPhone 16 Pro' test

# SPM
swift build && swift test
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `ObservableObject` + `@Published` | Use `@Observable` (iOS 17+) |
| Business logic in Views | Move to ViewModel |
| `onAppear` for async work | Use `.task { }` |
| Force-unwrapping optionals | `guard let`, `if let`, `??` |
| Custom UI for system features | Use native: `DatePicker`, `PhotosPicker`, `ShareLink` |
| Hardcoded colors/fonts | System colors + Dynamic Type |
| Massive ViewModel (500+ lines) | Compose smaller focused VMs |
| Missing accessibility labels | Add `.accessibilityLabel()` to all controls |
| Glass effect on content backgrounds | Glass is for floating/overlay elements only |
| Completion handlers in new code | Use `async/await` |
| Skipping LSP diagnostics | Always run diagnostics, fix all warnings |
| CocoaPods or Carthage | SPM only |
| Missing `@MainActor` on ViewModel | Always add `@MainActor` to ViewModels |

## Supporting Reference Files

| File | Contents |
|------|----------|
| `references/mvvm-observable-patterns.md` | Full MVVM pattern, composition, DI, navigation, testing, anti-patterns |
| `references/swiftui-component-patterns.md` | Lists, grids, forms, navigation, async loading, animation, search |
| `references/swift-code-quality.md` | Naming, protocols, errors, concurrency, optionals, memory |
| `references/hig-compliance-checklist.md` | Full HIG checklist: layout, typography, color, accessibility |
| `references/liquid-glass-native-components.md` | Glass APIs, mesh gradients, native components catalog, performance |
| `references/swiftdata-persistence.md` | @Model, ModelContainer, @Query, #Predicate, CRUD, migrations, testing |
| `references/app-intents-widgets.md` | AppIntent, AppEntity, WidgetKit, Live Activities, Control Center widgets |
| `references/networking-api-patterns.md` | URLSession async/await, API client pattern, auth, pagination, testing |
| `references/multiplatform-patterns.md` | Platform conditionals, macOS/watchOS/visionOS patterns, shared code |
| `references/notifications-background-tasks.md` | Push notifications, local notifications, BGTaskScheduler, background downloads |
