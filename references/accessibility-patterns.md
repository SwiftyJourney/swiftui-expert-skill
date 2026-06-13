# SwiftUI Accessibility & Adaptive Layout Patterns

Semantic styling, adaptive spacing, accessibility representations, and standard component usage.

## Table of Contents
- [Standard Components First](#standard-components-first)
- [Semantic Styling](#semantic-styling)
- [Adaptive Spacing with @ScaledMetric](#adaptive-spacing-with-scaledmetric)
- [Content Legibility Settings](#content-legibility-settings)
- [Grouping for VoiceOver](#grouping-for-voiceover)
- [Accessibility Representation for Custom Controls](#accessibility-representation-for-custom-controls)
- [Custom Interactive (Non-Button) Elements](#custom-interactive-non-button-elements)
- [Decorative Content](#decorative-content)
- [Canvas and Raw Drawing](#canvas-and-raw-drawing)
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

## Content Legibility Settings

Three accessibility settings the system handles for its own components but leaves to YOU for custom UI. Read each from the environment.

```swift
// 1. Button Shapes — system buttons gain shape cues automatically; custom styles do NOT
struct LinkButtonStyle: ButtonStyle {
    @Environment(\.accessibilityShowButtonShapes) private var showShapes
    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .underline(showShapes)            // add a shape cue when enabled
    }
}

// 2. Reduce Transparency — system materials opacify automatically; hand-rolled translucency does not
struct Card<Content: View>: View {
    @Environment(\.accessibilityReduceTransparency) private var reduceTransparency
    let content: Content
    var body: some View {
        content.background(reduceTransparency ? AnyShapeStyle(.background)
                                              : AnyShapeStyle(.ultraThinMaterial))
    }
}

// 3. Accessibility text sizes — @ScaledMetric scales numbers; it can't reflow layout
struct Row: View {
    @Environment(\.dynamicTypeSize) private var typeSize
    var body: some View {
        let layout = typeSize.isAccessibilitySize
            ? AnyLayout(VStackLayout(alignment: .leading))
            : AnyLayout(HStackLayout())
        layout { Icon(); Title(); Spacer() }
    }
}
```

**Rule:** Guard custom button cues, custom translucency, and layout for `isAccessibilitySize` yourself — the system only auto-adapts its own components.

**Why**: `@ScaledMetric` resizes a value but can't turn an `HStack` into a `VStack`; at accessibility sizes a horizontal row clips or truncates, so branch the container (here via `AnyLayout`, which preserves subview identity) when `dynamicTypeSize.isAccessibilitySize` is true.

---

## Grouping for VoiceOver

**Rule:** Merge a related label+value cluster into one element with `accessibilityElement(children: .combine)` so VoiceOver reads it as a single statement instead of forcing swipes between fragments.

```swift
// GOOD - read as one element: "Downloads, 248 files"
HStack {
    Text("Downloads")
    Spacer()
    Text("248 files")
}
.accessibilityElement(children: .combine)
```

**Why**: by default each `Text` is its own accessibility element, so the user swipes through disconnected fragments. `.combine` merges the descendants' labels/values/traits into one. (`.contain` keeps children as a navigable group; `.ignore` drops them so you can supply your own label.)

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

## Custom Interactive (Non-Button) Elements

When a raw tap gesture is genuinely unavoidable (prefer a real `Button` first — see [Standard Components First](#standard-components-first)), a tappable `Image`/shape is invisible-as-interactive to VoiceOver. Restore it with the trait + value + hint trio.

```swift
// BAD - VoiceOver sees an image, not a control
Image(systemName: isFavorite ? "star.fill" : "star")
    .onTapGesture { isFavorite.toggle() }

// GOOD - announced and operable as a button
Image(systemName: isFavorite ? "star.fill" : "star")
    .onTapGesture { isFavorite.toggle() }
    .accessibilityAddTraits(.isButton)              // signals interactivity
    .accessibilityLabel("Favorite")
    .accessibilityValue(isFavorite ? "On" : "Off")  // current state
    .accessibilityHint("Toggles whether this item is a favorite")  // what it does
```

**Why**: traits tell assistive tech the element is operable, the value conveys its current state, and the hint explains the outcome — the three pieces a real `Button` provides for free.

---

## Decorative Content

**Rule:** Remove purely decorative views from the accessibility tree so VoiceOver isn't cluttered, and give meaningful images a real label (SwiftUI otherwise defaults an image's label to its asset name).

```swift
Image("hero-flourish")
    .accessibilityHidden(true)        // skip decorative art entirely

Image(decorative: "divider")          // asset image with no announced label

Image("tui-bird")
    .accessibilityLabel("A tūī perched on a flax flower")  // override the "tui-bird, image" default
```

**Why**: undescribed decorative images announce noise ("image"); meaningful images announce their file name unless you supply a label conveying what sighted users see.

---

## Canvas and Raw Drawing

**Rule:** `Canvas` (and raw shape/gesture compositions) contribute **nothing** to the accessibility tree — their contents are invisible to VoiceOver, and a gesture-only control also fails AssistiveTouch/Switch Control (which drive controls via discrete steps). Provide `accessibilityRepresentation`, or add explicit traits plus `accessibilityAdjustableAction`.

```swift
// Canvas-drawn rating with no native semantics
Canvas { context, size in /* draw stars for `rating` */ }
    .accessibilityLabel("Rating")
    .accessibilityValue("\(rating) of 5")
    .accessibilityAdjustableAction { direction in
        switch direction {
        case .increment: rating = min(5, rating + 1)
        case .decrement: rating = max(0, rating - 1)
        @unknown default: break
        }
    }
```

**Why**: VoiceOver and Switch Control can't see pixels — without a representation or an adjustable action, custom-drawn controls are unusable for assistive-tech users even though they look complete in a sighted demo.

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
- [ ] Custom interactive (non-Button) elements have `.isButton` trait + value + hint
- [ ] Decorative images hidden (`accessibilityHidden`/`Image(decorative:)`); meaningful images have a real label
- [ ] `Canvas`/custom-drawn controls expose `accessibilityRepresentation` or an adjustable action
- [ ] Custom button shapes / translucency / layout adapt to Button Shapes, Reduce Transparency, and accessibility text sizes
- [ ] Interface tested at largest and smallest Dynamic Type sizes, and under Reduce Transparency, Increased Contrast, and Smart Invert
