# SwiftUI State Management Reference

## Property Wrapper Selection Guide

| Wrapper | Use When | Notes |
|---------|----------|-------|
| `@State` | Internal view state that triggers updates | Must be `private` |
| `@Binding` | Child view needs to modify parent's state | Don't use for read-only |
| `@Bindable` | iOS 17+: View receives `@Observable` object and needs bindings | For injected observables |
| `let` | Read-only value passed from parent | Simplest option |
| `var` | Read-only value that child observes via `.onChange()` | For reactive reads |

**Legacy (Pre-iOS 17):**
| Wrapper | Use When | Notes |
|---------|----------|-------|
| `@StateObject` | View owns an `ObservableObject` instance | Use `@State` with `@Observable` instead |
| `@ObservedObject` | View receives an `ObservableObject` from outside | Never create inline |

---

## @State

Always mark `@State` properties as `private`. Use for internal view state that triggers UI updates.

```swift
// Correct
@State private var isAnimating = false
@State private var selectedTab = 0
```

**Why Private?** Marking state as `private` makes it clear what's created by the view versus what's passed in. It also prevents accidentally passing initial values that will be ignored.

### iOS 17+ with @Observable (Preferred)

**Always prefer `@Observable` over `ObservableObject`.** With iOS 17's `@Observable` macro, use `@State` instead of `@StateObject`:

```swift
@Observable
@MainActor  // Always mark @Observable classes with @MainActor
final class DataModel {
    var name = "Some Name"
    var count = 0
}

struct MyView: View {
    @State private var model = DataModel()  // Use @State, not @StateObject

    var body: some View {
        VStack {
            TextField("Name", text: $model.name)
            Stepper("Count: \(model.count)", value: $model.count)
        }
    }
}
```

**Note**: Mark `@Observable` classes with `@MainActor` to ensure thread safety with SwiftUI, unless your project uses Default Actor Isolation set to `MainActor` — in which case the explicit attribute is redundant.

---

## @Binding

Use only when child view needs to **modify** parent's state. If child only reads the value, use `let` instead.

```swift
// Parent
struct ParentView: View {
    @State private var isSelected = false

    var body: some View {
        ChildView(isSelected: $isSelected)
    }
}

// Child - will modify the value
struct ChildView: View {
    @Binding var isSelected: Bool

    var body: some View {
        Button("Toggle") {
            isSelected.toggle()
        }
    }
}
```

### When NOT to use @Binding

```swift
// Bad - child only displays, doesn't modify
struct DisplayView: View {
    @Binding var title: String  // Unnecessary
    var body: some View { Text(title) }
}

// Good - use let for read-only
struct DisplayView: View {
    let title: String
    var body: some View { Text(title) }
}
```

---

## @StateObject vs @ObservedObject (Legacy - Pre-iOS 17)

**Note**: These are legacy patterns. Always prefer `@Observable` with `@State` for iOS 17+.

The key distinction is **ownership**:
- `@StateObject`: View **creates and owns** the object
- `@ObservedObject`: View **receives** the object from outside

```swift
// Legacy pattern - use @Observable instead
class MyViewModel: ObservableObject {
    @Published var items: [String] = []
}

// View creates it → @StateObject
struct OwnerView: View {
    @StateObject private var viewModel = MyViewModel()

    var body: some View {
        ChildView(viewModel: viewModel)
    }
}

// View receives it → @ObservedObject
struct ChildView: View {
    @ObservedObject var viewModel: MyViewModel

    var body: some View {
        List(viewModel.items, id: \.self) { Text($0) }
    }
}
```

### Common Mistake

Never create an `ObservableObject` inline with `@ObservedObject`:

```swift
// WRONG - creates new instance on every view update
struct BadView: View {
    @ObservedObject var viewModel = MyViewModel()  // BUG!
}

// CORRECT - owned objects use @StateObject
struct GoodView: View {
    @StateObject private var viewModel = MyViewModel()
}
```

