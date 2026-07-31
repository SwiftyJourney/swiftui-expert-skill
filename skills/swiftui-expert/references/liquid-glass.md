# SwiftUI Liquid Glass Reference (iOS 26+)

## Overview

Liquid Glass is Apple's design language introduced in iOS 26. It provides translucent, dynamic surfaces that blur and adjust luminosity for legibility, responding to content beneath them and to user interaction. The primary modifier is `glassEffect`, which captures the view's content and renders it through a glass surface. The default shape is `Capsule` (internally `DefaultGlassEffectShape`).

## Availability

All Liquid Glass APIs require iOS 26 or later. Always provide fallbacks:

```swift
if #available(iOS 26, *) {
    content
        .glassEffect()
} else {
    content
        .background(.ultraThinMaterial, in: Capsule())
}
```

## Core APIs

### glassEffect Modifier

The primary modifier for applying glass effects to views:

```swift
func glassEffect(_ glass: Glass = .regular, in shape: some InsettableShape) -> some View
```

The default variant is `.regular`. When no shape is provided, the default shape is `Capsule`.

#### Basic Usage

```swift
Text("Hello")
    .padding()
    .glassEffect()  // Default: .regular variant, capsule shape
```

#### With Custom Shape

```swift
Text("Rounded Glass")
    .padding()
    .glassEffect(in: .rect(cornerRadius: 16))

Image(systemName: "star")
    .padding()
    .glassEffect(in: .circle)

Text("Capsule")
    .padding(.horizontal, 20)
    .padding(.vertical, 10)
    .glassEffect(in: .capsule)
```

### Glass Variants

There are three `Glass` variants. There is no `.prominent` variant on `Glass` -- `.prominent` only exists as a button style (`.buttonStyle(.glassProminent)`).

#### Glass.regular

Standard glass appearance. Blurs the background and adjusts luminosity to ensure foreground content remains legible. Most system components use this variant.

```swift
.glassEffect(.regular)
// Or simply:
.glassEffect()
```

#### Glass.clear

Highly translucent glass. Ideal for components floating over visually rich backgrounds such as photos, videos, or maps. Use only when the underlying content is the priority and the glass element should recede.

```swift
.glassEffect(.clear)
```

#### Glass.identity

No glass effect is applied; the content remains completely unaffected. Useful for conditionally toggling a glass effect without branching logic.

```swift
.glassEffect(showGlass ? .regular : .identity)
```

### Tinting

Add a color tint to any glass variant:

```swift
.glassEffect(.regular.tint(.blue))
.glassEffect(.clear.tint(.red.opacity(0.3)))
```

### Interactivity

Make glass respond to touch and pointer hover:

```swift
// Interactive glass - responds to user interaction
.glassEffect(.regular.interactive())

// Combined with tint
.glassEffect(.regular.tint(.blue).interactive())
```

**Important**: Only use `.interactive()` on elements that actually respond to user input (buttons, tappable views, focusable elements). Do not apply it to static or decorative glass surfaces.

## GlassEffectContainer

Wraps multiple glass elements for proper visual grouping and spacing. The container manages how glass effects relate to each other visually, including proximity-based transitions.

```swift
GlassEffectContainer {
    HStack {
        Button("One") { }
            .glassEffect()
        Button("Two") { }
            .glassEffect()
    }
}
```

### With Spacing

Control the visual spacing between glass elements. The container's `spacing` parameter should match the actual spacing in your layout for proper glass effect rendering.

```swift
GlassEffectContainer(spacing: 24) {
    HStack(spacing: 24) {
        GlassChip(icon: "pencil")
        GlassChip(icon: "eraser")
        GlassChip(icon: "trash")
    }
}
```

## Glass Effect Union

`glassEffectUnion(id:namespace:)` combines multiple views into a single unified glass shape. This is different from `GlassEffectContainer` (which manages spacing and grouping) -- union merges the actual visual shapes of several views into one continuous glass surface.

