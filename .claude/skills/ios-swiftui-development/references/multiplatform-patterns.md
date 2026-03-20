# Multi-Platform Patterns

## Platform Conditionals

```swift
#if os(iOS)
// iOS-specific code
#elseif os(macOS)
// macOS-specific code
#elseif os(watchOS)
// watchOS-specific code
#elseif os(visionOS)
// visionOS-specific code
#endif

// Combine platforms
#if os(iOS) || os(macOS)
// Shared iOS + macOS code
#endif

// Check for specific features
#if canImport(UIKit)
import UIKit
#elseif canImport(AppKit)
import AppKit
#endif
```

## Platform-Adaptive Views

```swift
struct AdaptiveDetailView: View {
    let item: Item

    var body: some View {
        #if os(iOS)
        NavigationStack {
            content
                .navigationBarTitleDisplayMode(.inline)
        }
        #elseif os(macOS)
        content
            .frame(minWidth: 300)
        #endif
    }

    private var content: some View {
        // Shared view code
        VStack { Text(item.name) }
    }
}
```

## macOS Patterns

### NavigationSplitView (Sidebar)
```swift
struct MacAppView: View {
    @State private var selectedCategory: Category?
    @State private var selectedItem: Item?

    var body: some View {
        NavigationSplitView {
            List(categories, selection: $selectedCategory) { category in
                Label(category.name, systemImage: category.icon)
            }
            .navigationSplitViewColumnWidth(min: 180, ideal: 220)
        } content: {
            if let category = selectedCategory {
                ItemListView(category: category, selection: $selectedItem)
            }
        } detail: {
            if let item = selectedItem {
                ItemDetailView(item: item)
            } else {
                ContentUnavailableView("Select an Item", systemImage: "doc")
            }
        }
    }
}
```

### Settings Scene
```swift
@main
struct MyMacApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }

        #if os(macOS)
        Settings {
            SettingsView()
        }
        #endif
    }
}

struct SettingsView: View {
    var body: some View {
        TabView {
            GeneralSettings()
                .tabItem { Label("General", systemImage: "gear") }
            AccountSettings()
                .tabItem { Label("Account", systemImage: "person") }
        }
        .frame(width: 450, height: 300)
    }
}
```

### MenuBarExtra
```swift
@main
struct MenuBarApp: App {
    var body: some Scene {
        #if os(macOS)
        MenuBarExtra("Status", systemImage: "cloud.fill") {
            StatusMenuView()
        }
        .menuBarExtraStyle(.window)  // or .menu for simple menu
        #endif
    }
}
```

### Window Management
```swift
@main
struct MultiWindowApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }

        #if os(macOS)
        Window("Activity", id: "activity") {
            ActivityView()
        }
        .defaultSize(width: 400, height: 600)
        .keyboardShortcut("a", modifiers: [.command, .shift])
        #endif
    }
}
```

## watchOS Patterns

### Compact Navigation
```swift
struct WatchContentView: View {
    var body: some View {
        NavigationStack {
            List {
                ForEach(items) { item in
                    NavigationLink(value: item) {
                        ItemRow(item: item)
                    }
                }
            }
            .navigationTitle("Items")
            .navigationDestination(for: Item.self) { item in
                ItemDetailView(item: item)
            }
        }
    }
}
```

### Vertical Page TabView
```swift
TabView {
    SummaryView()
    DetailView()
    SettingsView()
}
.tabViewStyle(.verticalPage)
```

### Complications (WidgetKit)
```swift
struct WatchComplication: Widget {
    var body: some WidgetConfiguration {
        StaticConfiguration(kind: "complication", provider: ComplicationProvider()) { entry in
            ComplicationView(entry: entry)
        }
        .supportedFamilies([
            .accessoryCircular,
            .accessoryRectangular,
            .accessoryInline,
            .accessoryCorner
        ])
    }
}
```

## visionOS Patterns

### ImmersiveSpace
```swift
@main
struct VisionApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }

        ImmersiveSpace(id: "immersive") {
            ImmersiveView()
        }
        .immersionStyle(selection: .constant(.mixed), in: .mixed, .full)
    }
}

struct ContentView: View {
    @Environment(\.openImmersiveSpace) var openImmersiveSpace

    var body: some View {
        Button("Enter Immersive") {
            Task { await openImmersiveSpace(id: "immersive") }
        }
    }
}
```

### RealityView & Model3D
```swift
// Simple 3D model display
Model3D(named: "Robot") { model in
    model.resizable().scaledToFit()
} placeholder: {
    ProgressView()
}
.frame(width: 200, height: 200)

// Full RealityView for interactive 3D
RealityView { content in
    if let entity = try? await ModelEntity(named: "Scene") {
        content.add(entity)
    }
} update: { content in
    // Update entities when SwiftUI state changes
}
.gesture(TapGesture().targetedToAnyEntity().onEnded { value in
    // Handle tap on 3D entity
})
```

### Ornaments & Volumes
```swift
// Ornaments -- UI attached to window edges
WindowGroup {
    ContentView()
        .ornament(attachmentAnchor: .scene(.bottom)) {
            HStack {
                Button("Play", systemImage: "play.fill") { }
                Button("Pause", systemImage: "pause.fill") { }
            }
            .padding()
            .glassBackgroundEffect()
        }
}

// Volumes -- 3D content in bounded space
WindowGroup(id: "volume") {
    VolumeView()
}
.windowStyle(.volumetric)
.defaultSize(width: 0.5, height: 0.5, depth: 0.5, in: .meters)
```

## Shared Code Strategies

### SPM Modules
```
MyApp/
  Package.swift              # Shared SPM package
  Sources/
    SharedModels/            # Models shared across platforms
    SharedServices/          # Platform-agnostic services
    SharedViewModels/        # ViewModels (Foundation-only)
  MyAppiOS/                  # iOS app target
  MyAppmacOS/                # macOS app target
  MyAppWatch/                # watchOS app target
```

### Platform-Adaptive Type Aliases
```swift
#if os(iOS)
import UIKit
typealias PlatformColor = UIColor
typealias PlatformImage = UIImage
#elseif os(macOS)
import AppKit
typealias PlatformColor = NSColor
typealias PlatformImage = NSImage
#endif
```

## Platform-Specific HIG Differences

| Guideline | iOS | macOS | watchOS | visionOS |
|-----------|-----|-------|---------|----------|
| Touch targets | 44pt minimum | No minimum (pointer) | 38pt minimum | Hover highlight |
| Navigation | TabView + NavigationStack | NavigationSplitView sidebar | NavigationStack compact | WindowGroup + volumes |
| Text size | Dynamic Type | System font prefs | Dynamic Type (compact) | Dynamic Type |
| Primary input | Touch | Mouse/trackpad + keyboard | Touch + Digital Crown | Gaze + pinch |
| Window management | Single window (usually) | Multiple windows + resize | Single screen | Spatial windows |
| Toolbar placement | .navigationBarTrailing | .automatic (title bar) | N/A | .ornament |
| Settings | In-app or Settings.app | Settings scene (⌘,) | Companion app | In-app |
