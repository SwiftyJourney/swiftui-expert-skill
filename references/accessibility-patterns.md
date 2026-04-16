# SwiftUI Accessibility & Adaptive Layout Patterns

Semantic styling, adaptive spacing, accessibility representations, and standard component usage.

## Table of Contents
- [Standard Components First](#standard-components-first)
- [Semantic Styling](#semantic-styling)
- [Adaptive Spacing with @ScaledMetric](#adaptive-spacing-with-scaledmetric)
- [Accessibility Representation for Custom Controls](#accessibility-representation-for-custom-controls)
- [Custom Styles Preserve Accessibility](#custom-styles-preserve-accessibility)
- [Platform-Specific Adjustments](#platform-specific-adjustments)

---

## Standard Components First

Always start with native controls. Built-in components carry semantic meaning, accessibility traits, haptic feedback, and platform-appropriate behavior automatically.

### Provide Full Semantic Context

```swift
// GOOD — text + image gives SwiftUI full semantic context
Button("Edit", systemImage: "pencil") {
    // action
}
// Adapts appearance per context (toolbar, swipe action, list)
// VoiceOver automatically reads "Edit, button"

// BAD — custom label loses accessibility traits
Button {
    // action
} label: {
    Image(systemName: "pencil")
    // VoiceOver: "pencil, button" — meaningless to the user
}

// WORST — not even a button
Image(systemName: "pencil")
    .onTapGesture { /* action */ }
    // No button traits, no highlight state, invisible to VoiceOver
```

**Rule**: Use `Button("Title", systemImage:)` for actions. Apply `.labelStyle(.iconOnly)` when the design needs icon-only display — the accessibility label is preserved.

```swift
// Show icon only in swipe action, keep accessibility
ObservationRow(observation: observation)
    .swipeActions {
        EditObservationButton(observationID: observation.id)
            .labelStyle(.iconOnly)
    }
```

### Structural Components Define Context

High-order containers (`TabView`, `NavigationStack`, `NavigationSplitView`, `List`, `Form`) provide the semantic framework that allows controls to resolve their roles and spatial relationships automatically.

```swift
// Button adapts its appearance based on context:
// - In a toolbar: icon-appropriate sizing
// - In a swipe action: colored background
// - In a list row: standard row styling
// No extra code needed — SwiftUI handles it
```

---

## Semantic Styling

Use semantic styles that describe the **role** of an element, not its exact appearance. This ensures the interface adapts to Dark Mode, Dynamic Type, and high-contrast settings.

```swift
// GOOD — semantic styling adapts to any environment
struct EcosystemOverview: View {
    let name: String
    let details: String

    var body: some View {
        VStack(alignment: .leading) {
            Text(name)
                .font(.headline)           // Semantic: "this is a heading"
            Text(details)
                .foregroundStyle(.secondary) // Semantic: "this is secondary"
        }
    }
}

// BAD — hardcoded values break in Dark Mode and accessibility settings
VStack {
    Text(name)
        .font(.system(size: 17, weight: .bold))  // Won't scale with Dynamic Type
    Text(details)
        .foregroundColor(.gray)                    // Poor contrast in Dark Mode
}
```

### Hierarchy Through Semantic Tokens

| Semantic | Purpose |
|----------|---------|
| `.primary` | Main content |
| `.secondary` | Supporting content |
| `.tertiary` | Supplementary content |
| `.quaternary` | Decorative or least important |
| `.headline` | Section headers |
| `.subheadline` | Supporting headers |
| `.caption` | Metadata, timestamps |

---

## Adaptive Spacing with @ScaledMetric

Use `@ScaledMetric` for custom spacing and padding values that scale proportionally with the user's Dynamic Type settings.

```swift
// GOOD — spacing scales with Dynamic Type
struct SpeciesDetailView: View {
    let species: Species
    @ScaledMetric private var spacing = 22

    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: spacing) {
                SpeciesInfoSection(header: "Scientific Name", info: species.scientificName)
                SpeciesInfoSection(header: "Description", info: species.description)
                SpeciesInfoSection(header: "Status", info: species.status)
            }
        }
    }
}

// BAD — fixed spacing becomes cramped at larger text sizes
VStack(alignment: .leading, spacing: 22) { /* ... */ }
```

### Relative Scaling

By default, `@ScaledMetric` scales relative to the `body` text style. Use `relativeTo` for specific contexts:

```swift
@ScaledMetric(relativeTo: .caption) private var captionPadding = 8
@ScaledMetric(relativeTo: .headline) private var headerSpacing = 16
```

### When to Use

- Custom spacing between elements
- Custom padding that doesn't match system defaults
- Custom icon sizes that should scale with text
- Any fixed numeric value that relates to text content

**When NOT to use**: For standard padding, prefer the parameterless `.padding()` which applies system-standard adaptive spacing automatically.

---

## Accessibility Representation for Custom Controls

When building highly custom controls that can't use a style protocol, provide an `accessibilityRepresentation` using a standard system component. This ensures VoiceOver users interact with the control as if it were native.

```swift
struct SegmentedControl: View {
    let titleKey: LocalizedStringKey
    @Binding var selection: SegmentOption

    var body: some View {
        HStack {
            ForEach(SegmentOption.allCases) { option in
                Button {
                    selection = option
                } label: {
                    // Custom visual design
                    Text(option.rawValue)
                        .padding(.horizontal, 16)
                        .padding(.vertical, 8)
                        .background(selection == option ? Color.accentColor : .clear)
                        .clipShape(.capsule)
                }
            }
        }
        .accessibilityRepresentation {
            Picker(titleKey, selection: $selection) {
                ForEach(SegmentOption.allCases) { option in
                    Text(option.rawValue).tag(option)
                }
            }
            .labelsHidden()
        }
    }
}
```

**Why**: The visual interface is fully custom, but VoiceOver users experience a standard `Picker` — familiar navigation, value announcements, and adjustment gestures all work automatically.

### When to Use

- Custom drawing with `Canvas` — always needs `accessibilityRepresentation`
- Custom gesture-based controls — map to standard increment/decrement
- Custom segmented controls, sliders, or pickers that don't use style protocols

---

## Custom Styles Preserve Accessibility

When a design requires a custom appearance, check if the component's style protocol is available. Custom styles preserve all semantic traits automatically.

```swift
// Custom toggle that looks completely different but IS a Toggle for VoiceOver
struct SymbolToggleStyle: ToggleStyle {
    let symbolName: String

    func makeBody(configuration: Configuration) -> some View {
        Button {
            configuration.isOn.toggle()
        } label: {
            Label {
                configuration.label
            } icon: {
                Image(systemName: symbolName)
                    .symbolVariant(configuration.isOn ? .none : .slash)
                    .contentTransition(.symbolEffect)
            }
            .labelStyle(.iconOnly)
        }
    }
}

// Usage — inherits Toggle accessibility traits
Toggle("Notifications", isOn: $notificationsEnabled)
    .toggleStyle(SymbolToggleStyle(symbolName: "bell"))
```

**Available style protocols**: `ButtonStyle`, `ToggleStyle`, `LabelStyle`, `TextFieldStyle`, `PickerStyle` (limited), `GroupBoxStyle`, `GaugeStyle`, `ProgressViewStyle`.

---

## Platform-Specific Adjustments

Extract platform and availability checks into custom modifiers to keep view bodies clean.

```swift
extension View {
    func sheetNavigationTransition(
        sourceID: String,
        namespaceID: Namespace.ID
    ) -> some View {
        #if os(iOS)
        navigationTransition(.zoom(sourceID: sourceID, in: namespaceID))
        #elseif os(macOS)
        navigationTransition(.automatic)
        #endif
    }
}

struct LooseLineHeight: ViewModifier {
    @Environment(\.font) var font

    func body(content: Content) -> some View {
        if #available(iOS 26.0, macOS 26.0, *) {
            content.lineHeight(.loose)
        } else {
            content.font((font ?? .body).leading(.loose))
        }
    }
}

extension View {
    func looseLineHeight() -> some View {
        modifier(LooseLineHeight())
    }
}
```

**Why**: Platform and API checks are resolved at compile-time or app launch — they don't change during execution, so view identity remains stable.

---

## Checklist

- [ ] Actions use `Button` with text + image (not `Image().onTapGesture`)
- [ ] `.labelStyle(.iconOnly)` used to hide text visually while keeping accessibility label
- [ ] Semantic fonts used (`.headline`, `.body`, `.caption`) — not hardcoded sizes
- [ ] Semantic foreground styles used (`.primary`, `.secondary`) — not hardcoded colors
- [ ] `@ScaledMetric` used for custom spacing/padding that relates to text content
- [ ] Custom controls use `accessibilityRepresentation` when style protocols unavailable
- [ ] Custom styles preferred over manual reimplementation when style protocol exists
- [ ] Platform checks extracted into custom modifiers (not inline in view body)
- [ ] Interface tested at largest and smallest Dynamic Type sizes