```swift
struct WeatherIcons: View {
    @Namespace private var namespace

    let symbolSet = ["cloud.bolt.rain.fill", "sun.rain.fill", "moon.stars.fill", "moon.fill"]

    var body: some View {
        GlassEffectContainer(spacing: 20) {
            HStack(spacing: 20) {
                ForEach(symbolSet.indices, id: \.self) { item in
                    Image(systemName: symbolSet[item])
                        .frame(width: 80, height: 80)
                        .font(.system(size: 36))
                        .glassEffect()
                        .glassEffectUnion(id: item < 2 ? "1" : "2", namespace: namespace)
                }
            }
        }
    }
}
```

In this example, the first two icons merge into one glass shape and the last two merge into another, creating two unified glass regions within the container.

## Glass Button Styles

Built-in button styles for glass appearance:

```swift
// Standard glass button
Button("Action") { }
    .buttonStyle(.glass)

// Prominent glass button (higher visibility)
Button("Primary Action") { }
    .buttonStyle(.glassProminent)
```

**Note**: `.glassProminent` is a button style only. There is no `.prominent` variant on `Glass` itself.

### Custom Glass Buttons

For more control, apply the glass effect modifier directly:

```swift
Button(action: { }) {
    Label("Settings", systemImage: "gear")
        .padding()
}
.glassEffect(.regular.interactive(), in: .capsule)
```

## Morphing Transitions

Create smooth transitions between glass elements using `glassEffectID`, `@Namespace`, and `GlassEffectTransition`.

### glassEffectID

Pairs glass elements for matched-geometry morphing transitions:

```swift
struct MorphingExample: View {
    @Namespace private var animation
    @State private var isExpanded = false

    var body: some View {
        GlassEffectContainer {
            if isExpanded {
                ExpandedCard()
                    .glassEffect()
                    .glassEffectID("card", in: animation)
            } else {
                CompactCard()
                    .glassEffect()
                    .glassEffectID("card", in: animation)
            }
        }
        .animation(.smooth, value: isExpanded)
    }
}
```

### GlassEffectTransition

Specifies the type of transition when glass effects are added or removed within a container:

- **`.matchedGeometry`** -- Default for shapes within the container's spacing distance. Morphs shapes smoothly into each other.
- **`.materialize`** -- Used for shapes farther apart or when a simpler transition is appropriate. Also used for custom transition scenarios.

```swift
Image(systemName: "eraser.fill")
    .glassEffect()
    .glassEffectID("eraser", in: namespace)
    .glassEffectTransition(.materialize)
```

### Requirements for Morphing

1. Both views must share the same `glassEffectID`.
2. Use the same `@Namespace`.
3. Wrap in a `GlassEffectContainer`.
4. Apply animation to the container or parent.

## Modifier Order

**Critical**: `glassEffect` captures its content and sends it to the container for rendering. It must come AFTER all appearance and layout modifiers so that those modifiers are captured correctly.

```swift
// CORRECT order
Text("Label")
    .font(.headline)           // 1. Typography
    .foregroundStyle(.primary) // 2. Color
    .padding()                 // 3. Layout
    .glassEffect()             // 4. Glass effect LAST

// WRONG order - glass applied too early, modifiers after it are not captured
Text("Label")
    .glassEffect()             // Wrong position
    .padding()
    .font(.headline)
```

## Human Interface Guidelines

Apple's HIG provides specific guidance for Liquid Glass usage:

- **Do not use Liquid Glass in the content layer.** Glass is designed for controls and navigation elements (tab bars, sidebars, toolbars) that float above content. Use standard materials for content-layer elements.
- **Use Liquid Glass effects sparingly.** Standard system components from SwiftUI and UIKit pick up glass automatically. Only apply glass manually to the most important custom functional elements.
- **Only use `Glass.clear` over visually rich backgrounds.** Photos, videos, and maps are appropriate contexts. Use `Glass.regular` when legibility might be a concern, such as in alerts, sidebars, and popovers.
- **Performance awareness.** Creating too many `GlassEffectContainer` instances and applying too many glass effects outside of containers can degrade performance. Limit the number of glass effects rendered onscreen simultaneously.

