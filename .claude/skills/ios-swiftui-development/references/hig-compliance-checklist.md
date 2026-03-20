# Human Interface Guidelines Compliance Checklist

## Layout & Spacing

- [ ] Respect safe areas -- never place interactive content under status bar, home indicator, or Dynamic Island
- [ ] Use system standard margins (16pt on iPhone, 20pt on larger devices) or `.padding()`
- [ ] Minimum 44x44pt touch targets for all interactive elements
- [ ] Use `VStack`/`HStack` alignment guides, not manual offsets
- [ ] Support both portrait and landscape where appropriate
- [ ] Use `.containerRelativeFrame()` for responsive sizing
- [ ] Avoid fixed widths/heights -- use flexible layout (`.frame(maxWidth: .infinity)`)
- [ ] Respect keyboard avoidance -- SwiftUI handles this by default, don't fight it

## Typography

- [ ] Use Dynamic Type for ALL text -- never hardcode font sizes
- [ ] Use semantic font styles: `.largeTitle`, `.title`, `.headline`, `.body`, `.caption`, etc.
- [ ] Use SF Pro (system font) unless brand guidelines require custom fonts
- [ ] If custom fonts: register in Info.plist and scale with `@ScaledMetric`
- [ ] Establish clear text hierarchy: title > headline > body > caption
- [ ] Use `.minimumScaleFactor()` for text that must fit a fixed space
- [ ] Support Bold Text accessibility setting (use semantic weights)
- [ ] Limit line length to ~70 characters for readability

## Color & Materials

- [ ] Use system semantic colors: `Color.primary`, `.secondary`, `.accentColor`
- [ ] Use system backgrounds: `.background`, `.secondarySystemBackground`
- [ ] Support Dark Mode -- test BOTH appearances, never hardcode light-only colors
- [ ] Custom colors: define in asset catalog with light/dark variants
- [ ] Minimum contrast ratio 4.5:1 for normal text, 3:1 for large text (WCAG AA)
- [ ] Never convey meaning through color alone -- pair with icons, text, or shape
- [ ] Use system materials (`.regularMaterial`, `.ultraThinMaterial`) for translucency
- [ ] Liquid glass: let system components handle it; use `.glassEffect` sparingly on custom elements
- [ ] Respect `accessibilityReduceTransparency` and `accessibilityIncreaseContrast`

## Icons & Images

- [ ] Use SF Symbols for all standard icons (1000+ available)
- [ ] Match SF Symbol weight to adjacent text weight
- [ ] Use appropriate rendering mode: `.monochrome`, `.hierarchical`, `.palette`, `.multicolor`
- [ ] Custom icons: match SF Symbol design language (stroke weight, optical alignment)
- [ ] Provide @2x and @3x assets in asset catalog
- [ ] Tab bar icons: filled variant for selected, outline for unselected
- [ ] App icon: follow Apple's icon grid template, no transparency
- [ ] Use `Label("Title", systemImage: "icon.name")` for icon+text pairs

## Navigation

- [ ] Use `TabView` for top-level navigation (max 5 tabs, use "More" if needed)
- [ ] Use `NavigationStack` for hierarchical drill-down
- [ ] Modals (`sheet`) for self-contained tasks that can be dismissed
- [ ] `fullScreenCover` only for immersive content (photo viewer, video player)
- [ ] Provide clear back navigation -- never trap users in a screen
- [ ] Navigation titles: use `.large` for top-level, `.inline` for detail screens
- [ ] Preserve scroll position when navigating back
- [ ] Support swipe-back gesture (NavigationStack provides this by default)

## Interaction & Feedback

- [ ] Use haptic feedback for meaningful actions (not every tap):
  - `.impact(.light)` for selections
  - `.impact(.medium)` for toggles, confirmations
  - `.notification(.success/.warning/.error)` for outcomes
- [ ] Support standard gestures: tap, long press, swipe, pinch, drag
- [ ] Swipe actions on list rows for common actions (delete, archive, pin)
- [ ] Context menus (`.contextMenu`) for secondary actions
- [ ] Show loading indicators for operations >1 second
- [ ] Destructive actions require confirmation (`.destructive` button role)
- [ ] Provide undo where possible (`.onSubmit`, `UndoManager`)
- [ ] Respond to input immediately -- never block the main thread

## Accessibility

- [ ] All interactive elements have accessibility labels:
  ```swift
  Button { } label: { Image(systemName: "trash") }
      .accessibilityLabel("Delete item")
  ```
- [ ] Decorative images: `.accessibilityHidden(true)`
- [ ] Group related elements: `.accessibilityElement(children: .combine)`
- [ ] Custom actions for complex interactions: `.accessibilityAction`
- [ ] Support VoiceOver navigation order (logical reading order)
- [ ] Support Voice Control (all buttons must have discoverable labels)
- [ ] Respect Reduce Motion: check `accessibilityReduceMotion`
  ```swift
  @Environment(\.accessibilityReduceMotion) var reduceMotion
  withAnimation(reduceMotion ? nil : .spring()) { ... }
  ```
- [ ] Respect Reduce Transparency: check `accessibilityReduceTransparency`
- [ ] Provide sufficient touch target size even with larger text sizes
- [ ] Test with VoiceOver, Dynamic Type at max, and Bold Text enabled

## Platform Conventions

- [ ] Never hide or customize the status bar without good reason
- [ ] Respect home indicator -- don't place controls in bottom safe area
- [ ] Support pull-to-refresh (`.refreshable`) on scrollable content
- [ ] Use system share sheet (`ShareLink`) for sharing content
- [ ] Use system photo picker (`PhotosPicker`) for image selection
- [ ] Use system date/time pickers (`DatePicker`) for date input
- [ ] Use system color picker (`ColorPicker`) for color input
- [ ] Support Spotlight indexing for searchable content
- [ ] Handle interruptions gracefully (phone calls, notifications)
- [ ] Save state on background -- restore on foreground (`@SceneStorage`)

## Content States

Every data-driven view must handle ALL states:

| State | Implementation |
|-------|---------------|
| Loading | `ProgressView` with optional description |
| Empty | `ContentUnavailableView` with icon, message, and action |
| Error | `ContentUnavailableView` with error message and retry button |
| Loaded | Primary content |
| Partial/Offline | Cached content with refresh indicator |

```swift
// Pattern for all data views
Group {
    if viewModel.isLoading {
        ProgressView()
    } else if let error = viewModel.errorMessage {
        ContentUnavailableView {
            Label("Error", systemImage: "exclamationmark.triangle")
        } description: {
            Text(error)
        } actions: {
            Button("Retry") { Task { await viewModel.load() } }
        }
    } else if viewModel.items.isEmpty {
        ContentUnavailableView.search  // or custom empty state
    } else {
        // Main content
    }
}
```