### @StateObject instantiation in a View's initializer

This is an anti-pattern in general. Prefer storing the StateObject in the parent view or wherever the model is actually owned, then pass it down. If you must initialize with parameters:

```swift
// WRONG - creates a new ViewModel instance each time the initializer is called
struct MovieDetailsView: View {
    @StateObject private var viewModel: MovieDetailsViewModel

    init(movie: Movie) {
        let viewModel = MovieDetailsViewModel(movie: movie)
        _viewModel = StateObject(wrappedValue: viewModel)  // Called multiple times!
    }
}

// CORRECT - @autoclosure prevents multiple instantiations
struct MovieDetailsView: View {
    @StateObject private var viewModel: MovieDetailsViewModel

    init(movie: Movie) {
        _viewModel = StateObject(
            wrappedValue: MovieDetailsViewModel(movie: movie)
        )
    }
}
```

**Modern Alternative**: Use `@Observable` with `@State` instead.

---

## Don't Pass Values as @State

**Critical**: Never declare passed values as `@State` or `@StateObject`. The value you provide is only an initial value and won't update.

```swift
// Parent
struct ParentView: View {
    @State private var item = Item(name: "Original")

    var body: some View {
        ChildView(item: item)
        Button("Change") {
            item.name = "Updated"  // Child won't see this!
        }
    }
}

// Wrong - child ignores updates from parent
struct ChildView: View {
    @State var item: Item  // Accepts initial value only!

    var body: some View {
        Text(item.name)  // Shows "Original" forever
    }
}

// Correct - child receives updates
struct ChildView: View {
    let item: Item  // Or @Binding if child needs to modify

    var body: some View {
        Text(item.name)  // Updates when parent changes
    }
}
```

**Prevention**: Always mark `@State` and `@StateObject` as `private`. This prevents them from appearing in the generated initializer.

---

## @Bindable (iOS 17+)

Use when receiving an `@Observable` object from outside and needing bindings:

```swift
@Observable
final class UserModel {
    var name = ""
    var email = ""
}

struct ParentView: View {
    @State private var user = UserModel()

    var body: some View {
        EditUserView(user: user)
    }
}

struct EditUserView: View {
    @Bindable var user: UserModel  // Received from parent, needs bindings

    var body: some View {
        Form {
            TextField("Name", text: $user.name)
            TextField("Email", text: $user.email)
        }
    }
}
```

---

## let vs var for Passed Values

### Use `let` for read-only display

```swift
struct ProfileHeader: View {
    let username: String
    let avatarURL: URL

    var body: some View {
        HStack {
            AsyncImage(url: avatarURL)
            Text(username)
        }
    }
}
```

### Use `var` when reacting to changes with `.onChange()`

```swift
struct ReactiveView: View {
    var externalValue: Int  // Watch with .onChange()
    @State private var displayText = ""

    var body: some View {
        Text(displayText)
            .onChange(of: externalValue) { oldValue, newValue in
                displayText = "Changed from \(oldValue) to \(newValue)"
            }
    }
}
```

---

## Environment and Preferences

### @Environment

Access environment values provided by SwiftUI or parent views:

```swift
struct MyView: View {
    @Environment(\.colorScheme) private var colorScheme
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        Button("Done") { dismiss() }
            .foregroundStyle(colorScheme == .dark ? .white : .black)
    }
}
```

### @Environment with @Observable (iOS 17+ — Preferred)

**Always prefer this pattern** for sharing state through the environment:

```swift
@Observable
@MainActor
final class AppState {
    var isLoggedIn = false
}

// Inject
ContentView()
    .environment(AppState())

// Access
struct ChildView: View {
    @Environment(AppState.self) private var appState
}
```

### @EnvironmentObject (Legacy - Pre-iOS 17)