## Complete Examples

### Toolbar with Glass Buttons

```swift
struct GlassToolbar: View {
    var body: some View {
        if #available(iOS 26, *) {
            GlassEffectContainer(spacing: 16) {
                HStack(spacing: 16) {
                    ToolbarButton(icon: "pencil", action: { })
                    ToolbarButton(icon: "eraser", action: { })
                    ToolbarButton(icon: "scissors", action: { })
                    Spacer()
                    ToolbarButton(icon: "square.and.arrow.up", action: { })
                }
                .padding(.horizontal)
            }
        } else {
            HStack(spacing: 16) {
                // Fallback toolbar using standard buttons
            }
        }
    }
}

struct ToolbarButton: View {
    let icon: String
    let action: () -> Void

    var body: some View {
        Button(action: action) {
            Image(systemName: icon)
                .font(.title2)
                .frame(width: 44, height: 44)
        }
        .glassEffect(.regular.interactive(), in: .circle)
    }
}
```

### Card with Glass Effect

```swift
struct GlassCard: View {
    let title: String
    let subtitle: String

    var body: some View {
        if #available(iOS 26, *) {
            cardContent
                .glassEffect(.regular, in: .rect(cornerRadius: 20))
        } else {
            cardContent
                .background(.ultraThinMaterial, in: RoundedRectangle(cornerRadius: 20))
        }
    }

    private var cardContent: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text(title)
                .font(.headline)
            Text(subtitle)
                .font(.subheadline)
                .foregroundStyle(.secondary)
        }
        .padding()
        .frame(maxWidth: .infinity, alignment: .leading)
    }
}
```

### Segmented Control

```swift
struct GlassSegmentedControl: View {
    @Binding var selection: Int
    let options: [String]
    @Namespace private var animation

    var body: some View {
        if #available(iOS 26, *) {
            GlassEffectContainer(spacing: 4) {
                HStack(spacing: 4) {
                    ForEach(options.indices, id: \.self) { index in
                        Button(options[index]) {
                            withAnimation(.smooth) {
                                selection = index
                            }
                        }
                        .padding(.horizontal, 16)
                        .padding(.vertical, 8)
                        .glassEffect(
                            selection == index
                                ? .regular.tint(.accentColor).interactive()
                                : .regular.interactive(),
                            in: .capsule
                        )
                        .glassEffectID(
                            selection == index ? "selected" : "option\(index)",
                            in: animation
                        )
                    }
                }
                .padding(4)
            }
        } else {
            Picker("Options", selection: $selection) {
                ForEach(options.indices, id: \.self) { index in
                    Text(options[index]).tag(index)
                }
            }
            .pickerStyle(.segmented)
        }
    }
}
```

### Weather Icons with Glass Union

```swift
struct WeatherBar: View {
    @Namespace private var namespace
    let symbolSet = ["cloud.bolt.rain.fill", "sun.rain.fill", "moon.stars.fill", "moon.fill"]

    var body: some View {
        if #available(iOS 26, *) {
            GlassEffectContainer(spacing: 20) {
                HStack(spacing: 20) {
                    ForEach(symbolSet.indices, id: \.self) { item in
                        Image(systemName: symbolSet[item])
                            .frame(width: 80, height: 80)
                            .font(.system(size: 36))
                            .glassEffect()
                            .glassEffectUnion(
                                id: item < 2 ? "1" : "2",
                                namespace: namespace
                            )
                    }
                }
            }
        }
    }
}
```

## Fallback Strategies

### Using Materials

```swift
if #available(iOS 26, *) {
    content
        .glassEffect()  // Capsule shape by default
} else {
    content
        .background(.ultraThinMaterial, in: Capsule())
}
```

