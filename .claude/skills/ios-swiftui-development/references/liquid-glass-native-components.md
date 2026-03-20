# Liquid Glass & Native Components (iOS 26)

## What is Liquid Glass?

Liquid glass is iOS 26's design language: a translucent, light-refracting material applied to system chrome (navigation bars, tab bars, toolbars). It creates depth and hierarchy by letting background content show through with a glass-like distortion effect.

## Automatic Glass (System Components)

These components receive liquid glass styling automatically -- no code changes needed:

| Component | Glass Behavior |
|-----------|---------------|
| `NavigationStack` bar | Glass navigation bar |
| `TabView` | Floating glass tab bar |
| Toolbars (`.toolbar`) | Glass toolbar |
| `.bottomBar` placement | Glass bottom bar |
| System alerts | Glass alert background |
| Sheets (partial) | Glass sheet grabber area |
| Popovers | Glass popover chrome |

**Do NOT override or customize these.** The system handles glass appearance, blur, and tinting automatically.

## Manual Glass (.glassEffect)

For custom floating UI elements that should match system glass:

```swift
// Basic glass effect
Text("Floating Label")
    .padding()
    .glassEffect(.regular)

// Glass effect with shape
VStack { ... }
    .padding()
    .glassEffect(.regular.interactive)
    .clipShape(RoundedRectangle(cornerRadius: 16))
```

### GlassEffectContainer

Group multiple glass elements so they share tinting and blur context:

```swift
GlassEffectContainer {
    HStack {
        Button("Action 1") { }
            .glassEffect(.regular)

        Button("Action 2") { }
            .glassEffect(.regular)
    }
}
```

### When to Use Manual Glass

- Floating action buttons overlaying content
- Custom floating toolbars or controls
- Overlay indicators (e.g., now-playing bar)
- Picture-in-picture style floating panels

### When NOT to Use Glass

- Content backgrounds (cards, list rows, sections)
- Full-screen backgrounds
- Text containers or reading surfaces
- Inline UI elements that don't float
- Any element that doesn't overlay other content

## Mesh Gradients

Rich, dynamic backgrounds that look great behind glass:

```swift
MeshGradient(
    width: 3, height: 3,
    points: [
        [0, 0], [0.5, 0], [1, 0],
        [0, 0.5], [0.5, 0.5], [1, 0.5],
        [0, 1], [0.5, 1], [1, 1]
    ],
    colors: [
        .blue, .purple, .indigo,
        .cyan, .mint, .teal,
        .green, .yellow, .orange
    ]
)
.ignoresSafeArea()
```

**Tips:**
- Use as background behind NavigationStack for rich visual depth
- Animate point positions for dynamic, living backgrounds
- Keep colors harmonious -- 2-3 hue families work best
- Test with glass overlays to ensure readability

## Native Components Catalog

Always prefer system components over custom implementations:

| Need | Use | NOT |
|------|-----|-----|
| Date/time input | `DatePicker` | Custom date wheels |
| Color selection | `ColorPicker` | Custom color grid |
| Photo selection | `PhotosPicker` | Custom image browser |
| Share content | `ShareLink` | Custom share menu |
| Map display | `Map` (MapKit) | Custom map view |
| Video playback | `VideoPlayer` | Custom AVPlayer wrapper |
| Web content | `WebView` (SafariServices) | Custom WKWebView |
| File picking | `.fileImporter()` | Custom file browser |
| Camera capture | `.fullScreenCover` + `UIImagePickerController` | Custom camera UI |
| In-app purchase | StoreKit views (`SubscriptionStoreView`) | Custom paywall |
| Authentication | `SignInWithAppleButton` | Custom Apple Sign In |
| Text editing (rich) | `TextEditor` | Custom text view |
| Progress | `ProgressView` (linear/circular) | Custom spinners |
| Gauge/meter | `Gauge` | Custom progress bars |
| Charts | Swift Charts (`Chart`) | Custom chart drawing |
| Search | `.searchable()` | Custom search bar |

## Dark Mode Considerations

- Liquid glass adapts automatically to light/dark mode
- Custom glass effects (`.glassEffect`) also adapt automatically
- Test both modes -- glass opacity and tinting differs
- Behind glass, use system colors that adjust per mode
- Mesh gradients may need different color palettes per mode:
  ```swift
  @Environment(\.colorScheme) var colorScheme

  let colors = colorScheme == .dark
      ? [Color.indigo, .purple, .blue]
      : [Color.cyan, .mint, .teal]
  ```

## Performance Notes

- Liquid glass uses GPU-accelerated blur and compositing
- Avoid stacking multiple glass effects (causes overdraw)
- Limit animated content behind glass surfaces (reduces blur recomputation)
- Use `.drawingGroup()` for complex view hierarchies behind glass
- `MeshGradient` is GPU-efficient but avoid animating all 9+ points simultaneously
- Profile with Instruments (Core Animation template) if frame rate drops

## Accessibility

- Glass respects `accessibilityReduceTransparency` -- system falls back to opaque
- Always ensure text on glass meets contrast requirements (4.5:1)
- Don't rely on glass visual effect to convey meaning
- Test with "Increase Contrast" and "Reduce Transparency" enabled in Settings