```swift
// Legacy pattern - use @Observable with @Environment instead
class AppState: ObservableObject {
    @Published var isLoggedIn = false
}

ContentView()
    .environmentObject(AppState())

struct ChildView: View {
    @EnvironmentObject var appState: AppState
}
```

---

## When to Use a ViewModel

A "ViewModel" (a separate `@Observable` or `ObservableObject` class) is the right tool when you need:

1. **Testability**: you want to unit-test state transitions, validation, or business logic without spinning up a SwiftUI view
2. **Complex interdependent state**: multiple properties that evolve together (e.g., `isLoading`, `error`, `results` that must stay in sync)
3. **Async coordination**: orchestrating multiple async operations, retry logic, or cancellation
4. **Cross-view reuse**: the same logic powers more than one view

**When you don't need a ViewModel:**
- A view that displays a list from a parent-provided data source — `@State` in the view is fine
- Simple forms where properties map directly to model fields — use `@Bindable` or `@Binding`
- Views with trivial UI state (expanded/collapsed, selected tab) — `@State private` is cleaner

```swift
// Simple view — @State is sufficient
struct SettingsToggleRow: View {
    let title: String
    @Binding var isEnabled: Bool

    var body: some View {
        Toggle(title, isOn: $isEnabled)
    }
}

// Complex view with async, error, and loading — ViewModel is warranted
@Observable
@MainActor
final class SearchViewModel {
    var query = ""
    var results: [SearchResult] = []
    var isLoading = false
    var error: Error?

    private var searchTask: Task<Void, Never>?

    func search() {
        searchTask?.cancel()
        searchTask = Task {
            isLoading = true
            error = nil
            do {
                results = try await SearchService.search(query: query)
            } catch {
                self.error = error
            }
            isLoading = false
        }
    }
}

struct SearchView: View {
    @State private var viewModel = SearchViewModel()

    var body: some View {
        NavigationStack {
            List(viewModel.results) { result in
                SearchResultRow(result: result)
            }
            .searchable(text: $viewModel.query)
            .onChange(of: viewModel.query) { viewModel.search() }
            .overlay {
                if viewModel.isLoading { ProgressView() }
            }
        }
    }
}
```

**Important**: Using a ViewModel is a practical choice for managing complexity — it is not an endorsement of MVVM as an app-wide architecture. For architectural decisions, see `ios-architecture-expert`.

---

## Decision Flowchart

```
Is this value owned by this view?
├─ YES: Is it a simple value type?
│       ├─ YES → @State private var
│       └─ NO (class):
│           ├─ Use @Observable → @State private var (mark class @MainActor)
│           └─ Legacy ObservableObject → @StateObject private var
│
└─ NO (passed from parent):
    ├─ Does child need to MODIFY it?
    │   ├─ YES → @Binding var
    │   └─ NO: Does child need BINDINGS to its properties?
    │       ├─ YES (@Observable) → @Bindable var
    │       └─ NO: Does child react to changes?
    │           ├─ YES → var + .onChange()
    │           └─ NO → let
    │
    └─ Is it a legacy ObservableObject from parent?
        └─ YES → @ObservedObject var (consider migrating to @Observable)
```

---

## State Privacy Rules

**All view-owned state should be `private`:**

```swift
struct MyView: View {
    // Created by view - private
    @State private var isExpanded = false
    @State private var viewModel = ViewModel()
    @AppStorage("theme") private var theme = "light"
    @Environment(\.colorScheme) private var colorScheme

    // Passed from parent - not private
    let title: String
    @Binding var isSelected: Bool
    @Bindable var user: User

    var body: some View { /* ... */ }
}
```

---

## Avoid Nested ObservableObject

**Note**: This limitation only applies to `ObservableObject`. `@Observable` fully supports nested observed objects.

```swift
// Avoid - breaks animations and change tracking
class Parent: ObservableObject {
    @Published var child: Child  // Nested ObservableObject
}

// Workaround - pass child directly to views
struct ParentView: View {
    @StateObject private var parent = Parent()

    var body: some View {
        ChildView(child: parent.child)  // Pass nested object directly
    }
}

struct ChildView: View {
    @ObservedObject var child: Child

    var body: some View {
        Text("\(child.value)")
    }
}
```