### With Custom Shape

```swift
if #available(iOS 26, *) {
    content
        .glassEffect(in: .rect(cornerRadius: 16))
} else {
    content
        .background(.ultraThinMaterial, in: RoundedRectangle(cornerRadius: 16))
}
```

### Available Materials for Fallback

- `.ultraThinMaterial` -- Closest to glass appearance
- `.thinMaterial` -- Slightly more opaque
- `.regularMaterial` -- Standard blur
- `.thickMaterial` -- More opaque
- `.ultraThickMaterial` -- Most opaque

### Conditional Modifier Extension

```swift
extension View {
    @ViewBuilder
    func glassEffectWithFallback(
        _ glass: Glass = .regular,
        in shape: some InsettableShape,
        fallbackMaterial: Material = .ultraThinMaterial
    ) -> some View {
        if #available(iOS 26, *) {
            self.glassEffect(glass, in: shape)
        } else {
            self.background(fallbackMaterial, in: shape)
        }
    }
}

// Usage
Text("Label")
    .padding()
    .glassEffectWithFallback(in: .capsule)

Text("Card")
    .padding()
    .glassEffectWithFallback(.regular, in: .rect(cornerRadius: 16))
```

## Performance Guidelines

- **Minimize GlassEffectContainer instances.** Each container adds rendering overhead. Group related glass elements into a single container whenever possible.
- **Avoid excessive glass effects outside containers.** Standalone glass effects without a container are more expensive to render.
- **Limit onscreen glass effects.** The more glass surfaces rendered simultaneously, the greater the performance cost. Reserve glass for key interactive and navigational elements.
- **Use `Glass.identity` for conditional toggling.** Instead of conditionally wrapping views, use `.identity` to disable the effect without changing the view hierarchy.

## Best Practices

### Do

- Use `GlassEffectContainer` for grouped glass elements
- Apply `glassEffect` after all appearance and layout modifiers
- Use `.interactive()` only on tappable or focusable elements
- Match container spacing with layout spacing
- Provide `.ultraThinMaterial`-based fallbacks for older iOS versions
- Keep glass shapes consistent within a feature
- Use `Glass.regular` as the default for most controls
- Use `Glass.clear` only over visually rich backgrounds (photos, videos, maps)
- Use `glassEffectUnion` to merge related glass shapes into one surface

### Don't

- Apply glass to every element -- use sparingly, per HIG
- Use `.interactive()` on static or decorative content
- Use `Glass.clear` where legibility is a concern
- Apply glass to content-layer elements (it is for controls and navigation only)
- Mix different corner radii arbitrarily within a group
- Forget iOS 26 availability checks
- Apply glass before padding/frame/font modifiers
- Nest `GlassEffectContainer` instances unnecessarily
- Use `.glassEffect(.prominent)` -- this does not exist; use `.buttonStyle(.glassProminent)` for prominent buttons

## Checklist

- [ ] `#available(iOS 26, *)` with `.ultraThinMaterial` fallback
- [ ] `GlassEffectContainer` wraps grouped glass elements
- [ ] `.glassEffect()` applied after all appearance and layout modifiers
- [ ] `.interactive()` only on user-interactable elements
- [ ] `glassEffectID` with `@Namespace` for morphing transitions
- [ ] `glassEffectTransition` set appropriately (`.matchedGeometry` or `.materialize`)
- [ ] `glassEffectUnion` used where multiple views should merge into one glass shape
- [ ] Consistent shapes and spacing across the feature
- [ ] Container spacing matches layout spacing
- [ ] Glass variants chosen correctly (`.regular` for controls, `.clear` only over rich backgrounds)
- [ ] No glass applied to content-layer elements (HIG compliance)
- [ ] Performance: limited number of glass effects onscreen
- [ ] No usage of `.glassEffect(.prominent)` -- use `.buttonStyle(.glassProminent)` instead
