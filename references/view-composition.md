# SwiftUI View Composition Reference

View extraction, custom styles, specialized components, layout patterns, and reusability.

## Table of Contents
- [Extract Subviews, Not Computed Properties](#extract-subviews-not-computed-properties)
- [Prefer Modifiers Over Conditional Views](#prefer-modifiers-over-conditional-views)
- [Container View Pattern](#container-view-pattern)
- [Multi-Slot ViewBuilder Containers](#multi-slot-viewbuilder-containers)
- [Custom View Styles](#custom-view-styles)
- [Specialized Built-In Components](#specialized-built-in-components)
- [ZStack vs overlay/background](#zstack-vs-overlaybackground)
- [Reusability: Passing Primitives to Content Views](#reusability-passing-primitives-to-content-views)
- [Layout Principles](#layout-principles)
- [View Logic and Testability](#view-logic-and-testability)
- [Action Handlers](#action-handlers)
- [Anti-Pattern: Conditional Modifier Helpers](#anti-pattern-conditional-modifier-helpers)
- [Anti-Pattern: if/else in View Extensions](#anti-pattern-ifelse-in-view-extensions)

---

## Extract Subviews, Not Computed Properties

### The Problem with @ViewBuilder Functions

When you use `@ViewBuilder` functions or computed properties for complex views, the entire function re-executes on every parent state change:

```swift
// BAD - re-executes complexSection() on every tap
struct ParentView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Button("Tap: \(count)") { count += 1 }
            complexSection()  // Re-executes every tap!
        }
    }

    @ViewBuilder
    func complexSection() -> some View {
        ForEach(0..<100) { i in
            HStack {
                Image(systemName: "star")
                Text("Item \(i)")
                Spacer()
                Text("Detail")
            }
        }
    }
}
```

### The Solution: Separate Structs

Extract to separate `struct` views. SwiftUI can skip their `body` when inputs don't change:

```swift
// GOOD - ComplexSection body SKIPPED when its inputs don't change
struct ParentView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            Button("Tap: \(count)") { count += 1 }
            ComplexSection()  // Body skipped during re-evaluation
        }
    }
}

struct ComplexSection: View {
    var body: some View {
        ForEach(0..<100) { i in
            HStack {
                Image(systemName: "star")
                Text("Item \(i)")
                Spacer()
                Text("Detail")
            }
        }
    }
}
```

**Why**: SwiftUI compares the `ComplexSection` struct. Since nothing changed, it skips calling `body`. With `@ViewBuilder` functions, the code always executes.

### When @ViewBuilder Functions Are Acceptable

Use for small, simple sections that don't affect performance:

```swift
struct SimpleView: View {
    @State private var showDetails = false

    var body: some View {
        VStack {
            headerSection()  // OK - simple, few views
            if showDetails {
                detailsSection()
            }
        }
    }

    @ViewBuilder
    private func headerSection() -> some View {
        HStack {
            Text("Title")
            Spacer()
            Button("Toggle") { showDetails.toggle() }
        }
    }

    @ViewBuilder
    private func detailsSection() -> some View {
        Text("Some details here")
            .font(.caption)
    }
}
```

### When to Extract Subviews

Extract when:
- The view has multiple logical sections or responsibilities
- The view contains reusable components
- The view body becomes difficult to read
- You need to isolate state changes for performance
- The view is becoming large

---

## Prefer Modifiers Over Conditional Views

**Prefer "no-effect" modifiers over conditionally including views.** When you introduce a branch, consider whether you're representing multiple views or two states of the same view.

```swift
// Good - same view, different states
SomeView()
    .opacity(isVisible ? 1 : 0)

// Avoid - creates/destroys view identity
if isVisible {
    SomeView()
}
```

**Why**: Conditional view inclusion can cause loss of state, poor animation performance, and breaks view identity. Using modifiers maintains view identity across state changes.

### When Conditionals Are Appropriate

Use conditionals when you truly have **different views**, not different states:

```swift
// Correct - fundamentally different views
if isLoggedIn {
    DashboardView()
} else {
    LoginView()
}

// Correct - optional content
if let user {
    UserProfileView(user: user)
}
```

---

## Container View Pattern

### Avoid Closure-Based Content

Closures can't be compared, causing unnecessary re-renders:

```swift
// BAD - closure prevents SwiftUI from skipping updates
struct MyContainer<Content: View>: View {
    let content: () -> Content

    var body: some View {
        VStack {
            Text("Header")
            content()  // Always called, can't compare closures
        }
    }
}
```

### Use @ViewBuilder Property Instead

```swift
// GOOD - view can be compared
struct MyContainer<Content: View>: View {
    @ViewBuilder let content: Content

    var body: some View {
        VStack {
            Text("Header")
            content  // SwiftUI can compare and skip if unchanged
        }
    }
}

// Usage - SwiftUI can diff ExpensiveView
MyContainer {
    ExpensiveView()
}
```

---

## Multi-Slot ViewBuilder Containers

Containers accepting multiple `@ViewBuilder` parameters for different content slots:

```swift
struct CardView<Header: View, Content: View, Footer: View>: View {
    @ViewBuilder let header: Header
    @ViewBuilder let content: Content
    @ViewBuilder let footer: Footer

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            header
                .font(.headline)
            content
            Divider()
            footer
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding()
    }
}

// Usage
CardView {
    Text("Wildlife Observation")
} content: {
    Text("Spotted a kiwi near the trail entrance")
} footer: {
    Label("Today", systemImage: "calendar")
}
```

This is the idiomatic SwiftUI pattern for slot-based composition. Each slot is a separate `@ViewBuilder` property, allowing SwiftUI to diff each independently.

---

## Custom View Styles

Build custom styles when the same visual pattern appears across multiple instances. SwiftUI's style protocols (`ButtonStyle`, `LabelStyle`, `ToggleStyle`, etc.) let you define reusable appearance without repeating modifier chains.

```swift
// Prefer - reusable across all action buttons
struct ActionButtonStyle: ButtonStyle {
    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .font(.headline)
            .foregroundStyle(.white)
            .padding(.horizontal, 24)
            .padding(.vertical, 12)
            .background(configuration.isPressed ? Color.blue.opacity(0.8) : .blue)
            .clipShape(.rect(cornerRadius: 12))
            .scaleEffect(configuration.isPressed ? 0.97 : 1)
            .animation(.spring(response: 0.2), value: configuration.isPressed)
    }
}

// Usage
Button("Confirm") { confirm() }
    .buttonStyle(ActionButtonStyle())

Button("Submit") { submit() }
    .buttonStyle(ActionButtonStyle())
```

```swift
// Custom LabelStyle for icon-prominent labels
struct IconFirstLabelStyle: LabelStyle {
    func makeBody(configuration: Configuration) -> some View {
        HStack(spacing: 8) {
            configuration.icon
                .font(.title2)
            configuration.title
                .font(.headline)
        }
    }
}

Label("Settings", systemImage: "gear")
    .labelStyle(IconFirstLabelStyle())
```

### Custom Styles for Structural Changes

Styles can change layout **structure**, not just visual appearance:

```swift
// Custom ToggleStyle that changes structural arrangement
struct VerticalToggleStyle: ToggleStyle {
    func makeBody(configuration: Configuration) -> some View {
        VStack(spacing: 8) {
            configuration.label
                .font(.caption)
            Button {
                configuration.isOn.toggle()
            } label: {
                Image(systemName: configuration.isOn ? "checkmark.circle.fill" : "circle")
                    .font(.title)
                    .foregroundStyle(configuration.isOn ? .green : .secondary)
            }
        }
    }
}

// Standard horizontal toggle
Toggle("Notifications", isOn: $isOn)

// Same toggle, vertical layout via style
Toggle("Notifications", isOn: $isOn)
    .toggleStyle(VerticalToggleStyle())
```

**Key insight**: Custom styles preserve the semantic meaning of the component (it's still a `Toggle` for accessibility) while completely changing the visual structure.

**When to use**: three or more sites with the same visual pattern, or when the modifier chain becomes complex enough to obscure intent.

---

## Specialized Built-In Components

Prefer SwiftUI's specialized built-in views over building equivalents from scratch. They carry semantic meaning, accessibility support, and platform-appropriate behavior.

### Label

`Label` pairs an icon and text with proper semantic grouping:

```swift
// Prefer
Label("Favorites", systemImage: "heart.fill")

// Avoid
HStack {
    Image(systemName: "heart.fill")
    Text("Favorites")
}
```

### GroupBox

`GroupBox` provides a visually grouped container with an optional title:

```swift
GroupBox("Account") {
    VStack(alignment: .leading) {
        LabeledContent("Email", value: user.email)
        LabeledContent("Plan", value: user.plan)
    }
}
```

### Form and Section

Use `Form` with `Section` for settings and data-entry flows:

```swift
Form {
    Section("Profile") {
        TextField("Name", text: $name)
        TextField("Bio", text: $bio, axis: .vertical)
    }

    Section("Notifications") {
        Toggle("Push alerts", isOn: $pushEnabled)
        Toggle("Email digest", isOn: $emailEnabled)
    }
}
```

### LabeledContent

For key-value display in forms:

```swift
LabeledContent("Version", value: appVersion)
LabeledContent("Storage used") {
    Text(storageUsed, format: .byteCount(style: .file))
}
```

### Grid

`Grid` provides tabular layouts with alignment:

```swift
Grid(alignment: .leading) {
    GridRow {
        Text("Species")
            .gridColumnAlignment(.trailing)
        Text("Kiwi")
    }
    GridRow {
        Text("Status")
        Text("Endangered")
            .foregroundStyle(.red)
    }
}
```

### ViewThatFits

`ViewThatFits` enables adaptive content selection:

```swift
ViewThatFits {
    // SwiftUI tries each view in order, picks the first that fits
    Label("Add to Watchlist", systemImage: "binoculars")
    Label("Add to Watchlist", systemImage: "binoculars")
        .labelStyle(.iconOnly)
}
```

### ContentUnavailableView

`ContentUnavailableView` provides standardized empty states:

```swift
ContentUnavailableView(
    "No Observations",
    systemImage: "binoculars",
    description: Text("Start recording wildlife observations to see them here.")
)
```

### ControlGroup

`ControlGroup` groups related controls together:

```swift
ControlGroup {
    Button("Bold", systemImage: "bold") { }
    Button("Italic", systemImage: "italic") { }
    Button("Underline", systemImage: "underline") { }
}
```

---

## ZStack vs overlay/background

Use `ZStack` to **compose multiple peer views** that jointly define layout.

Prefer `overlay` / `background` when you're **decorating a primary view** — they express intent clearly and the primary view remains the layout anchor.

```swift
// GOOD - decoration belongs in overlay
Button("Continue") { }
.overlay(alignment: .trailing) {
    Image(systemName: "lock.fill")
        .padding(.trailing, 8)
}

// BAD - ZStack when overlay is clearer
ZStack(alignment: .trailing) {
    Button("Continue") { }
    Image(systemName: "lock.fill")
        .padding(.trailing, 8)
}
```

```swift
// GOOD - background for visual decoration
HStack {
    Image(systemName: "tray")
    Text("Inbox")
}
.background {
    Capsule().strokeBorder(.blue, lineWidth: 2)
}

// Use ZStack when the decoration must explicitly participate in layout sizing
ZStack {
    Color.blue.opacity(0.1)  // Acts as layout peer
    content
}
```

**Key difference**: In `overlay`/`background`, the child adopts the parent's proposed size. In `ZStack`, each child participates in layout independently.

---

## Reusability: Passing Primitives to Content Views

Prefer passing primitive or value types to presentational (content) views rather than full model objects. This makes views more reusable, easier to preview, and reduces coupling.

```swift
// Preferred for pure display views
struct UserRow: View {
    let name: String
    let handle: String
    let avatarURL: URL?
    let isVerified: Bool

    var body: some View {
        HStack {
            AsyncImage(url: avatarURL) { /* ... */ }
            VStack(alignment: .leading) {
                HStack {
                    Text(name).bold()
                    if isVerified {
                        Image(systemName: "checkmark.seal.fill")
                            .foregroundStyle(.blue)
                    }
                }
                Text(handle).foregroundStyle(.secondary)
            }
        }
    }
}

// Usage is simpler to preview at any state
UserRow(
    name: "Ada Lovelace",
    handle: "@ada",
    avatarURL: nil,
    isVerified: true
)
```

```swift
// When the view edits or coordinates model state, passing the model is fine
struct UserEditor: View {
    @Bindable var user: UserModel  // Needs full model for bindings

    var body: some View {
        Form {
            TextField("Name", text: $user.name)
            TextField("Handle", text: $user.handle)
        }
    }
}
```

**This is a recommendation, not a rule.** Views that coordinate complex state, use bindings, or are tightly scoped to a model may naturally receive the model directly. The goal is reusability and testability, not mechanical compliance.

---

## Layout Principles

### Relative Layout Over Constants

```swift
// Prefer - relative to available space
Text("Full width")
    .frame(maxWidth: .infinity)

Image("hero")
    .resizable()
    .containerRelativeFrame(.horizontal) { length, _ in length * 0.8 }

// Avoid - magic numbers
HeaderView()
    .frame(height: 150)  // Doesn't adapt to different screens
```

### Context-Agnostic Views

Views should work as full screens, modals, sheets, popovers, or embedded content:

```swift
// Good - adapts to given space
struct ProfileCard: View {
    let user: User

    var body: some View {
        VStack {
            AsyncImage(url: user.avatarURL) { /* ... */ }
            Text(user.name)
            Spacer()
        }
        .padding()
    }
}

// Avoid - assumes full screen
struct ProfileCard: View {
    var body: some View {
        Image("avatar")
            .frame(width: UIScreen.main.bounds.width)  // Wrong!
    }
}
```

### Avoid Layout Thrash

```swift
// Bad - deep nesting, excessive layout passes
VStack { HStack { VStack { HStack { VStack { Text("Deep") } } } } }

// Good - flatter hierarchy
VStack {
    Text("Shallow")
    Text("Structure")
}
```

```swift
// Bad - multiple nested GeometryReaders
GeometryReader { outer in
    GeometryReader { inner in /* ... */ }
}

// Good - single reader, or use iOS 17+ alternatives
Text("Content")
    .containerRelativeFrame(.horizontal) { width, _ in width * 0.8 }
```

Gate frequent geometry updates by thresholds:

```swift
// Bad - updates on every pixel change
.onPreferenceChange(SizeKey.self) { currentSize = $0 }

// Good - update only when significant change
.onPreferenceChange(SizeKey.self) { size in
    if abs(size.width - currentSize.width) > 10 {
        currentSize = size
    }
}
```

---

## View Logic and Testability

When logic is complex enough to warrant testing, place it in a separate model:

```swift
// Good - logic in testable model (iOS 17+)
@Observable
@MainActor
final class LoginViewModel {
    var email = ""
    var password = ""

    var isValid: Bool {
        !email.isEmpty && password.count >= 8
    }

    func login() async throws {
        // Business logic here
    }
}

struct LoginView: View {
    @State private var viewModel = LoginViewModel()

    var body: some View {
        Form {
            TextField("Email", text: $viewModel.email)
            SecureField("Password", text: $viewModel.password)
            Button("Login") {
                Task { try? await viewModel.login() }
            }
            .disabled(!viewModel.isValid)
        }
    }
}
```

For simple views, keeping state in `@State` directly is fine and often cleaner. See `state-management.md#when-to-use-a-viewmodel` for guidance.

---

## Action Handlers

**Separate layout from logic.** View body should reference action methods, not contain logic.

```swift
// Good - action references method
struct PublishView: View {
    @State private var viewModel = PublishViewModel()

    var body: some View {
        Button("Publish Project", action: viewModel.handlePublish)
    }
}

// Avoid - logic in closure
struct PublishView: View {
    @State private var isLoading = false

    var body: some View {
        Button("Publish Project") {
            isLoading = true
            // ... business logic directly in view body
        }
    }
}
```

**Why**: Separating logic from layout improves readability and testability. The body should be a structural representation of state, not a place to orchestrate async operations.

---

## Anti-Pattern: Conditional Modifier Helpers

```swift
// DANGEROUS — breaks view identity on every toggle
extension View {
    @ViewBuilder
    func applyIf<V: View>(
        _ condition: Bool,
        transform: (Self) -> V
    ) -> some View {
        if condition {
            transform(self)
        } else {
            self
        }
    }
}

// Usage looks clean but is architecturally dangerous
TabView { /* tabs */ }
    .applyIf(hasTheme) { $0.tint(themeColor) }
    // Toggling hasTheme destroys and recreates the ENTIRE TabView!
```

**Why this is harmful**: The `if/else` block returns different types, guaranteed to reset the view's lifetime whenever the condition toggles. This generic abstraction hides the identity-breaking branching at the call site, making it extremely hard to debug lost state, failed animations, or data reloads.

**Fix**: Use stable modifier chains with ternary operators or modifier parameters:

```swift
// SAFE — single stable view, no identity break
TabView { /* tabs */ }
    .tint(hasTheme ? themeColor : nil)
```

---

## Anti-Pattern: if/else in View Extensions

```swift
// DANGEROUS — resets identity of the view it's applied to
extension View {
    func themed(color: Color?) -> some View {
        if let color {
            self.tint(color)  // Branch A: TintModifier<Self>
        } else {
            self              // Branch B: Self
        }
    }
}

// Applied to a high-level container, this destroys everything inside:
TabView { /* multiple tabs with state */ }
    .themed(color: themeColor)  // Changing themeColor = full destruction

// SAFE — stable chain, no branching
extension View {
    func themed(color: Color?) -> some View {
        self.tint(color)  // nil tint is a no-op
    }
}
```

**Key principle**: keep modifier chains stable. Drive appearance changes with ternary operators or optional-aware modifiers (a `nil`/no-op value) rather than branching the hierarchy, so the view's structural identity — and its state and in-flight animations — persists as the UI adapts to state.

---

## Summary Checklist

- [ ] Complex views extracted to separate `struct` subviews (not `@ViewBuilder` functions)
- [ ] `@ViewBuilder` functions used only for small, simple sections
- [ ] Prefer modifiers over conditional views for state changes
- [ ] Container views use `@ViewBuilder let content: Content`
- [ ] Custom styles used when visual pattern repeats across 3+ sites
- [ ] Specialized built-ins used (Label, GroupBox, Form, LabeledContent)
- [ ] `overlay`/`background` preferred over `ZStack` for decoration
- [ ] Presentational views prefer primitive parameters (reusability recommendation)
- [ ] No hard-coded layout constants; use relative layout
- [ ] Views work in any context (context-agnostic)
- [ ] Business logic separated into testable models (when warranted)
- [ ] Action handlers reference methods, not inline logic
- [ ] Layout thrash avoided (flat hierarchies, gated geometry updates)
- [ ] No `applyIf` or conditional modifier helpers that break view identity
- [ ] View extensions avoid if/else branching (use ternary or optional-aware modifiers)
- [ ] Grid/ViewThatFits/ContentUnavailableView/ControlGroup considered before generic stacks
- [ ] Multi-slot containers use separate `@ViewBuilder` properties per slot
- [ ] Custom styles used for structural changes, not just visual appearance
