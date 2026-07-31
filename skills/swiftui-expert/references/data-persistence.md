# SwiftUI Data Persistence

How SwiftUI views read and write persisted state: child-to-parent preferences, `UserDefaults`-backed storage, per-scene restoration, and SwiftData wiring.

## Table of Contents

- [Preferences (child-to-parent data flow)](#preferences-child-to-parent-data-flow)
- [@AppStorage (UserDefaults-backed values)](#appstorage-userdefaults-backed-values)
- [@SceneStorage (per-scene state restoration)](#scenestorage-per-scene-state-restoration)
- [@AppStorage vs @SceneStorage](#appstorage-vs-scenestorage)
- [SwiftData wiring in SwiftUI](#swiftdata-wiring-in-swiftui)
- [Boundary: wiring vs persistence architecture](#boundary-wiring-vs-persistence-architecture)
- [Summary Checklist](#summary-checklist)

---

## Preferences (child-to-parent data flow)

The environment pushes data DOWN the tree; a `PreferenceKey` carries data the other way — UP from descendants to an ancestor. A child emits a value with `.preference(key:value:)`, and an ancestor collects it with `.onPreferenceChange(_:perform:)`. This is exactly how built-ins like `navigationTitle` and `toolbar` work: the child sets a value the enclosing container reads, which is why a deeply nested view can change the navigation title it never owns.

The key piece is `reduce(value:nextValue:)`. When several siblings emit the same preference, SwiftUI calls `reduce` to merge them into one value. That function is the reconciliation policy — you decide whether multiple contributions sum, keep the last, or take the max.

```swift
// GOOD: child can't know its ancestor's layout, so it reports up via a preference
struct WidthKey: PreferenceKey {
    static let defaultValue: CGFloat = 0
    // merge policy: the widest child wins
    static func reduce(value: inout CGFloat, nextValue: () -> CGFloat) {
        value = max(value, nextValue())
    }
}

struct Row: View {
    var body: some View {
        Label("Item", systemImage: "star")
            .background(GeometryReader { proxy in
                Color.clear.preference(key: WidthKey.self, value: proxy.size.width)
            })
    }
}

struct Container: View {
    @State private var maxWidth: CGFloat = 0   // collected up here
    var body: some View {
        VStack { Row(); Row() }
            .onPreferenceChange(WidthKey.self) { maxWidth = $0 }
    }
}
```

```swift
// BAD: reaching for a preference when a plain binding would do
struct Child: View {
    @State private var text = ""              // parent needs this value...
    var body: some View {
        TextField("Name", text: $text)
            .preference(key: TextKey.self, value: text)  // overkill
    }
}
// A @Binding passed down is the simpler default for known parent/child pairs.
```

**Rule:** Default to bindings for parent/child data flow; reach for a `PreferenceKey` only when a descendant cannot know or name its ancestor (measuring, titles, toolbars).

**Why:** `reduce` runs on every contribution from every matching subview, so high-frequency `.preference` updates (e.g. per scroll frame) are costly. Gate them behind thresholds and prefer bindings when the relationship is direct.

## @AppStorage (UserDefaults-backed values)

`@AppStorage` binds a property to a single `UserDefaults` key. Reading it is like reading `@State`; writing it persists immediately and re-renders every view observing that key. Its projected value `$value` is a `Binding`, so it drops straight into a `Picker`, `Toggle`, or `Stepper`.

```swift
// GOOD: small scalar preference, projected into a control
struct SettingsView: View {
    @AppStorage("preferredTheme") private var theme = Theme.system

    var body: some View {
        Picker("Theme", selection: $theme) {   // $ gives the binding
            ForEach(Theme.allCases, id: \.self) { Text($0.label).tag($0) }
        }
    }
}

// Custom suite for an app group, instead of .standard:
@AppStorage("badgeCount", store: UserDefaults(suiteName: "group.app"))
private var badgeCount = 0
```

```swift
// BAD: secrets in @AppStorage
@AppStorage("authToken") private var authToken = ""   // plaintext, world-readable
```

**Rule:** Use `@AppStorage` for small, non-sensitive scalar preferences (theme, sort order, a feature flag) and use `$value` to drive controls.

**Why:** `UserDefaults` is unencrypted plaintext on disk and trivially extractable from a backup. Never store tokens, passwords, or PII there — that belongs in the Keychain.

## @SceneStorage (per-scene state restoration)

`@SceneStorage` persists lightweight UI state that the system saves and restores automatically as part of state restoration — for example the last-selected tab or detail item, so a relaunched (or backgrounded-then-restored) window comes back where the user left it.

It differs from `@AppStorage` in three ways: it is scoped to a single scene rather than global; the SYSTEM decides when to persist (there is no guarantee and no API to force a write); and the value is discarded when that scene is explicitly closed.

```swift
// GOOD: per-window selection restored automatically, no cross-window bleed
struct LibraryView: View {
    @SceneStorage("selectedBookID") private var selectedBookID: String?

    var body: some View {
        NavigationSplitView {
            BookList(selection: $selectedBookID)
        } detail: {
            BookDetail(id: selectedBookID)
        }
    }
}
```

```swift
// BAD: using @AppStorage for per-window selection in a multi-window app
@AppStorage("selectedBookID") private var selectedBookID: String?
// Every iPad/Mac window shares this key, so opening one book changes them all.
```

**Rule:** Store transient, per-scene UI selection in `@SceneStorage`; store global app-wide preferences in `@AppStorage`.

**Why:** On iPad and Mac an app can have several scenes alive at once. `@AppStorage` is one global value shared by every window, so it bleeds selection across windows; `@SceneStorage` keeps each window's state isolated and lets the system restore it.

## @AppStorage vs @SceneStorage

| Aspect | `@AppStorage` | `@SceneStorage` |
|---|---|---|
| Scope | Global (whole app) | One scene / window |
| Backing | `UserDefaults` | System state-restoration store |
| Durability | Durable until you change it | Best-effort; system decides when to write |
| Lifetime | Survives app relaunch | Discarded when the scene is closed |
| Use for | Settings, preferences, flags | Last-selected tab/detail per window |

## SwiftData wiring in SwiftUI

SwiftData connects to a SwiftUI view tree at four touch points. Inject the container ONCE near the root; read `@Query` and the context in descendants.

1. `@Model` on a class generates the schema (the type conforms to `PersistentModel`).
2. `.modelContainer(for:)` provisions on-disk storage at a view-tree root.
3. `@Query` declaratively fetches model objects and re-runs as the store changes, auto-updating the view.
4. `@Environment(\.modelContext)` hands you the context for `insert` / `delete`.

```swift
// GOOD: container injected once at the root; @Query + context read below it
import SwiftData

@Model
final class Note {
    var title: String
    var createdAt: Date
    init(title: String) {
        self.title = title
        self.createdAt = .now
    }
}

@main
struct NotesApp: App {
    var body: some Scene {
        WindowGroup { NoteList() }
            .modelContainer(for: Note.self)        // provision once, high up
    }
}

struct NoteList: View {
    @Environment(\.modelContext) private var context
    @Query(sort: \Note.createdAt) private var notes: [Note]   // auto-updates

    var body: some View {
        List(notes) { Text($0.title) }
        Button("Add") { context.insert(Note(title: "Untitled")) }
    }
}
```

```swift
// BAD: @Query with no .modelContainer injected above it
struct OrphanList: View {
    @Query private var notes: [Note]   // runtime crash: no container in the tree
    var body: some View { List(notes) { Text($0.title) } }
}
```

**Rule:** Attach `.modelContainer` exactly once at a tree root, then consume `@Query` and `@Environment(\.modelContext)` in descendant views.

**Why:** `@Query` resolves its context from the environment. With no `.modelContainer` above it, there is no context to resolve and the view crashes at runtime — not at compile time, so it slips past the type checker.

## Boundary: wiring vs persistence architecture

This file covers the SwiftUI-side WIRING only — the property wrappers and modifiers that connect persisted state to views. Data-model design, schema migration, and the broader persistence architecture (where the store lives, how layers depend on it, test seams) belong to the `ios-architecture-expert` skill.

## Summary Checklist

- [ ] Prefer bindings for direct parent/child flow; use `PreferenceKey` only when a child can't name its ancestor
- [ ] Treat `reduce(value:nextValue:)` as the merge policy when multiple children contribute
- [ ] Gate high-frequency `.preference` emissions to avoid per-frame reconciliation cost
- [ ] Use `@AppStorage` for small global preferences; drive controls with its `$` binding
- [ ] Never put tokens or PII in `@AppStorage` — it is plaintext `UserDefaults`
- [ ] Use `@SceneStorage` for per-scene UI state so multi-window selection doesn't bleed
- [ ] Remember `@SceneStorage` is best-effort and dies when the scene is closed
- [ ] Inject `.modelContainer` once at a root; read `@Query` / `modelContext` below it
- [ ] Ensure a `.modelContainer` sits above every `@Query` or it crashes at runtime
- [ ] Defer model design, migration, and persistence architecture to `ios-architecture-expert`