---

### Environment Action Pattern

When passing actions through the environment, wrap closures in structs with `callAsFunction()`. Never store raw closures in `EnvironmentValues` — SwiftUI cannot compare closures, causing unnecessary re-renders.

```swift
// GOOD — struct-wrapped action
struct AddToWatchListAction {
    private let action: (Animal.ID) -> Void
    
    init(action: @escaping (Animal.ID) -> Void) {
        self.action = action
    }
    
    func callAsFunction(_ animalID: Animal.ID) {
        action(animalID)
    }
}

extension EnvironmentValues {
    @Entry var addToWatchlist: AddToWatchListAction = .init { _ in }
}

// Usage in view — feels like a function call
struct AnimalDetailView: View {
    @Environment(\.addToWatchlist) private var addToWatchlist
    
    var body: some View {
        Button("Add") { addToWatchlist(animalID) }
    }
}

// BAD — raw closure in environment
extension EnvironmentValues {
    @Entry var addToWatchlist: (Animal.ID) -> Void = { _ in }
    // Closures can't be compared — causes frequent re-evaluations
}
```

### Avoid High-Frequency EnvironmentValues

`EnvironmentValues` is a single large struct. Any update requires SwiftUI to compare its contents for every view reading any part of it. For data that updates at high frequency (scroll offsets, timers, sensor data), use `@Observable` instead.

```swift
// BAD — high-frequency scroll offset in EnvironmentValues
extension EnvironmentValues {
    @Entry var scrollOffset: Double = 0
}

// Even views that don't use scrollOffset pay the comparison cost
// when it updates on every scroll frame

// GOOD — wrap in @Observable for targeted updates
@Observable @MainActor
final class ScrollState {
    var offset: Double = 0
}

// Pass via environment — only views reading .offset get updates
ContentView()
    .environment(scrollState)
```

### @State Initialization with task(id:)

When a view model depends on a value from the parent, avoid initializing it in the view's `init`. `@State` only uses `initialValue` the first time — subsequent parent changes are ignored.

```swift
// BAD — state assignment skipped after first initialization
struct HabitatInfo: View {
    @State private var viewModel: HabitatViewModel
    
    init(habitatId: UUID) {
        _viewModel = State(initialValue: HabitatViewModel(id: habitatId))
        // If parent changes habitatId while this view exists,
        // the new ID is IGNORED — viewModel keeps the old one
    }
}

// GOOD — reactive to parent changes
struct HabitatInfo: View {
    let habitatId: UUID
    @State private var viewModel: HabitatViewModel?
    
    var body: some View {
        content
            .task(id: habitatId) {
                if viewModel?.id != habitatId {
                    viewModel = HabitatViewModel(id: habitatId)
                }
            }
    }
}
```

## Key Principles

1. **Always prefer `@Observable` over `ObservableObject`** for new code
2. **Mark `@Observable` classes with `@MainActor` for thread safety** (unless using default actor isolation)
3. Use `@State` with `@Observable` classes (not `@StateObject`)
4. Use `@Bindable` for injected `@Observable` objects that need bindings
5. **Always mark `@State` and `@StateObject` as `private`**
6. **Never declare passed values as `@State` or `@StateObject`**
7. Use a ViewModel when you need testability, complex state coordination, or async orchestration
8. With `@Observable`, nested objects work fine; with `ObservableObject`, pass nested objects directly
9. Wrap closures in structs with `callAsFunction()` for environment actions — never store raw closures in `EnvironmentValues`
10. Avoid high-frequency data in `EnvironmentValues` — use `@Observable` for scroll offsets, timers, and sensor data
11. Use `task(id:)` for reactive model initialization — never rely on `State(initialValue:)` for parent-dependent data
